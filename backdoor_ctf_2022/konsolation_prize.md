
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

So for now we haven't found anything useful on this site. After trying some basic paths in the URL one can find this:

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

Nice we got the source code of the server. Taking a look at the code also explains the error. The tried to open file that do not exist and python throwed a exception. But there is one problem:
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

Nothing either




 
