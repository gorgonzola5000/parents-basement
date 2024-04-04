+++
title = 'How to host a website using Cloudflare Pages'
date = 2024-01-14T07:07:07+01:00
draft = true
+++
## Introduction

This is my first ever blog post so keep this in mind. Have you ever wanted to share your knowledge or just something that you find interesting with other people? Maybe you spent a lot of time figuring something out and want to help some stranger out and get this weird warm feeling that you made some random person on the other side of the globe happy? You can easily do this by creating a blog.

In this tutorial I will be using Cloudflare Pages and Hugo to develop my website.

## Why have I chosen Cloudflare Pages

I don't want to make this post sound like an ad but Cloudflare is pretty great. They lease domains for as low as it gets. Right now, it starts at just $4,16 a year and is dependent on the TLD (e.g. ".com" in "example.com") you choose. Moreover, you can host a static website even on their free plan.

Another advantage of Cloudflare is that you won't have to deal with the hassle of exposing a service to the internet. It may not seem like a big deal, but IPv4 addresses' ports get scanned constantly by bots all over the world in hopes of finding a vulnerability to compromise your website and effectively as much as they can. The best way to expose a service is not to do do it. A few bucks a year isn't a bad price for not having to deal with all of this.

## Purchasing a domain

First we will need a domain. You could technically get away using GitHub Pages but have you ever came across a blog hosted this way while browsing web? Yeah, not really.

Go ahead and register an account on [Cloudflare](https://dash.cloudflare.com/sign-up) if you don't have it already.

Once you have done that, go to "Register Domains" under the "Domain Registration" section and search for an available domain. You can check which TLDs are the best value on [this site](https://www.worthyblog.com/cloudflare-domain-pricing-complete-list/). You will have to fill out your personal details and pay for the domain.

## Creating a website

In this tutorial we will be using Hugo, a static site generator. Static sites, compared to dynamic websites, are quicker since they do not require querying any databases on your server and files are stored in HTML, CSS and JS. They can also be hosted for free on Cloudflare Pages.

First, you will need to [install](https://gohugo.io/installation/) Hugo on your local machine. Once you've done it you can `git clone` this blog's snapshot at the time of writing this article from [this repository](github).