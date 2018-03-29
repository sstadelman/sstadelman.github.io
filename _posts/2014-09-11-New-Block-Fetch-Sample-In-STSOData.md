---
layout: post
title: "Update to STSOData: block fetch example, and forget about Logon finishing"
quote: Handling for kLogonFinished, and new fetch sample with completion block
image: /media/cover-images/at&t-day.jpg

---

#Update to STSOData:  handling for kLogonFinished, and new fetch sample with completion block

I've added a new request example to **DataController+FetchRequestsSample.h/m**, that passes a completion block returning the NSArray of entities.

You can copy/paste the implementation into your own **DataController+FetchRequests** category, and modify with your resource paths.

I've also eliminated the need to ever be aware of when the logon procedure is completed, when you are sending requests!

##Using the Database as the model, directly
My thinking in adding a completion block to the `fetch:` request was that for requests against the local database--at least, those with smaller result sets--there shouldn't be a particularly substantive difference in read time between accessing a property on the Model singleton, and just reading directly from the database.  I haven't done any benchmarking, but I would suspect that for requests with complex filters on large collections, reading from the db might win out.

With this approach, you could skip over the KVO implementation in your view controllers, and just fire off an asynchronous `fetch` when the view controller appears.  You might lose a few microseconds on populating the UI, but for result sets which aren't going to be shared by multiple screens, the tradeoff would be minimal.  Plus, it makes writing `XCTest`'s much more convenient.

	- (void)viewWillAppear:(BOOL)animated  {

	    [[DataController shared] fetchTravelAgenciesSampleWithCompletion:^(NSArray *entities) {
	        if (![self.objects isEqualToArray:entities]) {
	            self.objects = [NSMutableArray arrayWithArray:entities];
	            [self.tableView reloadData];
	        }
	    }];
	}

##Handling for `kLogonFinished`
During the implementation, I found an issue where the main view controller appears before the `logonDidFinish:` delegate is called on the **LogonHander**.  So, the fetch request was invoking `scheduleRequestForResource:` on **DataController.m**, which was calling `[self.store openStoreWithCompletion:^(BOOL success){}]`.  But, the store isn't initialized until `-setupStore` is called on DataController, which is listening for the `kLogonFinished` notification in `-loadWorkingMode`.

	-(void)loadWorkingMode
	{
	    [[NSNotificationCenter defaultCenter] removeObserver:self];
	    
	    if ([LogonHandler shared].logonManager.logonState.isUserRegistered &&
	        [LogonHandler shared].logonManager.logonState.isSecureStoreOpen) {
	        [self setupStore];
	    } else {
	        [[NSNotificationCenter defaultCenter] addObserverForName:kLogonFinished object:nil queue:nil usingBlock:^(NSNotification *note) {
	            
	            [self setupStore];
	        }];
	    }
	}

`self.store` is nil, and the fetch request needs to be held, until the store is setup.  In the example with KVO, this problem was handled in the view controller, because the `fetch` wasn't being sent until the view controller observed the `kLogonFinshed` notification.  But that's kind of a bothersome workaround, so let's go with the following:

In **OfflineStore.m** and **OnlineStore.m**'s `openStoreWithCompletion:`, add a check to see if `logonState` is registered, and secure store is open.  If both are true, then `logonDidFinish:` was called sometime in the past, and `kLogonFinished` notification already posted.  But, if they're false, then the logon procedure is still running.  And it's not good enough to just listen for `kLogonFinished`, since DataController's `-loadWorkingMode` is also listening, and doing the store setup.

So, I added three new notifications:  `kOnlineStoreConfigured`, `kOfflineStoreConfigured`, and `kStoreConfigured`, that are posted at the end of DataController `-setupStore`.  In the `openStoreWithCompletion:`, the online and offline stores listen for their configuration completion, before attempting to open.

	[self setOfflineStoreDelegate:self];
    [self setRequestErrorDelegate:self];
    
    __block void (^openStore)(void) = ^void() {
        
        NSError *error;
        
        [self openStoreWithOptions:[LogonHandler shared].options error:&error];
        
        if (error) {
            NSLog(@"error = %@", error);
        }
    };
    
    if ([LogonHandler shared].logonManager.logonState.isUserRegistered &&
        [LogonHandler shared].logonManager.logonState.isSecureStoreOpen) {
        
        openStore();
    
    } else {
    
        [[NSNotificationCenter defaultCenter] addObserverForName:kOfflineStoreConfigured object:nil queue:nil usingBlock:^(NSNotification *note) {
            NSLog(@"%s", __PRETTY_FUNCTION__);
            
            openStore();
        }];
    }

Then, the `kStoreConfigured` notification is posted, and handled within DataController's `-scheduleRequestForResource:withMode:withEntity:withCompletion`.  By using the block variant of the NSNotification observer, and creating `__block void ^executeRequestWhileOpeningStore` as a block, it allows for the initial request to be 'paused' until the store is initialized and configured, without creating a queue.  

    __block void (^executeRequestWhileOpeningStore)(void) = ^void() {
    
        [self.store openStoreWithCompletion:^(BOOL success) {
                        
            [self scheduleRequest:myRequest completionHandler:^(NSArray *entities, id<SODataRequestExecution> requestExecution, NSError *error) {
                                
                completion(entities, requestExecution, error);
            }];
        }];
    };
    
    if (self.store != nil) {
    
        executeRequestWhileOpeningStore();
        
    } else {
        [[NSNotificationCenter defaultCenter] addObserverForName:kStoreConfigured object:nil queue:nil usingBlock:^(NSNotification *note) {
            
            executeRequestWhileOpeningStore();

        }];
    }

##The final result
The net of this modification--all in the STSOData code, you don't need to do anything except follow-along--is that you can now send a request to DataController `-scheduleRequestForResource:withMode:withEntity:withCompletion`, without ***any*** handling for whether the logon procedure is finished, or the store is open.  When all that stuff is done, your request will be completed and returned!

