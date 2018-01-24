+++
title = "Queue"
author = "Stefan Majoor"
slug = "Queue"
date = "2015-10-22"
description = ""
aliases = ["/queue.html"]
keywords = ["Projects", "PHP", "FuelPHP", "Queue", "Open Source", "Software Development"]
tags = ["Projects", "PHP", "FuelPHP", "Queue", "Open Source", "Software Development"]
+++

In our organization we frequently need to execute some scripts that are either time consuming, or very heavy on the server. Most of the times these scripts needn't necessarily be executed synchronously. Therefore we use a queuing system to execute those scripts when the time is better.  For a long time we used an old open source PHP queuing system, named [fuel-queue](https://github.com/kavinsky/fuel-queue). This was good for the basic stuff we did with it, however this system had some major drawbacks. It had no exception management whatsoever, which meant that jobs now and then just disappeared. Lately we have encountered more and more projects that would need queuing. We made the decision to create a new queuing system. We really liked the idea behind fuel-queue, and have made our new system in such a way that it kept this idea. The best thing is that we made this new queuing system is completely open source!

### Presenting [Queue](https://bitbucket.org/codeyellow/queue)
Today we finally present our latest project. Queue is a queuing system made for the FuelPHP framework. This plugin is written from the ground up, taking what we liked in the old fuel-queue project in mind. The main focus during development was stability. This means that great care is taken in error handling, ensuring that no data is lost. Furthermore, a lot of unittests are written. Great care is taken in writing the software. The code is well documented, and can easily be adapted to your needs. This all results in a highly reliable software package which can easily be used in your own projects. The best news is that the project is released under MIT licence, so you can use the source code without restrictions.
 
<center> [Download Queue Now](https://bitbucket.org/codeyellow/queue) </center>
 
### Features
 *  A wide variety of features is available in this version of the queueing system.  A small selection of the features:
 * Simple usage, easy to integrate in your project. Great care is taken to make good documentation, and to create a software package which can easily be adapted to your personal needs.
 * Different queues can run in parallel. You can turn queues on and off whenever you want, meaning that you yourself can decide when you want to run which functions. This makes it possible for example to run tasks that are very heavy on the system on quieter times, preventing hinder to the end user of your product.
 * Executes any static function, with arguments.
 * Set priorities for the job. High priority jobs are executed earlier.
 * Delayed execution. Do you need to send an email tomorrow? You can just add it to the queue now. The queue will take care that it will be send whenever you want it to be send.
 * Detailed error messages. Of course jobs may fail because of a wide variety of reasons. Luckily the application will log the error messages, making it easier for you to find errors in your code.
 * Status panel showing all the jobs. A status panel for the queuing system is available, and can easily be integrated in your projects. This panel will show you the status of the different jobs in the project.
 
### Why use a queuing system?
Although you may not realize it, your software may benefit greatly from usage of queuing system.  From our perspective there are three reasons why a queuing system may come in handy:

 * Resolve long execution times. Now and then a user performs an action that needn't necessarily be executed at that moment. Slow page responses have a major impact on consumers. Therefore, using a queue can help to improve consumer satisfactions.
 * There are certain types of actions that do require a lot of power from the server. Think for example when a report needs to be made from a lot of data. This means that you need to have an extremely well performing server, or you will experience some performance issues. With a queuing system you can delay this actions until times that are less busy, such that users won't notice it.
 * A good usage for a queuing system is for delayed execution of processing. For example, do you need to send a mail in 24 hours? You can add it now to the queue, and you can be sure that it will be executed in 24 hours.

### Want to contribute?
If you have an awesome idea for the queuing system you are of course welcome to help. For development we are using BitBucket. This enables you to open new tickets. Just open a tickets if you find a bug, think of a good feature, and just have got an awesome idea in general.

