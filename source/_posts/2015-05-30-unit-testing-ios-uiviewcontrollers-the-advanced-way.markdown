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

The idea is to separate the business logic from the UIViewController to something like **BusinessLogicController**.
The BusinessLogicController should have no reference of UIKit, thus making the code modular and platform independent.

The UIViewController implements the **Controllable** protocol which basically has a set of methods -

1. **render** : Called to paint something on the screen
2. **getValue**: Called to get the value for a particular key
3. **showAlert**: Optional, called to show an Alert
4. **goToPage**: Optional, called to navigate to another page
5. **property eventable**: To dispatchEvents to the BusinessLogicController

The first 4 methods are called by the BusinessLogicController as and when required. The UIViewController
calls the dispatchEvent when a events like button clicked occur to let the BusinessLogicController know
about a particular event.

The BusinessLogicController implements **Eventable** which has just one method namely **dispatchEvent** and has a reference of UIViewController in form of
Controllable.

Lets take the example of Login. Lets say you have to code the following scenario -

- When the username & password are correctly entered take the user to HomeVC.
- If the credentials entered are incorrect display the message *"Incorrect username or password"*.
- Also, if the user exceeds 5 attempts display an alert saying *"Maximum number of retries exceeded"*.

<img align="center" src="{{root_url}}/images/diagrams/UnitTestingVC-advanced.png" />

With the above criteria the LoginBusinessLogicController would look something like this -

{% include_code [LoginBusinessLogicController] LoginBusinessLogicController.swift %}

Notice the it conforms to the Eventable protocol, which declares the method **dispatchEvent**.

To test such BusinessLogicControllers, you can create a StubbedControllable and use that to send events.
Lets look at how we could could implement tests for this -

{% include_code [LoginBusinessLogicControllerTests] LoginBusinessLogicControllerTests.swift %}

Note that neither the LoginBusinessLogicController nor its test is importing the UIKit module. The code written here
is independent of the iOS UIKit Framework which means the tests can run as a separately from the Simulator, faster in-parallel
if need be. As a bonus you can create a new iPad-App or Mac-App with the same BusinessLogicControllers wiring them with new ViewControllers.

I am pretty sure that by now you already have an idea of the LoginViewController's implementation

{% include_code [LoginViewController] LoginViewController.swift %}