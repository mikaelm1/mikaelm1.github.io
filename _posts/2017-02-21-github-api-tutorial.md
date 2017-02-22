---
layout: post
title: GitHub API Tutorial
subtitle: How to use GitHub's API
---

## Description

This guide will demonstrate some of the features of GitHub's API. We will make a very simple app with Node.js that will allow us to authenticate with GitHub and view, create, and delete repositories. The source code for the sample app can found [here](https://github.com/mikaelm1/github-api-tutorial). This tutorial is written as a project for my CS290 class at Oregon State University. 

- [Getting Started](#getting-started)
- [Base Application Setup](#base-application-setup)
- [Templates Setup](#template-setup)
- [Auth Routes](#auth-routes)

## Getting Started

Our app will allow a user to login via their GitHub account. In order for us to be able to implement this functionality, we need to [create an application](https://github.com/settings/applications/new), which you can do from your Settings tab. Under `Developer settings` select `OAuth applications` and register a new application. You can name your application whatever you like and use whatever url you want for the `Homepage URL` input. The important part is the `Authorization callback URL`. Enter `http://localhost:3000/gitauth`, we'll come back to this later. Your form should look something like this:

![AppForm]()

Once you create your application, you will see ClientID and Client Secret values. These values should be kept secret at all times, it's best not to keep them in any file that is under version control. We will be using these values shortly.

## Base Application Setup

Download or clone the sample application from [here](https://github.com/mikaelm1/github-api-tutorial) in order to follow along. Go to the project directory and run `npm install` to install the required dependencies. Now if your run `node app.js` you should see the output:
```
Server running at http://localhost:3000
Press Ctrl-C to terminate
```
If you visit `http://localhost:3000/home` you should see this page:

![HomePage]()

You can also build the app from scratch by following this guide. If you're building from scratch, create the following directory structure:

```
├── app.js
├── package.json
├── public
│   └── stylesheets
│       └── main.css
└── views
    ├── home.handlebars
    ├── layouts
    │   └── main.handlebars
    ├── login.handlebars
    ├── partials
    │   └── header.handlebars
    └── repo.handlebars
```

Most of the app logic will be in a file named `app.js`. The purpose of this tutorial is to demonstrate the use of GitHub's API, so we will not be going into detail about Node.js or express.js. Add the following inside `app.js`:

{% highlight javascript linenos %}
var express = require("express");
var handlebars = require("express-handlebars").create({defaultLayout: "main"});
var bodyParser = require("body-parser");
var request = require("request");

var app = express();
app.set('port', 3000);
app.engine('handlebars', handlebars.engine);
app.set('view engine', 'handlebars');
app.use(express.static(__dirname + "/public"));
app.use(bodyParser.urlencoded({extended: false}));
app.use(bodyParser.json());
{% endhighlight %}

Then we add the following at the bottom of the file to have the server listen for incoming calls:

{% highlight javascript linenos %}
app.listen(app.get('port'), function(){
    console.log("Server running at http://localhost:" + app.get('port'));
    console.log("Press Ctrl-C to terminate");
})
{% endhighlight %}

We'll be using some global variables in `app.js`. Add the following variables after `app.use(bodyParser.json());`:

```javascript
var BASE_URL = "https://api.github.com"
var CLIENT_ID = process.env.CLIENT_ID || "clientIDNotSetWillGet404";
var CLIENT_SECRET = process.env.CLIENT_SECRET || "clientSecretNotSet";
var RANDOM_STRING = "superrandom";
var userToken;
```

As the name implies, `BASE_URL` will be the base url for most of the calls we'll making to GitHub's API. `CLIENT_ID` and `CLIENT_SECRET` will hold the values GitHub gives us when we create an OAuth application, which we did previously. These variables attempt to get the values from environment variables and if those don't exist, they are assigned strings that will fail if we attempt to make authenticated calls with them. You'll need to set these environment variables (environment variables are one effective way of storing secrets). If you're on a Mac or Linux OS, you should be able to do `export CLIENT_ID=yourclientid` and `export CLIENT_SECRET=yoursecret`. 

## Templates Setup

If you're building from scratch, this would be a good idea to fill in all the template files with their corresponding codes. Go [here](https://github.com/mikaelm1/github-api-tutorial/tree/master/views) and copy and paste the template source code into its corresponding file.

## Auth Routes

Let's start with our first route. Inside `app.js`, after `var userToken;`, we have the following:
```javascript
app.get('/home', function(req, res){
    res.render("home", {userToken: userToken});
});
```

If you go to `http://localhost:3000/home` you'll see the `home.handlebars` page rendered. On the right of the navbar there is a `Login` button that will take the user to the login page. The button makes a `GET` request to a `/login` route so let's add that route:

```javascript
app.get('/login', function(req, res){
    var url = "https://github.com/login/oauth/authorize?state=" + RANDOM_STRING + "&client_id=" + CLIENT_ID + "&scope=user%20user:email%20user:follow%20repo%20delete_repo%20admin:org";
    res.render("login", {url: url, userToken: userToken});
});
```

This route builds a url with the `CLIENT_ID` we declared before. This is a good time to talk about the OAuth flow GitHub uses. First, we'll be making a `GET` request to `https://github.com/login/oauth/authorize?client_id=CLIENT_ID&state=randomstring&scope=user:email`, which will then return a session token to the Authorization callback url we provided when we created the OAuth application (http://localhost:3000/gitauth). The token we get from GitHub in our callback route is a temporary token that we need to use in order to get an access_token for the user, which is what we're after. We get the user's token by making another request with the temporary token GitHub gave us.

Before we get to the callback route, let's go over what the parameters in our url mean. The parameter `state` is an optional parameter that we can pass that will hold a random string, that we will be returned in our callback route. Then in our callback route, we can check to make sure that this string matches the one we sent, if it doesn't, then there is a good chance that the request did not originate from GitHub and should be handled appropriately. The `client_id` param is simple the key that will hold our `CLIENT_ID`. The `scope` parameter is where we ask for the different types of access to the user's account. Here we are asking for access to their email, followers, repos, and the ability to delete their repos. You can find out more about the different scope categories [here](https://developer.github.com/v3/oauth/#scopes). Now that we understand the first step of the OAuth flow, let's move on to the callback route:

```javascript
app.get('/gitauth', function(req, res){
    sessionCode = req.query.code;
    randomState = req.query.state;
    if (randomState !== RANDOM_STRING) {
        console.log("State is wrong! ERROR!");
        res.redirect("/home");
        return;
    } 
    var url = "https://github.com/login/oauth/access_token?client_id=" + CLIENT_ID + "&client_secret=" + CLIENT_SECRET + "&code=" + sessionCode;
    request.post({url: url, headers: {'Accept': 'application/json'}}, function(err, response, body){
        if (err) {
            console.log("Error getting user token: " + err);
            res.redirect("/home");
            return;
        } 
        var jsonBody = JSON.parse(body);
        var token = jsonBody.access_token;
        userToken = token;
        res.redirect("/home")
    });
});
```

This is the route that GitHub will send the user's access_token to if our previous request was successful. We first retrieve the sessionCode and state values from the request and check to make sure the request originated from GitHub, and if the state doesn't match, we redirect to the home page. Otherwise, we construct the url we need, and we add the `CLIENT_ID`, `CLIENT_SECRET`, and `sessionCode` as parameters. We'll be making a `POST` request, and if we don't specify that we want a JSON response, GitHub will send the response back as text or xml. Therefore, we add `{'Accept': 'application/json'}` to our request's header. If the GitHub doesn't respond with an error, we parse the body into JSON and assign the access_token to our global `userToken` variable and redirect to the home page. Now that the user is logged in, the `Login` link from the navbar should be replaced with a `Logout` link (but this link will not do anything if clicked on).