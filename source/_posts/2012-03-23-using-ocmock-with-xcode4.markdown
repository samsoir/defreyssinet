---
layout: post
title: "Adding OCMock to Xcode 4 projects"
date: 2012-03-23 18:20
comments: false
categories: [xcode, testing, ocmock]
---

Today I have been adding some new functionality to an iPhone application I am currently developing for [work](http://www.sittercity.com). Being the good developer that I am (_Ed_: and modest too), I started by writing some tests to cover the changes I needed to make. These days Xcode ships with the [SenTest](http://www.sente.ch/software/ocunit/) Objective C unit testing framework, which is great. But SenTest does not include a mocking framework. Fortunately [OCMock](http://ocmock.org/) exists to solve this very problem. However getting it into your iOS project and running takes a little bit of effort.

<!-- more -->

First of all grab the latest copy of the OCMock dmg from [http://ocmock.org#download](http://ocmock.org#download), _version 2.0.1_ at the time of writing. 
Next mount the disk image `ocmock-2.0.1.dmg` (version number may be different) so that
the directory structure of the mounted disk image should resemble the image below
   {% img /images/OCMock.png [OCMock directory structure] %}
 
At this point the OCMock library is ready to be added to your project. The disk image provided supplies linked libraries for iOS and Mac OS X, as well as the source. Naturally the project source is also available on [Github](http://github.com/erikdoe/ocmock). Be warned, the Github source will require compiling before it can be used in your own tests. If you are not comfortable with that concept, it is best just to use the precompiled binaries.

Before importing the OCMock library into your Xcode 4 project, create a folder for this library name something similar to `$(SRCROOT)/tests/libraries`. `$(SRCROOT)` is xcode shorthand for the project root folder for anyone scratching their head. In Xcode create a group within the `tests` group called `libraries` to mirror the layout of the filesystem. Now drag the contents of the OCMock _iOS_ folder on the mounted drive to the new `libraries` group within Xcode created previously.

Xcode _might_ be clever and realize what has just happened at this point and added `libOCMock.a` to your tests targets _Link Binary With Libraries_ build phase. If it has not, then you can drag `libOCMock.a` from your filetree into the _Link Binary With Libraries_ build phase to add it. Before continuing we need to ensure that Xcode is including the library and headers when the tests are run. Within the test target, check the _Build Settings_ entry for _Header Search Paths_ and _Library Search Paths_ includes `$(SRCROOT)/tests/libraries`.

Finally and most **importantly** Xcode needs to tell the compiler to force load the OCMock libraries at compile time, otherwise your tests will crash. Within the _Build Settings_ in your test build target add the following line to the _Other Linker Flags_ entry; `-force_load $(SRCROOT)/tests/libraries/libOCMock.a`.  

OCMock is now ready for use and bar any other unforseen problems, testing should be uneventful. The last step of instructing the compilier to force load the library caused me to loose 30 minutes this afternoon, so hopefully this will help anyone else having similar issues.
