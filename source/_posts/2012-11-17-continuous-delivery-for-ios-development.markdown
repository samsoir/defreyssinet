---
layout: post
title: "Continuous delivery for iOS development"
date: 2012-11-17 09:34
comments: false
published: false
categories: [ios, continuous delivery]
---

# Prologue

At the beginning of this year I started working on a new mobile project for my employer. The project marked a major departure from the existing software we wrote and maintained daily. This project would be for mobile devices. But not a new web site for mobile (after all there is no such thing as the _mobile web_), but native mobile application. Up until this point the core business had operated entirely on the web. Given these project requirements, understandably there was a lot of uncertainty surrounding the project. The number of unknowns presented high risk to the project. So in order to try and reduce that risk, I decided we must deliver whatever we eventually built using a _continuous delivery_ workflow.

Continuous delivery is a relatively new pattern that is increasingly becoming popular within software development communities. The general idea is to pull in existing practices of automated testing, continuous integration and continuous deployment to provide a constant and steady stream of improvements to the software being developed. The goal is that the continuous delivery software development process produces high-quality and easily distributed packages to development environments, resulting in the ability to test and feedback quickly. This in turn should enable teams to work more efficiently by quickly fixing bugs and responding to reports from the quality assurance process.

These series of posts will describe the continuous delivery pipeline for iOS that I built for our project. Please note that the technologies and patterns described were accurate as of 2012, but as with all technology subjects, probably will be out of date from the moment this article is published. So if you're reading this and it's 2014 and beyond, please look around for updates and/or alternatives.

<!-- more -->

## Why choose continuous delivery?

Whenever a proposal to change an existing process is made, it is prudent to do due diligence and examine the factors around the proposed change. Teams (or individuals if not in a team) must examine the current process identify what the driving force for change is. It could be that a small change to the existing process could solve any pain points being experienced.

When the continuous delivery pattern was proposed for this project, we were very mindful of the existing process. Up until January of 2012, the development cycle had consisted of two week sprints, followed by an indeterminate amount of time devoted to QA. Once QA was completed, the software was released, usually very early in the morning as releases would invariably require the site to be put into maintenance mode. However there were a number of compounding problems that made this process unworkable. 

Firstly the development team had grown significantly while QA remained roughly the same size. This meant QA was suddenly dealing with over twice the amount of changes compared to a year earlier. The QA process was taking longer to complete, sometimes as long as two weeks on top of the original two weeks taken to develop features in the first place. Additionally the QA process was almost entirely manual.

Another issue was our source control management (SCM) system. The SCM system used at the time was Subversion, and although it had served the company very well for a number of years. Ten developers all committing to separate branches started to make the integration process very expensive.

But the biggest issue was purely the confidence in our abilities to safely change the system. At this point the software had grown to about half a million lines of code, but there was limited global knowledge of the system. There were some unit tests, however they did little to provide much confidence. So the age old problem of changing something over here, breaks something else completely different over there became the status quo. This was a result of the fact that test-driven development was not the order of the day.

Consistently late deliveries and ultimately not a lot of love for the development team was the result of all these issues compounded. Now to be fair, these problems were not unique to our team. I have witnessed codebase rot in many previous projects. However, given the opportunity to start with a fresh slate, we all wanted to make a concerted effort to ensure the new project would not fall victim to the same issues.

Given the state of the union in January, the decision to move to a continuous delivery model was accepted unanimously by the development team, product and operations. Continuous delivery promised a much shorter development cycle, higher quality releases resulting and fewer defects. It was not a hard sell.

It is important to note that all three groups responsible for software were in agreement. Without universal buy-in from all the stakeholders, continuous delivery can not be a reality. This point is so important, I'll reiterate it. Continuous delivery is a big _commitment_, and a commitment that developers, product owners and operations _must all_ agree to.

## Continuous delivery overview

Continuous delivery is a combination of a few existing processes that are probably familiar to the reader already; _automated testing_; _continuous integration_; and _continuous deployment_. 

_Automated testing_ is a generic term that covers a few testing practices. Generally automated testing falls into two categories;

1. _Unit tests_ verify the code written by developers meets the requirements set by business rules. As the name suggests, these tests generally test individual units of code.
2. _Acceptance tests_ verify the application behaves the way the business expects through the interfaces provided, which is usually the user interface. 

The goal of the automated testing phase is to test the business logic and the acceptance criteria in an automated way without human intervention.

_Continuous integration_ is a process of integrating the code created by a single or multiple developers into the main project. The process is designed to reduce issues integrating new code into an existing codebase by testing it before integration. Continuous integration (CI) is usually managed by a server, the current unofficial standard CI server is [Jenkins](http://jenkins-ci.org), others are available.

The CI server in continuous delivery is usually the management system for the entire process. It is responsible for executing the automated testing, and integrating the new code into the project. In our process it also manages the distribution of the new code to our various environments. Additionally it also updates the status of user stories in our project management software. Our Jenkins instance is the real engine room of our continuous delivery process.

The final process in the continuous delivery pattern is _continuous deployment_. Continuous deployment is the act of, as the name suggests, continuously deploying new code to various environments as it becomes available. New code becoming available usually means that it has been fully tested and accepted automatically and now needs to be available for humans to either use in production, or verify within a sandbox or staging environment. Just like the previous other stages, continuous deployment should be autonomous. This autonomy sometimes causes product and operations managers to shudder. It shouldn't, and later I'll discuss how to mitigate any fear.

These three disciplines are then combined into a pipeline, which I imaginatively refer to as the continuous delivery pipeline. The diagram below visualises a basic continuous delivery pipeline.

![iOS Date Picker](/images/cd-pipeline-basic-animation.gif)

This pipeline is a very simple example, but demonstrates the basic concept. The process starts with a change to the source control management system. The continuous integration server detects that change and begins the pipeline execution. First the code is compiled and built ready for testing. Then the unit tests are run, followed by the automated acceptance tests. If those pass then usually a real human will need to run some manual acceptance tests and if all is good there, the build can be deployed. If any of the steps within the pipeline fail, the process is aborted and the development team alerted. Although this pipeline is simple, it actually describes the pipelines for at least one of the projects I currently work with. I tend to find that the simpler the pipeline, the more efficiently it operates.

It is also worth noting that in this pipeline, only one stage requires human involvement; the _User Acceptance Tests_. It is hard to avoid the requirement of a real human in the process. However, by automating all of the unit tests and acceptance tests, the user testing stage should be able to be reduced in scope and thus executed faster. The the goal of any continuous integration pipeline should be to automate as much as possible and remove humans from the process. The balance of automation versus human intervention will differ based on your own project requirements. However it is best to try and automate everything possible, especially any manual test cases that can be scripted.

This has been a very high level overview of _continuous delivery_ without much detail. Hopefully you will have a clearer idea of what continuous delivery is and the main core components  it encompasses;

- Automated testing
- Continuous integration
- Continuous deployment

For the rest of this series of articles on _Continuous Delivery for iOS_, each facet will be looked at in more detail.

- Part 1. Automated testing in iOS. This will cover how to write and test your code within Xcode using the standard SenTest framework along with OCMock. Automated acceptance testing will use Calabash for Cucumber. Finally I'll look at some of the newer iOS testing frameworks based on RSpec.
- Part 2. 



