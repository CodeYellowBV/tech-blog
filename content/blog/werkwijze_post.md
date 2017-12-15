+++
title = "Using Trello for our Development Workflow"
slug = "werkwijze02"
date = "2013-09-30"
description = ""
aliases = ["/werkwijze02.html"]
keywords = ["Workflows", "Werkwijze", "Trello", "Tools"]
categories = [""]
tags = ["Workflows", "Werkwijze", "Trello", "Tools"]
+++

author: Rene Cremers

> This is a repost of an article I wrote a couple of weeks ago, which featured another project as an example, but we received a request to change some aspects of the original story. We decided the quick fix was to pull the original site and rewrite it with other examples.

<img alt="Trello Logo" title="Trello Logo" src="/images/logo-blue-lg.png" style="width: 100%;" />

At Code Yellow we are experimenting with [Trello](http://www.trello.com) for structuring the development process. In essence Trello is a tool developed to manage lists in any way you want. For example you can use it for simple to-do lists, which fits nicely in a software development flow.

The biggest challenge for us was to keep track of several development tracks and feedback loops with the people involved in the development. For this post I will be using examples from one of our back-office software projects [Incenova](//signup.incenova.com) (for a closed beta preview of the project, please [contact us](mailto:contact@codeyellow.nl)). 

<img alt="Incenova teaser" title="Incenova.com" src="/images/incenova_pic01.png" style="width: 100%;" />

Incenova is a software platform specially designed and developed to manage data in organizations. Currently the very first version (which also kind of serves as a proof of concept) is being used at a company to collect all incoming data (from their people in the field), support the office staff to make some modifications on the data and ship the results in specified formats to their end-customers. At Code Yellow we invested heavily in the development of this very first version, so we could deploy the software within two months after the start of the development. Now that the first version of the software is running, we get a lot of feedback, how to further improve the product. To support and streamline the feedback loops back into the development flow, without creating too much overhead we decided to use Trello: 

* we created one Trello board as a global inbox (user feedback, bug reports and feature request appear here), customers are allowed to post stuff on this board,
* next we use (at least) one Internal Development board per product, with the simple stages of the development process,
* finally we have our testing board, which also keeps track which featureset is running on which development and staging server during the testing period. 

We move the cards across the boards, simultaneous to our workflow. Color coding the cards gives an overview at a glance for the team (for instance red means 'critical bug' - pretty straightforward). Because we have a lot of control of the product, we can keep the management and planning overhead to a minimum and focus on the development of some awesome stuff. 

Below is an example of the Development Board, the different stages and lists (yep, we are Dutch).

<img alt="Trello screenshot" title="Trello screenshot" src="/images/trello02.png" style="width: 100%;" />

This post merely scratches the surface of how we use software tools for our development workflow and of course the workflow is something that changes overtime (with new people, new tools and new projects), so this will not be the last post on this subject. A more detialed post will follow.... (sometime). 
