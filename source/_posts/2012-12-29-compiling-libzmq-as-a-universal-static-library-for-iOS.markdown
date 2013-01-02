---
layout: post
title: "Compiling libzmq as a universal static library for iOS"
date: 2013-01-01 13:53
comments: false
categories: [ios, libzmq, xcode, tutorial]
---

I am currently playing around with the [ZeroMQ](http://www.zeromq.org) messaging framework and iOS. This article describes the challenges encountered when integrating ZeroMQ with my iOS project and how I mitigated them. This resulted in a tool to automate producing a single libzmq static library for i386, armv7 and armv7s architectures. 

For the uninitiated, ZeroMQ provides a simple way to pass messages between processes, whether they are local or remote. It can use TCP, UDP, inproc, multicast and several protocols as the transport mechanism. In the past I have heard ZeroMQ (or _ØMQ_ from now on) referred to as a "sockets framework". Although this doesn't capture the entirety of what ØMQ is, it does provide a nice metaphor for my particular use case.

The project that required ØMQ in this instance was a proof of concept for another project I am currently engaged in. The application in question requires two iOS devices to communicate with each other in realtime across a network. ØMQ provides the communication layer, coupled with [Apple's Bonjour](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/NetServices/Articles/about.html) zero config network discovery protocol. As ØMQ is a C based library, integrating it with an iOS project should require relatively little effort. Obtain the [libzmq source](http://www.zeromq.org/area:download), compile and add to the Xcode project. But as it turned out, producing a static library for iOS development took some additional effort.

<!--more-->

Because Apple provide an iOS simulator for development rather than a more useful iOS emulator, any static library must be compiled for the simulator and then again for iOS devices. When an iOS application runs in the simulator, it is using the [x86](http://en.wikipedia.org/wiki/X86) (or i386) [instruction set](http://en.wikipedia.org/wiki/Instruction_set); iOS applications are not 64-bit, yet. However all iOS devices use an [ARM](http://arm.com) based CPU of some description, requiring different instructions from their x86 cousins. Apple currently support two [ARM architectures](http://en.wikipedia.org/wiki/ARM_architecture), named armv7 for iPhone 4/4S plus the iPad 2 and 3. And armv7s for iPhone 5 and the new iPad 4 (retina display)<sup>\[[1](#f1)\] \[[2](#f2)\]</sup>. Due to this, my project requires ØMQ support for the i386, armv7 and armv7s architecture. It is possible to compile the ØMQ static library three times, add them all to the project and only link to the correct version at compile time. But this can be fiddly, especially when dealing with two almost identical architectures such as armv7 and armv7s.

Fortunately Apple have encountered this multiple architecture problem in their recent past, when moving from the PowerPC to the current x86/x86_64 architecture. Apple wanted to ensure the transitional period was as smooth as possible for their users. So they created the concept of the universal binary, which they imaginatively named a "Universal Application". To the end user, they could execute the same application on their existing PowerPC powered iMac G5 or a shiny new Intel powered MacBook Pro without issue. However, this apparently magical execution of the same code on two different CPU architectures was nothing more than a cheap parlour trick. In reality a universal binary is a single resource that encapsulates the compiled code for two or more CPU architectures. Although the PowerPC to Intel transition was completed many years ago, the tools to produce these universal binaries remain. Only today iOS developers are combining x86 with ARM instructions.

It is worthwhile to note that universal libraries are also referred to as "fat" libraries, because they are larger than their single architecture counterparts. The overall iOS application size is a factor in how they can be downloaded to the device. Applications weighing in at over 20 megabytes will require a wifi connection to be downloaded, and all applications must be less than two gigabytes. Although the two gigabyte ceiling is unlikely to be a problem due to a fat library, the 20 megabyte limit could cause a concern. If size is a factor for your application then it is probably worthwhile to only include the correct library for the target architecture. I'll touch on this briefly a bit later.

The ØMQ page describing how to build [libzmq for iOS](http://www.zeromq.org/build:iphone) details the [cross-compiling](http://en.wikipedia.org/wiki/Cross_compiler) process. This is fine for creating an armv7 static library from an Intel powered Mac. But as discussed earlier, this library would not be usable by the iOS simulator. I require a universal static library for libzmq that can execute on all of the required architectures.

The process of creating a universal static library for libzmq is conceptually straight forward, but as it turns out it is quite cumbersome. In theory, all that is required is to compile the libzmq library for i386, armv7 and armv7s and then glue them together into the universal binary. Compiling the binary for ARM requires cross-compiling using a version of [GCC](http://gcc.gnu.org) with instructions for that CPU. All Mac systems with Xcode installed contain the ARM version of the compiler. The only challenge is figuring out the correct compile flags to set to ensure the right binary is produced.

To save a lot of time I automated the process into a single bash script. I have placed this script on Github for anyone who requires it; available from <https://github.com/samsoir/libzmq-ios-universal>. It is licensed under the open source [ISC license](http://opensource.org/licenses/ISC), so modifications and extensions are welcome. For those not willing to look at the [source](https://github.com/samsoir/libzmq-ios-universal/blob/master/compile_libzmq_ios_universal), the script performs the following operations.

1. Compiles a libzmq static library for the armv7, armv7s and i386 architectures<sup>\[[3](#f3)\]</sup>.
2. A tool called [lipo](https://developer.apple.com/documentation/Darwin/Reference/ManPages/man1/lipo.1.html) is used to perform the gluing of the three libraries into a single universal static library.
3. The header files for the library are copied into the universal folder.
4. Finally the script tidies up after itself and then provides instructions on what to do next.

Once the script has finished executing successfully, there will be four versions of the libzmq library. One version for each architecture, and of course the universal version. The individual architecture versions of libzmq remain so that you can use them in place of the universal version if application size is an issue.

The current project on Github is work in progress. It mostly supports my specific requirements at this time. If you try it and it does not work as designed for you, please do tell me or create a pull request. I do intend to add features and more support over time.

## Tutorial

To use the **libzmq-ios-universal** compilation script, you will require a Mac with OS X Snow Lion with Xcode 4.3.3 or later installed. Additionally, [Automake](http://www.gnu.org/software/automake/) is also required. I used [Homebrew](http://mxcl.github.com/homebrew/) to obtain Automake in this tutorial, but you can also download and compile it yourself if you do not have, or want to have, Homebrew installed.

First of all, if you have not done so already, install Automake using Homebrew. _Or build manually if preferred_

In a terminal type;

{%codeblock%}
brew install automake
{%endcodeblock%}

To verify whether automake is successfully installed;

{%codeblock%}
automake --version
{%endcodeblock%}

This should produce output similar to what is shown below. If automake is not found try reinstalling.

{%codeblock%}
automake (GNU automake) 1.12.3
Copyright (C) 2012 Free Software Foundation, Inc.
License GPLv2+: GNU GPL version 2 or later <http://gnu.org/licenses/gpl-2.0.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Written by Tom Tromey <tromey@redhat.com>
       and Alexandre Duret-Lutz <adl@gnu.org>.
{%endcodeblock%}

With Automake installed and available, it is time to get the compilation script and the libzmq source code. Start by cloning the libzmq-ios-universal project from Github. Change directory to a suitable location

{%codeblock%}
clone https://github.com/samsoir/libzmq-ios-universal.git
{%endcodeblock%}

Once the clone operation is completed, `cd libzmq-ios-universal` so that the next step downloads into the project folder.

Now download and untar the latest stable build of libzmq from the [ZeroMQ download page](http://www.zeromq.org/area:download). At the time of writing this is version 3.2.2.

{%codeblock%}
wget http://download.zeromq.org/zeromq-3.2.2.tar.gz && tar xfz zeromq-3.2.2.tar.gz
{%endcodeblock%}

If everything is fine up until now, we are ready to begin compiling libzmq for iOS. To begin the process type;

{%codeblock%}
./compile_libzmq_ios_universal
{%endcodeblock%}

A successful compilation will result in the following output;

{%codeblock%}
Initializing build directory...
Initializing dependency directory...
Compiling libzmq for armv7...
Compiling libzmq for armv7s...
Compiling libzmq for i386...
Creating universal static library for armv7, armv7s and i386
Copying libzmq headers into universal library...
Tidying up...
Finished compiling libzmq as a static library for iOS.

Universal library can be found in build/universal;
  "lib" folder contains "libzmq.a" static library
  "include" folder contains headers

To use in your project follow linking instructions
available on the iOS zeromq page
http://www.zeromq.org/build:iphone
{%endcodeblock%}

Note that in the unfortunate event that something goes wrong during the compilation, the root cause may not be immediately apparent. However the last command executed places all of its output into a file called `build/build.log` relative to the current path. The cause of the issue should be apparent within that file.

Assuming everything is fine, it is now possible to add the new universal static library to Xcode.

Change the directory to your Xcode project, for example;

{%codeblock%}
cd ~/Projects/<project name>
{%endcodeblock%}

I like to keep my external libraries in their own directory. So I create a directory called "lib" within the project to keep them in.

{%codeblock%}
mkdir -p <project name>/lib && open <project name>
cp /<path>/<to>/<libzmq>/build/universal <project name>/lib/libzmq
{%endcodeblock%}

This will result in a new directory named `lib` created within the `MyZMQProject` code base directory. Within that folder with be a `libzmq` folder containing the static library and headers. Xcode will nest the directories like this to keep separate targets apart within a project. Finally the command will open a Finder window at `./MyZMQProject` allowing you to see the `lib` folder.

{% img /images/libzmq/lib.png Finder showing lib folder %}

Ensure your Xcode project is open within Xcode and then drag the `lib` folder from Finder into the correct (and corresponding location) within Xcode. A dialog will appear asking how you want to add the folder to the project.

{% img /images/libzmq/xcode_add_to_project.png Xcode add folder to project %}

Once you select "Finish" the `lib` folder will be added to your project.

Before you can start using the library you need to update your projects settings to link the library to your project at compile time, and include the headers in the search path.

Within the main target of your project, filter the _Build Settings_ to only show items including "Search Path". Then update _Header Search Paths_ section to include your newly created `lib` folder. Add `${SRCROOT}/<project name>/lib/**` to the list and hit return. The double-star means search recursively through sub folders.

{% img /images/libzmq/header_search_paths.png Header search paths in project settings %}

Xcode 4.5 automatically adds the correct path for libzmq to the _Library Search Paths_ entry. However if this hasn't been added for you, the same entry that was added for _Header Search Paths_ will work here.

At this point you are ready to start using libzmq in your code. To use the ØMQ C library within your code, import the `zmq.h` header in any location where you require it. Libzmq will now run  in the simulator or on a device without modification.

## Footnotes

<strong id="f1">1.</strong> As the name implies, armv7 instructions can be executed on armv7s architectures, but the reverse is not true. The armv7s architecture provides additional instructions that provide faster execution of code over armv7, so it is worth supporting it when possible.

<strong id="f2">2.</strong> Anyone that is put off by this, it is worth mentioning that there is an [Objective C](https://github.com/jeremy-w/objc-zmq) higher level wrapper in existence.

<strong id="f3">3.</strong> I decided not to support armv6 at this point. It may be added in in future if demand shows it is worthwhile. Or there is a pull request.
