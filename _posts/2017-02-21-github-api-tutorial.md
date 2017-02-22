---
layout: post
title: GitHub API Tutorial
subtitle: How to use GitHub's API
---

## Description

This guide will demonstrate some of the features of GitHub's API. We will make a very simple app with Node.js that will allow us to authenticate with GitHub and view, create, and delete repositories. The source code for the sample app can found [here](https://github.com/mikaelm1/github-api-tutorial). This tutorial is written as a project for my Web Development class at Oregon State University. 

## Getting Started

Our app will allow a user to login via their GitHub account. In order for us to be able to implement this functionality, we need to [create an application](https://github.com/settings/applications/new), which you can do from your Settings tab. Under `Developer settings` select `OAuth applications` and register a new application. You can name your application whatever you like and use whatever url you want for the `Homepage URL` input. The important part is the `Authorization callback URL`. Enter `http://localhost:3000/gitauth`, we'll come back to this later. Your form should look something like this:

![AppForm]()

Once you create your application, you will see ClientID and Client Secret values. These values should be kept secret at all times, it's best not to keep them in any file that is under version control. We will be using these values shortly.


