# How to add a custom Milter to Zimbra.

This article describes how you can create an extension to Postfix. By implementing a Milter (an email extension protocol) you can create an extension that receives events whenever an email is sent or received. Your custom Milter extension can have event handlers for SMTP events (CONNECT, DISCONNECT), SMTP commands (HELO, MAIL FROM, etc.) as well as mail content (headers and body). 

Milters allow you to add or replace (custom) email headers, filter out specific content for implementing privacy policies, automatically add BCC recipients for archiving, add disclaimers etc. All this can be done conditionally for specific senders and/or recipients. You can also use a Milter to enhance the functionality of Zimbra Distribution lists.

Milters are also used in other Zimbra components such as DKIM and SpamAssasin. This is not a problem as you can have multiple Milters on Zimbra Postfix.

## Python 3 Milter demo

While there are Java, C and NodeJS implementations of Milters, the one used in this article is based on Python 3. The advantage of using Python in this scenario is that it avoids the need to compile (C/Java), which makes it easier to debug. In addition you can use LDAP and MariaDB in Python if you need to.

Milters can be installed on your Zimbra server or on a dedicated server. Installation steps for Ubuntu 20.04: 

     apt install python3-milter supervisor
     mkdir /etc/milter

This demo script checks if a user from our Zimbra server `example.com` sends an email to `specialcompany.com`, the Milter script will replace the `From` email header with that of the legal department, it will also add the legal department as a BCC recipient. This happens without user interaction. And since the Milter is fully server side, it will always be triggered. It does not matter if the user uses webmail, a mobile device or a desktop client.

Add the demo Milter script by using `nano` to `/etc/milter/custom-milter.py`. Read the in-code comments to understand how it works:
     
