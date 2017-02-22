---
layout: post
title: GitHub API Tutorial
subtitle: How to use GitHub's API
---

## Description

This guide will demonstrate some of the features of GitHub's API. We will make a very simple app with Node.js that will allow us to authenticate with GitHub and view, create, and delete repositories. We will also add the ability to create a webhook for a user's repository. The source code for the sample app can found [here](https://github.com/mikaelm1/github-api-tutorial). This tutorial is written as a project for my CS290 class at Oregon State University. 

- [Getting Started](#getting-started)
- [Base Application Setup](#base-application-setup)
- [Templates Setup](#templates-setup)
- [Auth Routes](#auth-routes)
- [Repo Routes](#repo-routes)
    - [Get Repo Data](#get-repo-data)
    - [Create Repo](#create-repo)
    - [Delete Repo](#delete-repo)
    - [Create Repo Webhook](#create-repo-webhook)
- [Conclusion](#conclusion)

## Getting Started

Our app will allow a user to login via their GitHub account. In order for us to be able to implement this functionality, we need to [create an application](https://github.com/settings/applications/new), which you can do from your Settings tab. Under `Developer settings` select `OAuth applications` and register a new application. You can name your application whatever you like and use whatever url you want for the `Homepage URL` input. The important part is the `Authorization callback URL`. Enter `http://localhost:3000/gitauth`, we'll come back to this later. Your form should look something like this:

![AppForm](https://cloud.githubusercontent.com/assets/16492296/23198297/69820a58-f87b-11e6-93b9-f6fbfcd2b560.png)

Once you create your application, you will see ClientID and Client Secret values. These values should be kept secret at all times, it's best not to keep them in any file that is under version control. We will be using these values shortly.

## Base Application Setup

Download or clone the sample application from [here](https://github.com/mikaelm1/github-api-tutorial) in order to follow along. Go to the project directory and run `npm install` to install the required dependencies. Now if your run `node app.js` you should see the output:
```
Server running at http://localhost:3000
Press Ctrl-C to terminate
```
If you visit `http://localhost:3000/home` you should see this page:

<img height="530" width="800" alt="screen shot 2017-02-21 at 9 19 53 pm" src="https://cloud.githubusercontent.com/assets/16492296/23198314/8dc46bd6-f87b-11e6-9df0-646d49912303.png">

You can also build the app from scratch by following this guide. If you're building from scratch, create the following directory structure:

```
├── app.js
├── package.json
├── public
│   └── stylesheets
│       └── main.css
└── views
    ├── layouts
    │   └── main.handlebars
    ├── partials
    │   └── header.handlebars
    ├── home.handlebars
    ├── login.handlebars
    └── repo.handlebars
```

Most of the app logic will be in a file named `app.js`. The purpose of this tutorial is to demonstrate the use of GitHub's API, so we will not be going into detail about Node.js or express.js. Add the following boilerplate code inside `app.js`:

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

As the name implies, `BASE_URL` will be the base url for most of the calls we'll be making to GitHub's API. `CLIENT_ID` and `CLIENT_SECRET` will hold the values GitHub gave us when we created an OAuth application, which we did previously. These variables attempt to get the values from environment variables and if those don't exist, they are assigned strings that will fail if we attempt to make authenticated calls with them. You'll need to set these environment variables (environment variables are one effective way of storing secrets). If you're on a Mac or Linux OS, you should be able to do `export CLIENT_ID=yourclientid` and `export CLIENT_SECRET=yoursecret` from the terminal to set up the environment variables. 

## Templates Setup

If you're building from scratch, this would be a good time to fill in all the template files with their corresponding codes. Go [here](https://github.com/mikaelm1/github-api-tutorial/tree/master/views) and copy and paste the template source code into its corresponding file.

## Auth Routes

Let's start with our first route. Inside `app.js`, after `var userToken;`, we have the following:
```javascript
app.get('/home', function(req, res){
    res.render("home", {userToken: userToken, message: flashMessage});
    flashMessage = null;
});
```

If you go to `http://localhost:3000/home` you'll see the `home.handlebars` page rendered. On the top right of the navbar there is a `Login` button that will take the user to the login page. The button makes a `GET` request to a `/login` route so let's add that route:

```javascript
app.get('/login', function(req, res){
    var url = "https://github.com/login/oauth/authorize?state=" + RANDOM_STRING + "&client_id=" + CLIENT_ID + "&scope=user%20user:email%20user:follow%20repo%20delete_repo%20admin:org";
    res.render("login", {url: url, userToken: userToken});
});
```

This route builds a url with the `CLIENT_ID` we declared before. This is a good time to talk about the OAuth flow GitHub uses. First, we'll be making a `GET` request to `https://github.com/login/oauth/authorize?client_id=CLIENT_ID&state=randomstring&scope=user:email`, which will then return a session token to the Authorization callback url we provided when we created the OAuth application (`http://localhost:3000/gitauth`). The token we get from GitHub in our callback route is a temporary token that we need to use in order to get an access_token for the user, which is what we're after. We get the user's token by making another request with the temporary token GitHub gave us.

Before we get to the callback route, let's go over what the parameters in our url mean. The parameter `state` is an optional parameter that we can pass that will hold a random string, that we will be returned in our callback route. Then in our callback route, we can check to make sure that this string matches the one we sent, if it doesn't, then there is a good chance that the request did not originate from GitHub and should be handled appropriately. The `client_id` param is simply the key that will hold our `CLIENT_ID`. The `scope` parameter is where we ask for the different types of access to the user's account. Here we are asking for access to their email, followers, repos, and the ability to delete their repos. You can find out more about the different scope categories [here](https://developer.github.com/v3/oauth/#scopes). Now that we understand the first step of the OAuth flow, let's move on to the callback route:

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

This is the route that GitHub will send the user's `access_token` to if our previous request was successful. We first retrieve the `sessionCode` and `state` values from the request and check to make sure the request originated from GitHub, and if the state doesn't match, we redirect to the home page. Otherwise, we construct the url we need, and we add the `CLIENT_ID`, `CLIENT_SECRET`, and `sessionCode` as parameters. We'll be making a `POST` request, and if we don't specify that we want a JSON response, GitHub will send the response back as text or xml. Therefore, we add `{'Accept': 'application/json'}` to our request's header. If GitHub doesn't respond with an error, we parse the body into JSON and assign the `access_token` to our global `userToken` variable and redirect to the home page. Now that the user is logged in, the `Login` link from the navbar should be replaced with a `Logout` link (but this link will not do anything if clicked on). From this point on, whenever we make a call to an endpoint that is only available to an authenticated user, we will provide the `userToken` in the url as a parameter, `?access_token=userToken`.

## Repo Routes

#### Get Repo Data

Now that we have an authenticated user, we can begin making calls to the more fun features of GitHub's API. In this section we will learn how to get data related to a user's repositories, both an authenticated and unauthenticated user, and see how the two different calls are made. GitHub has many different endpoints and not all of them require an authenticated user, but many do. We will also implement features that will allow an authenticated user to create and delete their own repositories.

On the home page, the user can fetch all the repositories for a given user, and if the user is logged in and leaves the input field empty, then their own repositories will be fetched. Clicking on the `Fetch My Repos` button sends a `POST` request to this route:

```javascript
app.post('/repo', function(req, res){
    userName = req.body.username;
    var url = BASE_URL;
    var useAuth = true;
    if (userName === "") {
        // get logged in user's repos 
        url += "/user/repos";
    } else {
        // get repos for unauth user 
        useAuth = false;
        url += "/users/" + userName + "/repos";
    }
    var repoData = []
    getData(url, useAuth, function(body){
        if (body === ""){
            console.log("Error");
        } else {
            body.forEach(function(r){
                var repo = {};
                repo.repoName = r.name;
                repo.repoCreated = r.created_at;
                repo.repoDesc = r.description;
                repoData.push(repo);
            });
        }
        repoData.sort(byDate); // sort by newest created at top
        var dataCount = repoData.length;
        res.render("repo", {userName: userName, repoArray: repoData, dataCount: dataCount, userToken: userToken});
    });
});
```

The urls for getting a user's repositories are different depending on if we are getting the data for an authenticated user or for someone else. If we're getting the data for an unauthenticated user, we will only get back their public data. The url for an authenticated user is `/user/repos`, while the one for an unauthenticated user is `/users/:username/repos`, where `username` is the GitHub username of some user. We will be making multiple `GET` requests in our app and so it's a good idea to have a generic function that can handle all of those calls, which means less code duplication:

```javascript
function getData(url, withAuth, callback) {
    if (withAuth && userToken) {
        url += "?access_token=" + userToken;
    }
    request.get({url, headers: {'user-agent': 'node.js'}}, function(err, response, body){
        if (err){
            console.log("Error making GET call: " + err);
            callback("");
            return;
        }
        var jsonBody = JSON.parse(body);
        callback(jsonBody);
    })
}
```

This helper function takes a url string, a `boolean` value to determine wheter the call is to an authenticated route, and a callback function. We need a callback function because we're making an http request and don't know how long it will take for us to get a response (another way of doing this would be to use [promises](https://developers.google.com/web/fundamentals/getting-started/primers/promises)). We need to set a `user-agent` header value, otherwise GitHub will return an error. If we get an error in the response we send back an empty string, else we parse the body to JSON and send it back via the callback function. Back in our `POST` `/repo` route, we decalare a `repoData` array that will hold our repo objects. We iterate through the JSON response and create a `repo` object with three fields: `repoName`, `repoCreated`, and `repoDesc`. We then sort the array using the helper function shown below, and then pass this array to `repo.handlebars` template, which will display the repositories as a list.

```javascript
function byDate(lh, rh) {
    if (lh.repoCreated < rh.repoCreated) {
        return 1;
    } else if (lh.repoCreated > rh.repoCreated) {
        return -1;
    }
    return 0;
}
```


#### Create Repo

Now that we know how to fetch data for a user's repositories, it's time to create a new repository for a user. This will require that the user be authenticated. A user is authenticated if our global `userToken` variable has a value. The home page has a form that the user can use to create a repository by entering a name for it and a description. The submit button of this form sends a `POST` request to `/repo/create`:

```javascript
app.post('/repo/create', function(req, res){
    // Only for authenticated users 
    var repoName = req.body.reponame;
    if (!userToken || !repoName) {
        flashMessage = "Need to be logged in to create repository";
        res.redirect("/home");
        return;
    }
    var repoDesc = req.body.description;
    var url = BASE_URL + "/user/repos"
    // will default to MIT license and no README
    var postBody = {'name': repoName, 'description': repoDesc, 'license_template': 'mit'};
    postData(url, true, postBody, function(body){
        if (body === ""){
            console.log("Error getting POST response");
            res.redirect("/home");
            return;
        }
        var repoData = []
        var repo = {};
        repo.repoName = body.name;
        repo.repoCreated = body.created_at;
        repo.repoDesc = body.description;
        var repoOwner = body.owner.login;
        repoData.push(repo);
        res.render("repo", {userName: repoOwner, repoArray: repoData, dataCount: 1, userToken: userToken, message: "Successfully created repo"});
    });
});
```

If the user is not authenticated or does not provide a repository name, they get redirected back to the home page and nothing happens. Otherwise, we construct the url, which for creating a repository is a `POST` to `/user/repos`. It's safe to assume that we will be making multiple `POST` requests and so we make another generic function that will handle all of our `POST` requests:

```javascript
function postData(url, withAuth, body, callback) {
    if (withAuth && userToken) {
        url += "?access_token=" + userToken;
    }
    request.post({url, headers: {'user-agent': 'node.js', 'Content-Type': 'application/json'}, json: body}, function(err, response, body){
        if (err){
            console.log("Error making POST call: " + err);
            callback("");
            return;
        }
        callback(body);
    });
}
```

The helper function takes a url string, a boolean value to determine if it's an authenticated call, a dictionary for any possible values to be passed in the request body, and a callback function which will return the data to the calling function. The request edits the url if necessary based on if the call is to an authenticated user or not. It also sets a `user-agent` header and a `Content-Type` header to get the response in JSON format. If we get an error from GitHub, it returns an empty string, else it returns the JSON response.

Back in our route to create a repository, before we make the request, we construct the body values we want to send. In our case, we will be sending the name and description the user entered, along with the `license_template` key that we set to `mit`, which will generate a `LICENSE` file with an MIT License in the newly created repository. There are many more options we can set, such as creating a `README`, when creating a repository and you can read about them [here](https://developer.github.com/v3/repos/#create). Once we have the url and body constructed, we call our helper `postData` function. If we don't get back an error, we construct a `repo` object from the JSON response by parsing out the name of the repo, date it was created, its description, and its owner's username. We then render the `repo.handlebars` template passing in the `repo` object in an array (the template is expecting an array) along with some other values like a `message` field, which the template will render as an alert at the top of the screen. 


#### Delete Repo

The user of our application now has the ability to fetch the data for a user's repositories and can even create a new repository. Now let's add the ability to delete a repository. If you remember earlier, when authenticating the user, we added the `delete_repo` scope request because without it we wouldn't be able to delete a user's repository. Our home page has a form that let's the user delete one of their repositories by entering their GitHub username along with the name of the repository. This will not work for unauthenticated users. The form sends a `POST` request to route `/repo/delete`. We also add a helper function for making `DELETE` requests: 

```javascript
// helper function for sending DELETE requests
function deleteData(url, callback) {
    if (!userToken) {
        callback("");
        return;
    }
    url += "?access_token=" + userToken;
    request.delete({url: url, headers: {'user-agent': 'node.js', 'Content-Type': 'application/json'}}, function(err, response, body){
        if (err){
            console.log("Error making DELETE call: " + err);
            callback("");
            return;
        }
        callback({'message': response.headers.status});
    });
}
// Route to delete repo
app.post('/repo/delete', function(req, res){
    var userName = req.body.username;
    var repoName = req.body.reponame;
    if (!userToken || !userName || !repoName) {
        flashMessage = "Error deleting repo";
        res.redirect("/home");
    }
    var url = BASE_URL + "/repos/" + userName + "/" + repoName;
    deleteData(url, function(body){
        if (body === "") {
            console.log("Got empty body for delete repo");
            flashMessage = "Error deleting repo";
            res.redirect("/home");
            return;
        }
        flashMessage = "Successfully deleted repo";
        res.redirect("/home");
    });
});
```

The `deleteData` function takes a url string and a callback function, which will return the response. There is no need for a boolean checking for auth because all `DELETE` calls will require an authenticated user and if the user is not authenticated, `deleteData` returns an empty string via the callback function. Otherwise, it makes the `DELETE` request to GitHub. If there is an error, the callback function returns an empty string. If our request is successful, GitHub will return a status of 204, which can be found in the header of the response. GitHub sends an empty body value on a successful request.

Back in our route that will be making the `DELETE` request, we make sure that the user is authenticated and that they entered both a username and reponame that they want deleted. If either of these is false, we redirect back to the `/home` route with an error message. Otherwise, we build up the url and call the `deleteData` function. If we don't get an empty `body` response from `deleteData`, we know we successfully deleted the repository. We then redirect to the `/home` route with a success message.

#### Create Repo Webhook

A webhook is kind of like a subscription to some events, where if any of the events we subscribe to occur, we get notified. GitHub has a variety of webhooks that a user can subscribe to and you can read all about them [here](https://developer.github.com/webhooks/). We will be making a webhook that will send a notification whenever the user's repository is starred, gets a pull request, or whenever any code is pushed to the repository. The home page of our application has a form that gets the user's GitHub username and the name of their repository and sends a `POST` request to `/repo/webhook`:

```javascript
app.post('/repo/webhook', function(req, res){
    var userName = req.body.username;
    var repoName = req.body.reponame;
    if (!userToken || !userName || !repoName) {
        console.log("Need user and repo to create a webhook");
        flashMessage = "Need username and name of repo to create a webhook";
        res.redirect("/home");
        return;
    }
    var url = BASE_URL + "/repos/" + userName + "/" + repoName + "/hooks";
    var events = ["push", "pull_request", "watch"];
    var callbackURL = "http://localhost:3000/repo/webhook/callback";
    var config = {"url": callbackURL, "content_type": "json"};
    var reqBody = {"name": "web", "active": true, "events": events, "config": config}
    postData(url, true, reqBody, function(body){
        if (body === "") {
            flashMessage = "Error creating webhook";
            res.redirect("/home");
            return;
        } 
        flashMessage = "Successfully created webhook.";
        var hook = {
            "id": body.id,
            "test_url": body.test_url,
            "ping_url": body.ping_url,
            "name": body.name
        };
        res.redirect("/home");
        testWebhook(hook);
    });
});

// callback route for the webhook
app.post('/repo/webhook/callback', function(req, res){
    console.log("Need to have this route be accessible from the Internet in order to get any requests. Also, set up websockets or some other way of pushing from the server to display this message on the client side!");
});
```

First, we make sure that the user is logged in and that they entered their username and name of repository. If not, we set an error message and redirect back to the home page. Otherwise, we construct the url `/repos/:owner/:repo/hooks` along with the types of events we want to subscribe to. There are many different event types to choose from and you can see them all [here](https://developer.github.com/webhooks/#events). Next we construct a callback url, which is the url we want GitHub to send a notification to in the event that one of the events we subscribed to is triggered. GitHub's webhooks will send a `POST` request and that is why our callback route is a `POST` route. All of these values will go into the body of our request, so we add them and send the `POST` request. If we don't get an error, we construct a `hook` object from the JSON body, redirect to the home page, and call a function that will test the new webhook:

```javascript
function testWebhook(hook) {
    console.log("Testing hook:");
    console.log(hook);
    var url = hook.test_url + "s";
    console.log(url);
    request.post({url: url, headers: {'user-agent': 'node.js', 'Content-Type': 'application/json'}}, function(err, response, body){
        if (err){
            console.log("Error making POST call: " + err);
            return;
        }
        console.log("Successfully sent test request to webhook");
        console.log(body);
    });
}
```

This function takes a webhook object and uses it's `test_url` property to send a test request. We need to add an `s` to the end of the `test_url` because as of this writing, the `test_url` parameter that GitHub sends does not exactly match the url used for testing a webhook, it's mising an `s`. Once we get the response we print out the body to the console. Here is the response we get from GitHub: `{"message":"Not Found","documentation_url":"https://developer.github.com/v3"}`. This happens because the callback we setup for the webhook, `/repo/webhook/callback`, is not accessible from the Internet. If developing locally, you will need to expose your server to the Internet by using a tool called [ngrok](https://ngrok.com/) in order to receive notifications from the webhook. I'll leave this as an exercise for you to do on your own. 

## Conclusion

We've created an application that allows a user to get information about their own repositories or those of another GitHub user. We've implemented GitHub's OAuth flow, which allowed us to create and delete repositories belonging to the logged in user. A user also has the ability to create a webhook for one of their repositories. These are just some of the many endpoints that GitHub's API exposes. There are many more things you can build with [their API](https://developer.github.com/v3/). And for those who rather not write all of these functionalities from scratch, you'll be able to find a [library](https://developer.github.com/libraries/) in one your favorite languages.  