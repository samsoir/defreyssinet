---
layout: post
title: "Refactoring the PhoneGap iOS DatePicker"
date: 2012-03-31 10:00
comments: false
published: true
categories: [ios, phonegap, code]
---

![iOS Date Picker](/images/datepicker.png)

Recently my team were using [_PhoneGap_](http://phonegap.com/), the HTML5 middleware that enables web developers to write mobile apps once using technologies they know; HTML5, CSS and Javascript. The completed PhoneGap/HTML5 application is then compiled into binaries for each of the [supported mobile platforms](http://phonegap.com/about/features). The idea is simple, developers write code once and deploy many times. PhoneGap can certainly be used for creating simple applications, but stray from the [core API](http://docs.phonegap.com/en/1.4.1/index.html) and you quickly find yourself either writing a native extension; or more likely finding a plugin that does the task for you.
<!-- more -->

As part of a research project for my employer we were writing a mobile application using the PhoneGap platform. The application in question was a relatively simple concept, however it quickly became apparent we would have to go outside of the core API. 

PhoneGap does not provide a standard way to create a date picker, instead delegating the vast majority of interface responsibilities to the application and/or framework chosen to author the application. In our case we were using [JQuery Mobile](http://jquerymobile.com/) for the application user interface. There are others available, [Sencha Touch](http://www.sencha.com/products/touch/) being another popular one from the creators of [ExtJS](http://www.sencha.com/products/extjs/).

Date input is a potentially complex operation on a mobile device. There are many components to a full date and making the user type it in is hardly the optimum interface. Of course there are many web forms that expect the user to do just that. But in the mobile context, there is a chance the operator will only be using one hand to interact with the device, so making them use a complex input control, such as a keyboard, would instantly have a negative effect on their experience. 

Presenting the standard [JQuery Mobile Datepicker](http://jquerymobile.com/demos/1.0a4.1/experiments/ui-datepicker/) or similar will equally frustrate the user. The JQuery Mobile Datepicker component is awful, but hardly the most efficient design. Also, PhoneGap gives the user the impression the app they are using is a native app, so they will most likely expect a native control in this situation. We could use an iOS DatePicker implementation in Javascript such as [Cubiq's Spinning Wheel Plugin](http://cubiq.org/spinning-wheel-on-webkit-for-iphone-ipod-touch) to present a picker in the PhoneGap. But this ensures the picker will use an iOS interface even when used on an Android, Windows Mobile or one of the two remaining Blackberry devices. Another great way to ruin the user experience is to start presenting unfamiliar controls.

Luckily modern mobile operating systems have solved this problem by supplying a native Datepicker control. Using the native Datepicker control on each platform requires a native [PhoneGap extension](http://wiki.phonegap.com/w/page/36752779/PhoneGap%20Plugins). Naturally the first thing to do is look for an existing solution to this problem within the PhoneGap ecosystem. A quick search revealed that Datepicker extensions do exist for PhoneGap on [iOS](https://github.com/phonegap/phonegap-plugins/tree/master/iPhone/DatePicker) and [Android](https://github.com/phonegap/phonegap-plugins/tree/master/Android/DatePicker). After downloading the extension and plugging it into our PhoneGap application, the extension worked&hellip; but unfortunately it would only work the one time. 

The application would display the `UIDatePicker` as expected, allowing the user to select a date and dismiss the action sheet. However when the date picker was summoned subsequently, it would not respond to the user input and the date selected was not returned to the application. It was time to look at the [source code](https://github.com/phonegap/phonegap-plugins/blob/59f3cf98c40ade9268c0546b7537ad769a17088e/iPhone/DatePicker/DatePicker.m) to see if there was anything obvious going wrong. The source quickly revealed some issues that required urgent attention.

## Memory management

First off there are some basic Objective C memory management violations within the DatePicker extension. The `DatePicker.h` header file defines `@property (nonatomic, retain) UIDatePicker *datePicker;` accessor method. Using this accessor for assignment will increment the reference count of the `UIDatePicker` object passed to it by one. If we examine the following code, we can see how the accessor is used within the DatePicker implementation.

{%codeblock lang:objc DatePicker.m https://github.com/phonegap/phonegap-plugins/blob/59f3cf98c40ade9268c0546b7537ad769a17088e/iPhone/DatePicker/DatePicker.m#L34 %}
self.datePicker = [[UIDatePicker alloc] initWithFrame:pickerFrame];
{%endcodeblock%}

The `alloc` method will allocate memory for the `UIDatePicker` object and increment the retain count by one. Assigning the resulting object to the `datePicker` instance variable via the accessor will increment the retain count a second time. Later in the implementation the `datePicker` property is released, decreasing the retain count. It is now `1` for those still counting. An object with a retain count greater that zero will be retained by the autorelease pool.

Memory management in Objective C is a vital concept to understand if you are going to write code any code. Even if you intend to use the new [Automatic Reference Counting](http://clang.llvm.org/docs/AutomaticReferenceCounting.html) features provided by Clang, I believe it's imperative to understand the fundamentals of memory management. I am not using ARC at the moment and do not intend to any time soon, the reasons behind this decision are probably worthy of another blog post at some other time. However there are three simple rules within the Objective C language for managing memory that, when followed, will see you good in the vast majority of situations;

{%blockquote%}
When creating a new object using either the;

 1. [alloc]
 2. [new]
 3. [copy] or derivatives, such as [mutableCopy]
 
methods, then you are responsible for releasing that object when you are finished with it.
{%endblockquote%}

The result of this oversight within the PhoneGap DatePicker extension is that the `UIDatePicker` object created in the code block shown above is never released, creating a memory leak. If there was only ever one instance of the `UIDatePicker` object then this would not be so much of an issue, although all memory leaks are bad. One leaking date picker is not going to consume vast amounts of memory. But this situation gets worse.

## Further inspection

{%codeblock lang:objc DatePicker.m https://github.com/phonegap/phonegap-plugins/blob/59f3cf98c40ade9268c0546b7537ad769a17088e/iPhone/DatePicker/DatePicker.m#L19-68 %}
datePickerSheet = [[UIActionSheet alloc] initWithTitle:nil delegate:nil cancelButtonTitle:nil destructiveButtonTitle:nil otherButtonTitles:nil];

[datePickerSheet setActionSheetStyle:UIActionSheetStyleBlackTranslucent];

CGRect pickerFrame = CGRectMake(0, 40, 0, 0);

self.datePicker = [[UIDatePicker alloc] initWithFrame:pickerFrame];
bool allowOldDates = ([[options objectForKey:@"allowOldDates"] intValue] == 1)?YES:NO;
if (!allowOldDates) {
	self.datePicker.minimumDate = [NSDate date];
}
if ([mode isEqualToString:@"date"]) {
	self.datePicker.datePickerMode = UIDatePickerModeDate;

	NSDateFormatter *dateFormatter = [[NSDateFormatter alloc] init];
	[dateFormatter setTimeZone:[NSTimeZone defaultTimeZone]];
	[dateFormatter setDateFormat:@"MM/dd/yyyy"];
	NSDate *date = [dateFormatter dateFromString:dateString];
	[dateFormatter release];
	self.datePicker.date = date;
}
else if ([mode isEqualToString:@"time"])
	self.datePicker.datePickerMode = UIDatePickerModeTime;

[datePickerSheet addSubview:self.datePicker];
{%endcodeblock%}

The code block above presents an abstract from the `DatePicker show:withDict:` method. This is the Objective C method the PhoneGap Javascript plugin is bound to. Each time a HTML control is selected that invokes the `show:withDict:` method, the code above is executed. 

To provide some context, PhoneGap plugins in Objective C are lazily instantiated, thus loaded into memory only when they are first called. This is quite a nice feature. However PhoneGap relies on the extension code playing nicely and tiding up after itself when it is finished working, or if the operating system warns of low memory. PhoneGap calls the `onMemoryWarning` method within the extension if the operating system is running low on memory, allowing the plugin to release objects superfluous to requirements at that moment. No such luck here. It wouldn't be so bad if this extension was not leaking memory.

Returning to the code shown above. The `DatePicker show:withDict:` method is being invoked by PhoneGap when the plugins Javascript counterpart is executed. Each time this method is executed a new `UIDatePicker` is created and assigned to the plugin and action sheet. Because the previous `UIDatePicker` is not released and also not removed from the Action Sheet view, the new instance is placed over the existing instance. The new instance _should_ always be placed on top, but who knows what happens when there are several `UIDatePicker` controls layered up on top of each other. This begins to explain the strange behaviour experienced earlier where repeated use of the _DatePicker_ extension resulted in it not responding or returning the correct date. 

## Delegates and Single Responsibility Principle

One other implementation detail was also causing concern. The `UIActionSheet` class has a [delegate protocol](http://developer.apple.com/library/ios/#documentation/uikit/reference/UIModalViewDelegate_Protocol/UIActionSheetDelegate/UIActionSheetDelegate.html). The Cocoa frameworks use the delegate pattern consistently across many user interface controls. Unfortunately the _DatePicker_ extension does not use the provided delegates.

{% codeblock lang:objc DatePicker.m https://github.com/phonegap/phonegap-plugins/blob/59f3cf98c40ade9268c0546b7537ad769a17088e/iPhone/DatePicker/DatePicker.m#L70-77 %}
- (void) dismissActionSheet:(id)sender {
	[datePickerSheet dismissWithClickedButtonIndex:0 animated:YES];
	[datePickerSheet release];
	[datePicker release];
	NSString* jsCallback = [NSString stringWithFormat:@"window.plugins.datePicker._dateSelected(\"%i\");", (int)[self.datePicker.date timeIntervalSince1970]];
	[super writeJavascript:jsCallback];
	isVisible = NO;
}
{% endcodeblock %}

The `dismissActionSheet:` method is invoked by a `UISegmentedControl` button defined earlier in the class. When the user selects the _Close_ button, this method is invoked. But this method is performing a lot of actions, most are outside the scope of the method name. The current responsibilities of this method are;

1. Remove the `UIActionSheet` from the interface
2. Free the `UIActionSheet` from memory
3. Free the `UIDatePicker` from memory (although we saw earlier this won't happen)
4. Return the date picked to the PhoneGap API
5. Update the state of the DatePicker extension

That is a lot of responsibility for a simple UI action. A method named `dismissActionSheet:` should do just that, dismiss the action sheet. Therefore to maintain the single responsibility principle this method should only perform the action it prescribes.

{% codeblock lang:objc %}
- (void)dismissActionSheet:(id)sender {
	[self.datePickerSheet dismissWithClickedButtonIndex:0 animated:YES];
}
{% endcodeblock %}

The new implementation of `dismissActionSheet:` solely performs the action requested of it, namely removing the `UIActionSheet` from the interface. But in the process a lot of the other functionality previously performed has been lost. Returning to the list of responsibilities above, there are three tasks that are important to the functionality of this extension. (Memory management of UI elements should not be handled here)

1. __Return the date picked to the PhoneGap API__
2. __Update the state of the DatePicker extension__
3. Remove the `UIActionSheet` from the interface

Having already removed the `UIActionSheet` from the interface, only the first two tasks remain. 

The first task is to return the date picked to the PhoneGap API. Because a value is being transmitted from the one platform (Objective C) to another (JS), there is potential for things to go wrong. It is probably best to send the value to the PhoneGap API before removing the `UIActionSheet`. The `UIActionSheetDelegate` class provides a method `actionSheet:willDismissWithButtonIndex:` that is invoked prior to the `UIActionSheet` being dismissed.

{% codeblock lang:objc %}
- (void)actionSheet:(UIActionSheet *)actionSheet willDismissWithButtonIndex:(NSInteger)buttonIndex
{
	NSString* jsCallback = [NSString stringWithFormat:@"window.plugins.datePicker._dateSelected(\"%i\");", (int)[self.datePicker.date timeIntervalSince1970]];
	[super writeJavascript:jsCallback];
}
{% endcodeblock %}

The date selected by the user is now passed to the PhoneGap API as the `UIActionSheet` is being dismissed by the system. If there are any problems passing the value back to PhoneGap, an alert can be provided before the interface is dismissed. (Ideally, there would be a `UIActionSheetDelegate` method for cancelling the dismiss; `actionSheet:shouldDismissWithButtonIndex:` for example. Unfortunately no such luck here)

The final task is to update the state of the extension. As the extension will hang around after the user has dismissed it, the state will be important for managing memory and components so it is worth keeping. Again the `UIActionSheetDelegate` protocol provides the perfect method for updating the state, `actionSheet:didDismissWithButtonIndex:`.

{% codeblock lang:objc %}
- (void)actionSheet:(UIActionSheet *)actionSheet didDismissWithButtonIndex:(NSInteger)buttonIndex
{
	isVisible = NO;
}
{% endcodeblock %}

All of the original functionality tied to the _Close_ button has now been refactored into new methods that use the standard Objective C and Cocoa patterns. Additionally, the single responsibility principle has been maintained.

## The refactored DatePicker extension and PhoneGap

There are many more improvements that can be made to this DatePicker extension for PhoneGap. It is easy to critique others code, but it is far more valuable to give back. As we had to fix the issues detailed here and a number of others, we pushed the changes back to GitHub and requested they were [pulled](https://github.com/phonegap/phonegap-plugins/pull/363) into the project. To the credit of the PhoneGap plugins project, they [merged the changes in](https://github.com/phonegap/phonegap-plugins/commit/d07bc00fc3ce05df85dafd25bd1da8108e521fa9) promptly. 

There are no doubt other improvements that can be made to the DatePicker extension, however those improvements will not be completed by myself or the team. After attempting to build our application in PhoneGap, we ultimately abandoned the project and reverted to native development on iOS and Android. 

Unfortunately PhoneGap promises a lot, but does not always deliver on these promises. Additionally many of the _official_ extensions are implemented to a similar standard as the _DatePicker_ extension examined here. The questionable quality extends beyond iOS to Android as well, as different but equally crippling issues were found with the Android _DatePicker_ extension. I am sure that PhoneGap has a place in the mobile development space, but beyond basic brochure applications I am not entirely sure what it is.

## Update

After this was initially posted, people have pointed out that HTML5 does provide a [datetime input type](http://www.w3.org/TR/html5/states-of-the-type-attribute.html#date-and-time-state-type-datetime). This would obviously provide a better way to perform all of the above within the browser context. The support is still rather patchy, so keep an eye on PPK's excellent work [testing HTML5 support](http://www.quirksmode.org/html5/inputs_mobile.html) across all mobile devices. As with all things in the mobile web, support for `<input type="datetime"></input>` can only improve with time.