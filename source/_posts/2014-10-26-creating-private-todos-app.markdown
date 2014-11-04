---
layout:     post
title:      "Creating private todos app"
date:       2014-10-26 11:59:59
author:     "Alexey Galkin"
categories: [node, derby]
---

## Abstract

In this post I will show how to setup derby development environment, create fresh derby application with aid of the yeoman template, add some logic and security to it.

User of the application will be able to:

- Register
- Login and logout
- View his personal list of todos
- Add new todo to the list
- Remove todo from the list
- Mark todo as completed
- Filter todos by completion status

<!-- more -->

## Prerequisites

First of all you have to make sure that your development environment meets minimum requirements for derby development. These are:

- **nodejs** - most convenient way to install it is to use [node version manager](https://github.com/creationix/nvm)
- **mongodb** - please refer to one of guides from the [official site](http://docs.mongodb.org/manual/installation)

## Starting off

### Preparation

In order to start scaffolding our application we need to install two additional modules - yeoman and generator-derby:
{% highlight bash %}
➜  npm install yo -g
➜  npm install generator-derby -g
{% endhighlight %}

Now let's create new directory for our application and cd into it:
{% highlight bash %}
➜  mkdir private-todos
➜  cd private-todos
{% endhighlight %}

### Scaffolding

We are ready to use generator-derby in order to create some strong ground for our application. Start it with ```yo derby``` and then pick the following options:

- Select project level features - derby-login
- Select derby-login packages - none
- Input Derby-app name - private-todos
- Select "private-todos"-application features - Bootstrap 3

After that yeoman will generate our application and install all required packages. We are ready to go!

Lets see what we got:

<pre>
.
|-- README.md
|-- apps
|   |-- error
|   |   |-- index.js
|   |   |-- styles
|   |   |   |-- index.css
|   |   |   `-- reset.css
|   |   `-- views
|   |       |-- 403.html
|   |       |-- 404.html
|   |       |-- 500.html
|   |       `-- index.html
|   `-- private-todos
|       |-- index.js
|       |-- styles
|       |   `-- index.css
|       `-- views
|           |-- home.html
|           `-- index.html
|-- components
|-- config
|   |-- defaults.json
|   `-- login.js
|-- package.json
|-- public
|   |-- derby
|   `-- img
|       `-- favicon.ico
|-- server
|   |-- config.js
|   |-- error.js
|   |-- express.js
|   |-- routes.js
|   `-- store.js
|-- server.js
`-- test
</pre>

- **app** will contain derby-apps which are essentially building block of every derby application
- **components** is for placing custom components
- **config** contains configuration options for our application, default settings and settings of derby-login module
- **public** is for placing our public assets that have to be served to the clients
- **server** contains some code that will bootstrap our application on startup
- *server.js* is the entry point to our application, we will see into that shorly
- **test** is for placing tests

Okay, freshly scaffolded application is supposed to be fully operational. So lets start it and check what it actually does.
 
> Before starting please make sure that you have mongod started on the default port. Which is 27017. If that is not your case by some reason, please setup MONGODB_URL environment variable with correct mongodb connection string.

To start out application we have to run command `node server`:

<pre>
➜  node server
Master pid  5489
Bundle created: private-todos
5490 listening. Go to: http://localhost:3000/
</pre>

Nice, it worked. Lets open this url in a browser.
{% img screenShot /images/posts/001/001-first-run.png %}

It looks like something gone wrong. Somehow we are not on the home page and we can't return there. We always get redirected at login page and that leads to 404-error. Why is that?

## Authentication

### What happened?

Actually everything worked as expected. Derby-login module does huge amount of job when plugged in as express middleware. One of it's default settings is to secure whole application effectively denying access to every page except of login, logout and few other pages. That is why we don't see our freshly created homepage. But we hadn't created login and logout logic yet. That is why we are getting 404-error.

> That might not be obvious from the start but derby-login module has enormous amount of settings. Most of them are set to their defaults when using it from yeoman generator. If you want to check whole range of possibilities that derby-login offers then the best place to start would be [derby-login-example application](https://github.com/derbyparty/derby-login-example).

### Implementing login and logout

We could create several handlers in our main app (which is private-todos). But for the sake of separation of concerns we will split our application into two apps:

- **private-todos** - this app will handle all todo-specific logic
- **login** - this app will handle login and registration logic

> We could create something to handle logout logic as well, but derby-login already does that for us.

To do that we have to create one more app inside of **apps** directory:
{% highlight bash %}
➜  mkdir apps/login
➜  mkdir apps/login/views
➜  mkdir apps/login/styles
{% endhighlight %}

Then lets create empty index.css file in **apps/login/styles** directory.
{% highlight bash %}
➜  touch apps/login/styles/index.css
{% endhighlight %}

Now we have to create few more files to implement login logic:

*apps/login/index.js*:
{% highlight javascript %}
var derby = require('derby');

var app = module.exports = derby.createApp('login', __filename);

if (!derby.util.isProduction) global.app = app;

app.use(require('d-bootstrap'));
app.use(require('derby-login/components/notAuth'));

app.loadViews(__dirname + '/views');
app.loadStyles(__dirname + '/styles');

app.get('/login', function(page, model){
    page.render('login');
});
{% endhighlight %}

*apps/login/views/index.html*:
{% highlight html %}{% raw %}
<import: src='./login'>
<Title:>
  {{_page.title}}
  <Body:>
    <view name="{{$render.ns}}"/>
{% endraw %}{% endhighlight %}

*apps/login/views/login.html*:
{% highlight html %}
<index:>
    <div class="container">
        <div class="row">
            <h1>Welcome to private todos app </h1>
        </div>
        <div class="row">
            <div class="col-sm-4 col-sm-offset-1">
                <view name="auth:login" />
            </div>
            <div class="col-sm-4 col-sm-offset-1">
                <view name="auth:register" />
            </div>
        </div>
    </div>
{% endhighlight %}

We are almost done. One more thing we have to do is to setup derby-login to omit email verification process. That is turned on by default and in real life your should not turn it off. But today for the sake of simplicity we will do it.

Change the following lines in *config/login.js*:
{% highlight javascript %}
module.exports = {
  // … code omited
    user: {
        id: true,
        email: true
    }
};
{% endhighlight %}

To:
{% highlight javascript %}
module.exports = {
  // … code omited
    user: {
        id: true,
        email: true
    },
    confirmRegistration: false
};
{% endhighlight %}

Last thing we have to do is to plug our login app into our derby application. That is done through *server.js*, we have to change it:
{% highlight javascript %}
// … code omited
var apps = [
    require('./apps/private-todos')
    // <end of app list> - don't remove this comment
];
// … code omited
{% endhighlight %}

To:
{% highlight javascript %}
// … code omited
var apps = [
    require('./apps/private-todos'),
    require('./apps/login')
    // <end of app list> - don't remove this comment
];
// … code omited
{% endhighlight %}

Now we can try to start our application again:

<pre>
➜  node server
Master pid  6465
Bundle created: private-todos
Bundle created: login
6466 listening. Go to: http://localhost:3000/
</pre>

And the result is:
{% img screenShot /images/posts/001/002-login-works.png %}

Now we can register. After that we will be redirected to the homepage of our application:
{% img screenShot /images/posts/001/003-after-registration.png %}

We hadn't implemented logout link yet but derby-login already has us covered - lets navigate to */auth/logout*:
{% img screenShot /images/posts/001/004-logout-works.png %}

Now it is time to implement todos logic.

## Implementing private todos

### Adding todos logic

We are going to add some code that will allow users to manage their todo list. We will take that code from awesome sample app [derby-example-todo](https://github.com/zag2art/derby-example-todo). We need to copy following files:.

- *derby-example-todo/app/index.js* to *apps/private-todos/index.js*
- *derby-example-todo/app/css/index.css* to *apps/private-todos/styles/index.css*
- *derby-example-todo/app/views/index.html* to *apps/private-todos/views/index.html*

Since generator-derby creates slightly different directory structure we need to change *apps/private-todos/index.js* from:
{% highlight javascript %}
var derby = require('derby');
var app = module.exports = derby.createApp('todos', __filename);

global.app = app;

app.loadViews (__dirname+'/views');
app.loadStyles(__dirname+'/css');
// … code omited
{% endhighlight %}
To:
{% highlight javascript %}
var derby = require('derby');
var app = module.exports = derby.createApp('private-todos', __filename);

global.app = app;

app.loadViews (__dirname+'/views');
app.loadStyles(__dirname+'/styles');
// … code omited
{% endhighlight %}

Also lets remove *apps/private-todos/views/home.html* since we don't really need that anymore - all our view logic is contained in *apps/private-todos/views/index.html*.

Now it is time to run our application and login. We can see empty todo list:
{% img screenShot /images/posts/001/005-empty-todos.png %}

We can add several items and check that it actually live updates:
{% img screenShot /images/posts/001/006-todos-added.png %}

> Take note that our todos list does not use bootstrap styles. That is due to the fact that derby creates different "bundle" for every app registered. In our case "login" bundle has bootstrap styles embedded while our "private-todos" bundle uses its own styles. That gives us really nice separation of code.

But if we create another user and login then we will see the same todo list. Obviously there is very little private here. We need to fix that.

### Adding some privacy

First of all lets add some way to distinguish users so we can test our application easily. Lets add current user's email in the top right corner along with logout link. 

First we need to add the following styles at the end of *apps/private-todos/styles/index.css*:
{% highlight css %}
#userInfo {
    position: fixed;
    top:10px;
    right:20px;
}
{% endhighlight %}

Now we have to change *apps/private-todos/views/index.html*:
{% highlight html %}
<Body:>
  <section id="todoapp">
    <view name="header"/>
    <view name="main"/>
    <view name="footer"/>
  </section>
<!-- … code omited -->
{% endhighlight %}

To:
{% highlight html %}
<Body:>
  <section id="todoapp">
    <view name="userInfo"/>
    <view name="header"/>
    <view name="main"/>
    <view name="footer"/>
  </section>

<userInfo:>
{% raw %} <div id="userInfo">{{_session.user.email}} (<a href="/auth/logout">logout</a>)</div> {% endraw %}
<!-- … code omited -->
{% endhighlight %}

Finally we have to make sure that ```_session.user.email``` is accessible from the view. So we have to change *apps/private-todos/index.js*:
{% highlight javascript %}
// … code omited
app.get('/',          getPage('all'));
app.get('/active',    getPage('active'));
app.get('/completed', getPage('completed'));
// … code omited
{% endhighlight %}

To:
{% highlight javascript %}
// … code omited
app.get('*', function(page, model, params, next) {
    if (model.get('_session.loggedIn')) {
        var userId = model.get('_session.userId');
        var user = model.at('users.' + userId);
        model.subscribe(user, function() {
            model.ref('_session.user', user);
            next();
        });
     } else {
        next();
     }
});

app.get('/', getPage('all'));
app.get('/active', getPage('active'));
app.get('/completed', getPage('completed'));
// … code omited
{% endhighlight %}

Lets check what we have now:
{% img screenShot /images/posts/001/007-userinfo-added.png %}

Okay, it looks like we have our application personalized. But obviously todos list is now shared between users. You can easily check that if you login with two different users. That is not what we want and that has to be changed.

### Adding some privacy

Actually it is very easy to add some privacy to our todos - we have to add ownerId field upon creation of each todo and for each user we have to load only his(her) todos. Lets implement that.

We need to change **getPage** method in *apps/private-todos/index.js*:
{% highlight javascript %}
function getPage(filter) {
    return function(page, model) {
        model.subscribe('todos', function() {
            model.filter('todos', filter).ref('_page.todos');
            model.start('_page.counters', 'todos', 'counters');
            page.render();
        });
    }
}
{% endhighlight %}

To:
{% highlight javascript %}
function getPage(filter) {
    return function(page, model) {
        var todos = model.query('todos', {
            ownerId: model.get('_session.userId')
        });
        model.subscribe(todos, function() {
            model.filter('todos', filter).ref('_page.todos');
            model.start('_page.counters', 'todos', 'counters');
            page.render();
        });
    };
}
{% endhighlight %}

And **addTodo** method in *apps/private-todos/index.js*:
{% highlight javascript %}
app.proto.addTodo = function(newTodo) {
    if (!newTodo) return;
        this.model.add('todos', {
            text: newTodo,
            completed: false
        });
    this.model.set('_page.newTodo', '');
};
{% endhighlight %}

To:
{% highlight javascript %}
app.proto.addTodo = function(newTodo) {
    if (!newTodo) return;
    this.model.add('todos', {
        text: newTodo,
        completed: false,
        ownerId: this.model.get('_session.userId')
    });
    this.model.set('_page.newTodo', '');
};
{% endhighlight %}

Okay, lets check what we have now. Lets login as one of the users and add some todos:
{% img screenShot /images/posts/001/008-aa-todos.png %}

Now lets login as another user and see that he has empty todo list:
{% img screenShot /images/posts/001/009-zz-todos.png %}

It looks like everything works now. But actually while our application looks like it secures todos of one user from another, it has some not very pleasant backdoor. Lets open console in the browser and type few commands:
{% highlight javascript %}
var auths = app.model.query('auths',{})
app.model.fetch(auths)
auths.get()
{% endhighlight %}

As the result we see that our server happily returns contents of ```auths``` collection which has all of our users with password hashes etc:
{% img screenShot /images/posts/001/010-security-breach.png %}

Things are really bad since using the same trick we could query all todos and even add todos to list of any user.

### Securing the application

Obviously we need to setup our server to allow users only to view, modify and delete only their own todos and nothing more. For that purpose we need to install [share-access](https://github.com/dmapper/share-access) module:
{% highlight bash %}
npm install share-access -S
{% endhighlight %}

Now we have to configure share-access for our project. First of all lets create file with share-access configuration - *server/shareAccess.js*:
{% highlight javascript %}
var shareAccess = require('share-access');

module.exports = shareAccess;

function isOwner(docId, doc, session) {
    return doc.ownerId === session.userId;
}

shareAccess.allowRead('todos', isOwner);
shareAccess.allowDelete('todos', isOwner);
shareAccess.allowCreate('todos', isOwner);

shareAccess.allowUpdate('todos', function(docId, oldDoc, newDoc, path, session){
    return isOwner(docId, oldDoc, session);
});
{% endhighlight %}

Configuration is very straightforward - basically on every request to our collection we are checking if user is the owner of the todo. If not - we are denying access. All collections are secured by default. If we would want to add some permissive rules - we would have to do it for every other collection in our application.

Now we need to plug our configuration into our server. That is done through *server.js*:
{% highlight javascript %}
var async = require('async');
var derby = require('derby');

var http  = require('http');
var chalk = require('chalk');

var publicDir = process.cwd() + '/public';

derby.run(function(){
// … code omited 
{% endhighlight %}

To:
{% highlight javascript %}
var async = require('async');
var derby = require('derby');

var http  = require('http');
var chalk = require('chalk');

var publicDir = process.cwd() + '/public';
var shareAccess = require('./server/shareAccess');

derby.use(shareAccess);

derby.run(function(){
// … code omited 
{% endhighlight %}

Now if we try to run our application again and to do our little trick with querying 'auths' collection, we will get 403-error:
{% img screenShot /images/posts/001/011-security-works.png %}

# Wrapping up

Today we created new derby application from the scratch, added some logic to it and secured it from malicious access. Finalized source code for this application can be found [here](https://github.com/glkn-blog/001-private-todos)

# Credits

- [Decision mapper tutorials](http://decisionmapper.com/tutorials)
- [derby-login module](https://github.com/derbyparty/derby-login)
- [derby-login-example application](https://github.com/derbyparty/derby-login-example)
- [derby-example-todo](https://github.com/zag2art/derby-example-todo)
