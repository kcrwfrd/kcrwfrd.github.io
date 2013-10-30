---
layout: post
title: "Cloning an existing installation of Octopress"
date: 2013-10-30 14:16
comments: true
categories: 
---

The way Octopress deploys to Github Pages is a little funky. When you setup your blog for deployment to Github Pages, a second copy of the repo is instantiated in `./_deploy`, and is checked out to the `master` branch.

You create and edit posts in the `source` branch, in your project root. During deployment, your site gets generated, and the updates copied to `_deploy`. From there, a commit is made to `master`, and a git push is performed, updating the website.

When you clone your blog to a different computer, you need to make sure it is setup in the same way, like so:

``` bash
# Assuming that you have your Ruby environment configured
$ git clone -b source git@github.com:username/username.github.io.git username.github.io
$ cd username.github.io
$ git clone -b master git@github.com:username/username.github.io.git _deploy
$ bundle install
```

Thank you to [zerosharp.com](http://blog.zerosharp.com/clone-your-octopress-to-blog-from-two-places/) and [dblock.org](http://code.dblock.org/octopress-setting-up-a-blog-and-contributing-to-an-existing-one) for helping me figure that one out.
