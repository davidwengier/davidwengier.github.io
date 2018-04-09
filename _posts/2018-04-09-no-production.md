---
layout: post
title: ! I don't want to remote into production
published: true
---

A friend of mine tweeted this article, and excellent summary today, about a recent production outage at Travis CI:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">A read-only prod is a slightly safer prod, if you don&#39;t have access to truncate all tables, then it is less likely to happen :P<a href="https://t.co/7urWybtZA8">https://t.co/7urWybtZA8</a></p>&mdash; NullOps (@NullOpsio) <a href="https://twitter.com/NullOpsio/status/983237339634810880?ref_src=twsrc%5Etfw">April 9, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

I personally feel even stronger about this: I don't want access to production, read only or otherwise. I don't even want to install the database management software, be it SQL Server Management Studio or MySQL client, if I can help it.

I started a new job today and during the onboarding it was mentioned that production server access was locked down by IP so if needed, it would have to be done via a VPN if I wasn't in the office. Not to speak ill of my new employer, let me be clear this was only mentioned as a "just in case" option, as part of an answer to someone elses direct question, and not part of regular onboarding information that people need to know. In fact everyone there knows that its not a good idea, and something that should be worked to remove in future, and helping to do that is a large part of my role, but I digress.

I said flat out: I don't want access.

## I don't trust myself, and you should too

Eventually everyone has that moment where they do the wrong thing. Perhaps they run an UPDATE statement without a WHERE clause, or they're connected to the wrong environment when they tweak some configuration value. Most people, you hope, only make these sorts of mistakes once, and I've made mine (quite a few years ago, don't worry!) and I don't want to do it again. The easiest way I can think of to guarantee that is to simply make it impossible.

Yes, I can double check everything I do.
Yes, I can get people to check over my shoulder, or review.
Yes, I can work through checklists with well documented steps.

Or I can just make incorrect actions impossible. If I can't remote to a server, I can't be remoted to the wrong server. If I can't connect to a database, I can't forget part of a script or statement. 

## If you want something fixed, make it a problem

The ideal situation for production environments (or indeed most other environments) is that their setup and configuration is completely automated and needs no manual work. By making manual work impossible you force people and teams to do the necessary work to create tooling to enable that. There can be no shortcuts taken and temptation is removed by virtue of a firm wall between developers and where their code is deployed. If I need to make a change to a database schema I want the only option to be to create a change script, or similar. I would apply that script to my dev environment and in time to testing and staging environments.

If the only possible path is automated tooling then by the time a deployment to production is done not only are you guaranteed that the tooling is in place, you've also ideally tested its execution a few times in various environments.

## Data is cheating too

If you want to take this to the logical extreme, which I do, I also don't want to modify any data in the database directly. If I'm building out a new feature that requires data in a specific shape, either to run or simply for me to manually test with, then I would rather build the seeding scripts, or ideally the data management front end, first. The seeding scripts can help with functional/integration tests, or the data management work presumably solves some future need in the product (and if thats not the case then don't do it! But also consider why that data is needed).

## Codified knowledge is shared knowledge

The other advantage of creating tooling, scripts or otherwise automating things is that anything that is codified and committed to a source repository is something that other people can reason about, read and hopefully understand. There is nothing that reduces [Bus Factor](https://en.wikipedia.org/wiki/Bus_factor) like having a series of scripts and tools that anyone can pick up.

Essentially, avoid making manual work a necessity as much as is humanly possible. Of course be pragmatic about things, in particular there is nothing wrong with doing manual work once to get a feel for it, before automating, but I loathe to do something twice or three times. Additionally some manual work can be fun to opt in to, and I specifically avoid using tools like Ninite or Chocolately for this reason, as I simply enjoy the process of building a new machine.

But I don't want to touch the production environment.