```python
#!/usr/bin/python3
## To roll your own milter, create a class that extends Milter.  
#  This is a useless example to show basic features of Milter. 
#  See the pymilter project at https://pymilter.org based 
#  on Sendmail's milter API 
#  This code is open-source on the same terms as Python.

## Milter calls methods of your class at milter events.
## Return REJECT,TEMPFAIL,ACCEPT to short circuit processing for a message.
## You can also add/del recipients, replacebody, add/del headers, etc.

from __future__ import print_function
import Milter
try:
  from StringIO import StringIO as BytesIO
except:
  from io import BytesIO
import time
import email
from email import message_from_binary_file
from email import policy
import mimetypes
import os
import sys
from socket import AF_INET, AF_INET6
from Milter.utils import parse_addr
if True:
  # for logging process - usually not needed
  from multiprocessing import Process as Thread, Queue
else:
  from threading import Thread
  from Queue import Queue

logq = None

class myMilter(Milter.Base):

  def __init__(self):  # A new instance with each new connection.
    self.id = Milter.uniqueID()  # Integer incremented with each call.
    self.mustRewriteFrom = 'false'
    self.comingFromMe = 'false'
    self.fromHeader = '';

  # each connection runs in its own thread and has its own myMilter
  # instance.  Python code must be thread safe.  This is trivial if only stuff
  # in myMilter instances is referenced.
  @Milter.noreply
  def connect(self, IPname, family, hostaddr):
    # (self, 'ip068.subnet71.example.com', AF_INET, ('215.183.71.68', 4720) )
    # (self, 'ip6.mxout.example.com', AF_INET6,
    #	('3ffe:80e8:d8::1', 4720, 1, 0) )
    self.IP = hostaddr[0]
    self.port = hostaddr[1]
    if family == AF_INET6:
      self.flow = hostaddr[2]
      self.scope = hostaddr[3]
    else:
      self.flow = None
      self.scope = None
    self.IPname = IPname  # Name from a reverse IP lookup
    self.H = None
    self.fp = None
    self.receiver = self.getsymval('j')
    self.log("connect from %s at %s" % (IPname, hostaddr) )
    
    return Milter.CONTINUE


  ##  def hello(self,hostname):
  def hello(self, heloname):
    # (self, 'mailout17.dallas.texas.example.com')
    self.H = heloname
    self.log("HELO %s" % heloname)
    #if heloname.find('.') < 0:	# illegal helo name
    #  # NOTE: example only - too many real braindead clients to reject on this
    #  self.setreply('550','5.7.1','Sheesh people!  Use a proper helo name!')
    #  return Milter.REJECT
    
    return Milter.CONTINUE

  ##  def envfrom(self,f,*str):
  def envfrom(self, mailfrom, *str):
    self.F = mailfrom
    self.R = []  # list of recipients
    self.fromparms = Milter.dictfromlist(str)	# ESMTP parms
    self.user = self.getsymval('{auth_authen}')	# authenticated user
    self.log("mail from:", mailfrom, *str)
    # NOTE: self.fp is only an *internal* copy of message data.  You
    # must use addheader, chgheader, replacebody to change the message
    # on the MTA.
    self.fp = BytesIO()
    self.canon_from = '@'.join(parse_addr(mailfrom))
    self.fp.write(b'From %s %s\n' % (self.canon_from.encode(),
        time.ctime().encode()))
    return Milter.CONTINUE


  ##  def envrcpt(self, to, *str):
  @Milter.noreply
  def envrcpt(self, to, *str):
    if 'specialcompany.com' in to:
       self.mustRewriteFrom = 'true'
    rcptinfo = to,Milter.dictfromlist(str)
    self.R.append(rcptinfo)
    self.log("mail envrcpt:", self.R)
    
    return Milter.CONTINUE


  @Milter.noreply
  def header(self, name, hval):
    self.fp.write(b'%s: %s\n' % (name.encode(),hval.encode()))	# add header to buffer
    if name == 'From':
       self.fromHeader = '%s' % hval
    #self.log("header: ", name.encode(), hval.encode())	# log header
    return Milter.CONTINUE

  @Milter.noreply
  def eoh(self):
    self.fp.write(b'\n')				# terminate headers
    return Milter.CONTINUE

  @Milter.noreply
  def body(self, chunk):
    self.fp.write(chunk)
    return Milter.CONTINUE

  def eom(self):
    self.fp.seek(0)
    #msg holds the entire message
    msg = email.message_from_binary_file(self.fp, policy=policy.default)
    self.log("msg:", msg);

    #example on how to iterate through attachments
    for attachment in msg.iter_attachments():
      #attachment holds the attachment object so that it can be used with a new MIMEMultipart() message
      self.log("Attachment filename is %s" % (attachment.get_filename(),))
      self.log("Attachment content/type is %s" % (attachment.get_content_type(),))
      data = attachment.get_content()
      self.log("Attachment content is %s" % (data,))

    # many milter functions can only be called from eom()    
    self.log("eom reached", self.fromHeader)
    if 'example.com' in self.fromHeader:
       self.comingFromMe = 'true'

    if 'true' in self.comingFromMe:
       if 'true' in self.mustRewriteFrom:
          self.log("rewriting from")
          self.chgheader('From',1,'Legal Dept. <legal@example.com>')
          # example of adding a Bcc:
          self.addrcpt('<%s>' % 'legal@example.com')
    return Milter.ACCEPT

  def close(self):
    # always called, even when abort is called.  Clean up
    # any external resources here.
    return Milter.CONTINUE

  def abort(self):
    # client disconnected prematurely
    return Milter.CONTINUE

  ## === Support Functions ===

  def log(self,*msg):
    t = (msg,self.id,time.time())
    if logq:
      logq.put(t)
    else:
      # logmsg(*t)
      pass

def logmsg(msg,id,ts):
    print("%s [%d]" % (time.strftime('%Y%b%d %H:%M:%S',time.localtime(ts)),id),
        end=None)
    # 2005Oct13 02:34:11 [1] msg1 msg2 msg3 ...
    for i in msg: print(i,end=None)
    print()
    sys.stdout.flush()

def background():
  while True:
    t = logq.get()
    if not t: break
    logmsg(*t)

## ===
    
def main():
  bt = Thread(target=background)
  bt.start()
  #socketname = os.getenv("HOME") + "/pythonsock"
  socketname = "inet:8800"
  timeout = 600
  # Register to have the Milter factory create instances of your class:
  Milter.factory = myMilter
  flags = Milter.CHGBODY + Milter.CHGHDRS + Milter.ADDHDRS
  flags += Milter.ADDRCPT
  flags += Milter.DELRCPT
  Milter.set_flags(flags)       # tell Sendmail which features we use
  print("%s milter startup" % time.strftime('%Y%b%d %H:%M:%S'))
  sys.stdout.flush()
  Milter.runmilter("pythonfilter",socketname,timeout)
  logq.put(None)
  bt.join()
  print("%s milter shutdown" % time.strftime('%Y%b%d %H:%M:%S'))

if __name__ == "__main__":
  # You probably do not need a logging process, but if you do, this
  # is one way to do it.
  logq = Queue(maxsize=4)
  main()
```

