---
layout: post
title: Building APIs with Go - Part I (WIP)
subtitle: How to build an API with Go. Intro and setting up database.
---

This will be the first article in a series where I will document my experience as I build an API for a workout application using Go. These articles are heavily influenced by the [Flask Mega Tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world) written by Miguel Grinberg.

### App Description

The application will be called Nine Chambers. This will only be an API and we will be using Postman and/or curl to make calls against the routes. It's possible that a client for this app will be developed in the future. The initial plan as of this writing is to allow users to create workouts that will consist of different exercises and then be able to mark those workouts as completed. We will add features as the app progresses.

### Requirements

This article assumes that you are familiar with the Go language and already have it installed. Also, being familiar with basics of backend development would be useful.

### Getting Started

First we need to create a directory where the app will live:
```
mkdir nine-chambers 
```

We will be using Postgresql as our database. I like to start off by first thinking about the models that I will need. For now we know we will need User, Exercise, and Workout models. Now we need to figure out how these models will be related to each other.

{% highlight javascript linenos %}
var foo = function(x) { 
    return(x + 5); 
} 
foo(3)
{% endhighlight %}
