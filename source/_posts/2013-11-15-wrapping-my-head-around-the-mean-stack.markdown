---
layout: post
title: "Wrapping My Head Around the MEAN Stack"
date: 2013-11-15 11:12
comments: true
categories: [Code, Javascript, NodeJS, Express, AngularJS, MongoDB]
published: false
---

I took to using the [MEAN stack boilerplate](http://www.mean.io/) (Mongo, Express, Angular, Node) for my latest project, and I've loved it. In the course of using it, I identified some issues I wanted to address:

## 1. Templating is fragmented between NodeJS and AngularJS
I wanted all templates to be generated server-side with Jade, but I wanted all of the UI logic and URL routing to be driven by Angular. Essentially, the Node/Express layer should function mostly as a JSON api for my Angular client to consume.

## 2. Authenticated user data is exposed to Angular as a global JS variable embedded in the template
This felt like a bit of a no-no. OAuth 2.0 authentication with Facebook/Twitter/Google/etc. needs to take place server-side (I used [Passport](http://passportjs.org/)), but how do you pass that data back to Angular? I incorporated it in the JSON API for Angular to consume.

## 3. I wanted to use HTML5 History API URLs instead of hashbang URLs

## 4. I wanted to use [UI-Router](https://github.com/angular-ui/ui-router) to route my URLS and manage application state in Angular
