---
layout: post
title: "Unit testing iOS's UIViewControllers"
date: 2015-03-28 20:46:47 +0530
comments: true
sharing: true
categories:
 - unit testing
 - tdd
 - UIViewController
 - ios
---
{% blockquote %}
“All code is guilty, until proven innocent.” – Anonymous
{% endblockquote %}

Unit tests are one of the corner stones of software development these days. But when it comes to writing unit tests for user interaction
like UIViewControllers we hardly write any mainly because we assume that we cannot simulate user interactions in unit tests. This post tries to
remove this misconception and shows how easily you could write unit tests for UIViewControllers.
For the rest of this post I will refer the UIViewController as view controller.

Some of the things to note before you start things:

* The view of the view controller needs to be loaded and wired properly before you simulate any interactions.
* The interactions need to happen with the view elements, if on tapping the login button a selector onLoginClicked: is called then this does not mean
 that you directly call that method from your unit test. You need to simulate that click instead.
* For mocking & stubbing objects in objective-C I highly recommend the framework [OCMOCK](http://ocmock.org/ "Link to OCMock's official page").
  You can use [cocoapods](http://cocoapods.org/) to install OCMock.

##Example

Lets say we have a login view controller, which could have some simple scenarios like the following:

* If username & password are correct then push a HomeVC.
* If the username or password is incorrect then show an alert.

Lets try writing a unit test for the first scenario.

``` objc
- (void)testValidLogin{

    // Setting up things
    UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
    LoginVC *loginVC = [storyboard instantiateViewControllerWithIdentifier:@"ViewController"];
    UINavigationController *navigationController = [[UINavigationController alloc]initWithRootViewController:loginVC];
    id navControllerPartialMock = OCMPartialMock(navigationController);
    UIView *view = loginVC.view; // Wires up the view and other outlets and then calls the viewDidLoad method.


    // Setting up expections
    OCMExpect([navControllerPartialMock pushViewController:[OCMArg any] animated:YES]);


    // Run code under test
    loginVC.textFieldUsername.text = @"brucewayne";
    loginVC.textfieldPassword.text = @"i am batman";
    [loginVC.buttonLogin sendActionsForControlEvents:UIControlEventTouchUpInside];

    // Verifying the expectations
    OCMVerifyAll(navControllerPartialMock);
}
```

The test is pretty straight forward. One of the things to note is that accessing **loginVC.view** is necessary.
The view property of the view controller is loaded lazily when the view is accessed for the first time.
It is this time that iOS reads the XML present in the storyboard / xib and adds the views accordingly.
Without this the IBOutlets and their parent view will be nil.

Lets now see the implementation of our loginButtonTapped method to understand how to test the second scenario:

``` objc
- (IBAction)loginButtonTapped:(id)sender {

    if ([self.textFieldUsername.text isEqualToString:@"brucewayne"] && [self.textfieldPassword.text isEqualToString:@"i am batman"] ) {
        HomeVC *homeViewController = [[HomeVC alloc]init];
        [self.navigationController pushViewController:homeViewController animated:YES];
    }else {
        [[[UIAlertView alloc]initWithTitle:@"Incorrect" message:@"Incorrect username or password" delegate:nil cancelButtonTitle:@"Ok" otherButtonTitles: nil] show];
    }
}
```

A test for this could be to expect a UIAlertView on the screen when you enter incorrect username or password. The problem here is that
you don't have the access to UIAlertView's object to stub it. Of course you could do dependency injection here, but it doesn't make much sense mainly because the view controllers in
iOS aren't initialized that way. Luckily ObjC with its message passing can do some magic at runtime, enter the world of
**method swizzling**. Method swizzling is mainly about replacing a method's implementation with some other implementation. There is a [great post on NSHipster](http://nshipster.com/method-swizzling/)
about this. I will be using a 3rd party lib to swizzle methods called [Swizzlean](https://github.com/rbaumbach/Swizzlean "Github page for Swizzlean"),
its not necessary but it makes things simpler to work with. Lets now look at the test:

``` objc
- (void)testInvalidLogin{

    // Setting up things
    UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
    LoginVC *loginVC = [storyboard instantiateViewControllerWithIdentifier:@"ViewController"];
    UIView *view = loginVC.view;

    // Method swizzling
    __block BOOL alertViewShown = NO;
    Swizzlean *swizzle = [[Swizzlean alloc] initWithClassToSwizzle:[UIAlertView class]];
    [swizzle swizzleInstanceMethod:@selector(show) withReplacementImplementation:^() {
        alertViewShown = YES;
    }];

    // Run code under test
    loginVC.textFieldUsername.text = @"brucewayne";
    loginVC.textfieldPassword.text = @"Why so serious?";
    [loginVC.buttonLogin sendActionsForControlEvents:UIControlEventTouchUpInside];

    // Asserting
    XCTAssertTrue(alertViewShown,@"Show show Alert View when the invalid login");
    [swizzle resetSwizzledInstanceMethod];

}
```

The resetSwizzledInstanceMethod is also important or it might break some other tests that you have written.Method swizzling is a
powerful technique when you need it, but should be used sparingly.

Unit testing is often neglected but it is, in fact, the most important level of testing, just be careful not
to over engineer things.