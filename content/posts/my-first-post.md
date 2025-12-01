---
title: "Blog Set Up"
date: 2019-12-15T16:18:54-05:00
draft: false
tags: ['Hugo', 'n00b']
---

# Framework
This blog is set up using [Hugo](https://gohugo.io/) framework. It's a fast and flexible framework written in Golang, and is a popular static site generator. There are various types of themes that are avaiable for use in Hugo, and the one I'm using is called [m10c](https://themes.gohugo.io/hugo-theme-m10c/).

Hugo is fairly easy to get started with, and the basic use of it might only involve a few CLI command. Most of the configurations and contents can be edited by adding key-value pairs in the `config.toml` file. More flexible content management done through Hugo variables and functions in extra HTML files. I'm yet very new to Hugo, so hopefully I can dive deeper later while mataining this blog and share more of it here (stay tuned!).

# Deployment
A common way to deploy a personal site is to use Github pages (`username.github.io`). While that cound be a pretty good choice, I decided to use [Netlify](https://www.netlify.com/) as an alternative for two reasons: first, I had little experience in setting up and maintaining my own domain and I want to get my hands dirty; second, Netlify provides easy access to CI/CD, SSL certificate, etc., saving me some time to configure these on my own. Netlify is easy to use. There's also an official tutorial to deploy Hugo site to Netlify [here](https://gohugo.io/hosting-and-deployment/hosting-on-netlify/).

The process can be briefly described as following:
1. Sign up on Netlify and bind a new site with the repo of your blog;
2. Purchase you domain and use it to customize the Netlify domain;
3. Set the name servers of your domain to be the ones identified by Netlify;
4. Set the deploy setting (base path, build command, publish directory, etc.) on Netlify, then build and deploy.

Though we could follow the tutorial most of the time, I still made a few mistakes when doing it myself. Some of them are:
1. When configuring the deploy environment variables, make sure to set `HUGO_VERSION` to be the version we are using (which can be shown by `hugo version` command). Otherwise the deploy log may show you the exit code 255.
2. Though we can quickly get our Hugo site to work locally, it might be rendered incorrectly on remote end. In my case I made three mistakes (make sure you avoid these!): 
* when adding a theme repo, I used `git clone` (Netlify dislikes this) instead of `git submodule add`
* in `config.toml` file, I used `http` at first instead of `https`
* in Netlify deploy settings, I used `hugo` as build command. But I had the status of my test post as draft, meaning I need `hugo -D` to include the drafts.

Fortunately, I figured out these mistakes and have this blog working now. So keep lerning, keep blogging from now on!