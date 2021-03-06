---
layout: post
title: "CoreData - The Right Way, Part-I"
date: 2015-04-09 22:34:33 +0530
comments: true
categories:
- iOS
- OOP
- Core Data
---
{% blockquote %}
“Always code as if the guy who ends up maintaining your code will be a violent psychopath who knows where you live”
― John Woods
{% endblockquote %}

If you don't know what's CoreData you are in the wrong place. I suggest you go through [WWDC 2010](https://developer.apple.com/videos/wwdc/2010/) video
Mastering CoreData, before you proceed with the blog.

CoreData is easy to start off with but difficult to master, and the generated code that Apple provides when you create a new project with
CoreData doesn't help either. In the generated code project you will find methods like persistentStoreCoordinator, managedObjectContext, saveContext related to CoreData are being
implemented by the AppDelegate, which completely violates the [Single Responsibility Principle](http://blog.8thlight.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html).
Methods related to setting up CoreData, upgrading CoreData , creating ManagedObjectContext etc should be present in a different class. This kind of
project is great to spike out things, but you never write this in production code.

If you are not convinced with my argument, try to answer the following questions -

1. What would you do when you want to manage 2 different CoreData databases for 2 separate purposes ?
2. How could you TDD this kind of code when everything you need is inside AppDelegate  ?
3. Why is CoreData methods tied so tightly with something like AppDelegate ?
4. Why the hell do you have to type cast like the below every time you want to access the NSManagedObjectContext.
``` objc
 AppDelegate appDelegate = (AppDelegate*) [UIApplication sharedApplication].delegate
```

<!-- more -->

##Subclassing NSManagedObjectContext:

According to me, NSManagedObjectContext should be created and passed along to different functions when needed. Using NSManagedObjectContext with singleton pattern (As suggested by
code generated by Apple) reduces the power that CoreData provides. Context is there so that you can create several at a time, use separate contexts in separate threads, discard a
context without saving it not needed, or merge a 2 or 3 contexts.

So, extracting methods out into a NSManagedObjectContext subclass will make things look something like this:

``` objc
@implementation TTManagedObjectContext

+(TTManagedObjectContext*)managedObjectContextForManagedObjectModel:(NSString*)momName andSqliteFileName:(NSString*)filename{
    NSPersistentStoreCoordinator *coordinator = [self persistentStoreCoordinatorForManagedObjectModel:momName andFilename:filename];
    TTManagedObjectContext *context = [[TTManagedObjectContext alloc]init];
    [context setPersistentStoreCoordinator:coordinator];
    return context;
}


+ (NSManagedObjectModel *)managedObjectModelForName:(NSString*)momName {

    NSURL *modelURL = [[NSBundle bundleForClass:[self class]] URLForResource:momName withExtension:@"momd"];
    if (modelURL == nil){
        NSLog(@"Could not find file:%@.momd",momName);
        abort();
    }
    NSManagedObjectModel *managedObjectModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
    return managedObjectModel;
}

+ (NSPersistentStoreCoordinator *)persistentStoreCoordinatorForManagedObjectModel:(NSString*)mom andFilename:(NSString*)filename{

    NSPersistentStoreCoordinator *persistentStoreCoordinator = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:[self managedObjectModelForName:mom]];
    NSError *error = nil;
    if (filename != nil) {
        NSURL *storeURL = [[self applicationDocumentsDirectory] URLByAppendingPathComponent:filename];
        [persistentStoreCoordinator addPersistentStoreWithType:NSSQLiteStoreType configuration:nil URL:storeURL options:nil error:&error];
    }else{
        [persistentStoreCoordinator addPersistentStoreWithType:NSInMemoryStoreType configuration:nil URL:nil options:nil error:&error];
    }

    if (error) {
        NSLog(@"ERROR OCCURED:%@",error);
        abort();
    }

    return persistentStoreCoordinator;
}

+ (NSURL *)applicationDocumentsDirectory {
    return [[[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory inDomains:NSUserDomainMask] lastObject];
}

@end

```

Now you can write unit tests such as the following, plus your code is now modular and ready for re-use:

``` objc
- (void)setUp {
    [super setUp];
    logContext = [TTManagedObjectContext managedObjectContextForManagedObjectModel:@"Log" andSqliteFileName:nil];
    charactersContext = [TTManagedObjectContext managedObjectContextForManagedObjectModel:@"Characters" andSqliteFileName:nil];
// Note: SqliteFilename is nil which according to our new implementation makes in-memory db, so the unit tests now run faster
// and you do not need to worry about clearing the db after running the tests.
}


- (void)testSaveInCharacters {

    NSManagedObject *batman = [NSEntityDescription insertNewObjectForEntityForName:@"SuperHero" inManagedObjectContext:charactersContext];
    [batman setValue:@"Batman" forKey:@"name"];
    [batman setValue:@3 forKey:@"power"];
    [batman setValue:@5 forKey:@"brains"];
    NSManagedObject *superman = [NSEntityDescription insertNewObjectForEntityForName:@"SuperHero" inManagedObjectContext:charactersContext];

    [superman setValue:@"Superman" forKey:@"name"];
    [superman setValue:@5 forKey:@"power"];
    [superman setValue:@1 forKey:@"brains"];

    NSManagedObject  *log = [NSEntityDescription insertNewObjectForEntityForName:@"Log" inManagedObjectContext:logContext];
    [log setValue:@"Batman wins !!!" forKey:@"message"];
    [log setValue:@1 forKey:@"priority"];

    [charactersContext save:nil];
    [logContext save:nil];

    NSFetchRequest *fetchRequestCharacter = [[NSFetchRequest alloc]initWithEntityName:@"SuperHero"];
    NSFetchRequest *fetchRequestLog = [[NSFetchRequest alloc]initWithEntityName:@"Log"];

    NSArray *superHeros =  [charactersContext executeFetchRequest:fetchRequestCharacter error:nil];
    NSArray *logs = [logContext executeFetchRequest:fetchRequestLog error:nil];

    XCTAssertEqual(superHeros.count,2);
    XCTAssertEqual(logs.count, 1);
    XCTAssertEqual([logs[0] valueForKey:@"message"],@"Batman wins !!!");
}
```

This has really modularised things for us, in the next part we will look how can we take this even further.
You can find the full code on my github repo. [CoreDataExample](https://github.com/tapthaker/CoreDataExample)

