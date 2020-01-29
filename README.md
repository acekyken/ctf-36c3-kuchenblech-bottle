# ctf-36c3-kuchenblech-bottle
Write-Up for the python bottle challenge from the junior ctf of 36c3


<https://github.com/fkt/36c3-junior-ctf-pub/tree/master/chal3>

```python
import string
import urllib2
from bottle import route, run, post, get, request

@get('/')
def index():
with open(__file__) as f:
return '<pre>' + "".join({'<':'&lt;','>':'&gt;'}.get(c,c) for c in f.read()) + '</pre>'


@post('/isup')
@get('/isup')
def isup():
try:
name = request.params.get('name', None)
is_remote = request.environ.get('REMOTE_ADDR') != '127.0.0.1'
is_valid = all(x in name for x in ['http', 'kuchenblech'])
if is_remote and not is_valid:
raise Exception
result = urllib2.urlopen(name).read()
return result
except:
return 'Error'

run(host='0.0.0.0', server="paste", port=8080, reloader=False)
```
Let: 102.102.102.102 = challenge ip

I have to be honest, I don't quite remember the order of our thought process. I do remember that it took us a very long time to figure out. We did learn a lot though about python, http requests and their attribute. I will try to account for all of our ideas, as well as what we tried to find the solution, as well as the solution we eventually found. This is not a straight-forward guide on how to get the flag of this challenge, but an insight into how a process of finding a flag can look like.

If we visit the given url/ip, we get the above source code. Through looking at it, we conclude that this must be python code. We see the import from bottle import .. and we can google this and find out that bottle is a web-framework.

First of all, we see that [/isup(](file:///isup()) can be reached via a post and a get request. If this site is visited, then the parameters of the request are checked for an argument name. This is written into the variable name. In the next line the ip address of the request origin gets checked. If it comes from 127.0.0.1, so localhost, this is true. otherwise, is_remote is false.

There is another boolean variable. is_valid becomes true if 'http' and 'kuchenblech' are in name. 

If is_remote is true and is_valid is false, an exception is raised. Otherwise, with urllib the url is opened and the result written to result.

Obviuously, we don't want an exception to occur. We know from the hint that there is a file at \ called flag. Our first idea what that this is also a path in the webserver. We think that we have to give the correct parameter and maybe then we can get a link our something. 

So firstly, in our browser, we open the developer tools (comes with firefox and chromium, not sure about other browsers). Here, we can switch to the network-tab and then reload the page: we see a get-request for [/isuop.](file:///isuop.) We take this and modify it with the developer tools so that it is a POST request with the parameter 'name' containing 'http' and also 'kuchenblech'. No luck. 
In the following hours, we try to find out how objects are transmitted to bottle and try to give a (json-)represention of a dictionary holding http and kuchenblech to the variable name. This, of course, did not result in anything. We look at bottle and the params.get() function: 


<https://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.params>

<https://bottlepy.org/docs/dev/api.html#data-structures>

get cleary says that it gets the most recent value of a key. So all of our tries of writing http and kuchenblech successively into name are cleary futile. These tries were based on the guess that all() operates on dicts. We now thought that bottle might have some serialization form of python objects, and this is what we had to use. 

Let's look at that all-statement again: is_valid = all(x in name for x in ['http', 'kuchenblech'])

<https://docs.python.org/3/library/functions.html?highlight=all#all>

x in name?

<https://docs.python.org/3/reference/expressions.html#in>

This now means that we were wrong and all() also works on strings. duh. 

Okay, so this means that handing a parameter like name=httpkuchenblech should also be "valid". Maybe an exception is thrown at a different point? Hmmmm..
Well, there is this 
result = urllib2.urlopen(name).read()
line and it takes name and wants to url-open it. Hm, maybe it has to be a url?
We try "<http://102.102.102.102/flag?kuchenblech>"
because this has to be a valid url, right? Nope, error. 
We can maybe try things like google.de/?kuchenblech? 
ha, one of those worked!! yesss. Okay, now we need to construct a url that will display the file. 
At this point I think we had the epiphany that flag might be a file on the local file system and that we could provide a url that opens a local file.
If you ever stored an html file on your local machine and opened it in a browser, you know that the url bar will hold something like file:/* on linux. URL is short for unified resource location, and there are different URI schemes arround. On the web, we often use the http protocol to access files on servers. On our local machine, we can access fles via the file URI scheme https://tools.ietf.org/html/rfc8089
The second thing that comes to mind is that and exception is raised only if is_remote is true AND not is_valid is true. This means that if is_remote is false, so the request came from localhost, the exception will not be raised, even if is_valid is false. 
Those 2 last ideas lead to the correct input:
http://102.102.102.102/isup?name=http%3A%2F%2F127.0.0.1%3A8080%2Fisup%3Fkuchenblech%26name%3Dfile%3A%2F%2F%2Fflag