A new instance of our Python Milter is created every time an email is passed to Postfix. So you can use class variables to keep track of things while events are triggered. The events this Python script uses are `header, envrcpt, eom`, the others are shown for reference and do some logging. Some events such as `eom` end-of-message are triggered once. Others such as `header, envrcpt` are triggered for each one of the mail headers/recipients. The order in which the events trigger are similar to how an email is sent over SMTP. So CONNECT... RCPT TO... headers, body etc.

Please note that the email From header can only be obtained via the `header` event. The `envfrom` event can be used to get the From that is used in the SMTP session. These may or may not be the same so pay attention to it.

In most cases you will have to gather all the variables you need and set them on the class instance as the events fire. Then implement your custom functionality in the `eom` (end of message) event.

Example, the `envrcpt` event is called for all the recipients of the email, when we see a `specialcompany.com` recipient, we set `self.mustRewriteFrom = 'true'` so we know there was a `specialcompany.com` recipient when we receive the `eom` event. `eom` is one of the last events to be triggered.

```python
  def envrcpt(self, to, *str):
    if 'specialcompany.com' in to:
       self.mustRewriteFrom = 'true'
```

All event handlers need to return one of `Milter.CONTINUE, Milter.ACCEPT or Milter.REJECT`. Continue means continue processing this email in the next event. Accept means our Milter is all done, and Postfix or a next Milter can start processing it. Reject tells Postfix to reject this message, the user will see a prompt or get a message saying sending failed.

Set up supervisord to load our Python script:

nano `/etc/supervisor/conf.d/milter.conf`

      [program:milter-custom]
      command=/etc/milter/custom-milter.py
      process_name=milter-custom
      priority=1
      redirect_stderr=true
      stdout_logfile=/var/log/milter-custom.log
   
Enable and start the service.

     chmod +rx /etc/milter/custom-milter.py
     systemctl restart supervisor
     systemctl enable supervisor

Then check if the milter service is running:

     tail -f /var/log/milter-custom.log
     netstat -tulpn | grep 8800 #should show the service

If you made any typos, you will see them in the log, and there will be nothing listening on port 8800. Try again and issue `systemctl stop supervisord && systemctl start supervisord` or `systemctl stop supervisor && systemctl start supervisor`. 

**It is advised to install a host firewall, so you can reject incoming connections to Milters that are not coming from your Zimbra cluster. On a single server you can use `socketname = "inet:127.0.0.1:8800"` in the Python script.**

If it works, enable it for Zimbra:

     su - zimbra
     zmprov ms `zmhostname` zimbraMtaSmtpdMilters inet:127.0.0.1:8800
     zmprov ms `zmhostname` zimbraMtaNonSmtpdMilters inet:127.0.0.1:8800
     zmprov ms `zmhostname` zimbraMilterServerEnabled TRUE
     zmmtactl restart

     postconf smtpd_milters
     smtpd_milters = inet:127.0.0.1:8800, inet:127.0.0.1:7026

     #if you have no milter running at 7026, you can:
     postconf -e 'smtpd_milters = inet:127.0.0.1:8800'

Try sending some emails and:

     tail -f /var/log/milter-custom.log
     tail -f /var/log/zimbra.log

You can also run the milter without supervisord, stop supervisord and just run it like `python3 /etc/milter/custom-milter.py`.


## See also: 

- http://www.postfix.org/MILTER_README.html
- https://github.com/sdgathman/pymilter
- https://iomarmochtar.wordpress.com/2017/09/13/zimbra-prevent-user-customizing-from-header/ (Python 2 is out of support, for reference)
- https://github.com/Zimbra-Community/mailing-lists/tree/master/milter (Python 2 is out of support, for reference)
