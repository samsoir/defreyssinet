---
layout: post
title: "Impementing a Continuous Delivery pipeline (and abandoning SCRUM)"
date: 2012-01-20 09:34
comments: false
published: false
categories: [continuous delivery, system architecture, jenkins]
---

The internet moves quickly. New content is continuously published to the web on a hourly basis the world over. But software changes seem to take considerably more effort to push live than publishing content. The challenges of delivering software on the web is the bottleneck in most organizations. Wouldn't it be great if releasing code to production was as quick as pushing content to a news site? *Continuous Delivery* provides a methodology for deploying code quickly.

<!-- More -->

For many years I have been involved in setting up and managing software deployments for operations on varying scales. But regardless of the size of the website, they all had a rigid release schedule. In many ways the standard agile development process encourages this way of releasing code. Each sprint cycle starts with planning, then development, followed by delivery and finally release. Developers do some work, QA tests the work, and if it is acceptable the work sits in a release branch for a period of time until release. The *"period of time"* could be anything from a couple of hours to several days, even weeks in some circumstances.

This is probably a familiar model to any developer or product owner currently working in an agile way. It would also be incorrect to claim that it isn't a good release model. But this model is not very scalable over time, as it only delivers successfully if the following conditions are met;

 * The development team has at most four members.
 * The team maintains a maximum ratio of two developers to one tester.
 * The scope of the release remains fixed throughout.

Any one of these conditions fail in a release cycle and it will most likely result in a late release. The real challenge is keeping on top of these factors while a development team is growing rapidly- a common problem for a lot of startups. 


I have witnessed this systematic failure of the sprint cycle across many projects. The reason failure is so common cannot be so easily described, but I believe the agile process is part of the cause.



The agile process is not to blame though, as it is purely a framework that is interpreted by development teams. But there is a strong dogma around agile, particularly *SCRUM*, that blinds many from the simple truth- the sprint cycle is failing. The blindness to this truth is probably caused by the fact that the SCRUM gospel has been repeatedly prescribed as a panacea to all software delivery woes. In reality SCRUM is just one project management framework amongst many. Before anyone thinks I am slamming SCRUM, it should be noted that its popularity speaks for itself. I have participated in SCRUM projects that have been successful.

To fix a problem, you have to understand it fully. There are few better ways of understanding a process than to visualize it, process diagrams naturally lend themselves to this function. I have been involved in many projects where no diagrams are published for the development process. It still amazes me how often this is overlooked. 

*Fig 1* below is a simplified representation of a sprint iteration. The process demonstrates the process that progresses a story from conception on the left to reality (i.e. in production) on the right. The diagram is an abstraction as the specific process varies wildly between projects I have experienced.

[{%img http://f.cl.ly/items/0C3F2m0L3E462l3n0t04/Image%202012.01.16%2018:24:27.png 'Fig 1. Simplified Sprint Process Diagram' 'Simplified sprint process diagram illustrating the abstracted tasks in a traditional SCRUM sprint iteration '%}](http://f.cl.ly/items/0C3F2m0L3E462l3n0t04/Image%202012.01.16%2018:24:27.png)




Once the problem is clearly defined then change can be introduced *slowly*. I emphasize slowly because I am yet to encounter an individual or organisation relishes change. 


Finally as an incremental change is introduced, the process should repeat; analysis, review, implement, increment.



 - Natural intuition is to split team
 - QA becomes the constraint (theory of constraints) even with multiple small teams (visualize the flow)
 - Kanban? It may be a better development methodology over SCRUM for larger departments.
 - Monitor the flow. Automation, Jenkins, Git, Jira.
