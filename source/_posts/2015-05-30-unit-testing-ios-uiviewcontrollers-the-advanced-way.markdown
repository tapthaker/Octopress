---
layout: post
title: "Unit Testing iOS UIViewControllers - The advanced way"
date: 2015-05-30 18:44:43 +0530
comments: true
categories:
- iOS
- OOP
- UIViewControllers
- Unit Testing
---
{% blockquote %}
“Am I suggesting 100% test coverage? No, I’m not suggesting it. I’m demanding it. Every single line of code that you write should be tested. Period.”
― Uncle Bob
{% endblockquote %}

I have already [written a post wrt this topic](http://tapthaker.github.io/blog/2015/03/28/unit-testing-ioss-uiviewcontrollers/), but after discussing with several colleagues
I have realized that the approach I was talking about might not be scalable. The approach discussed there relies on several hacks which though **cool** can cause some unexpected behaviour.
We are relying on the internal implementation of UIViewControllers which might change anytime when devs at Apple feel like. Plus that approach makes us comfortable in writing core business logic in the
UIViewControllers instead of creating a ViewModels for them.

The approach I am going to describe here does not necessarily apply only to UIViewControllers. It can in fact be applied to any MVC pattern implementation. The problem with the current iOS implementation is that
the UIViewController is very tightly coupled with the UIView. So if you want to test a your logic when a particular UIButton is tapped you cannot do that without inflating the whole view hierarchy.

<!-- more -->