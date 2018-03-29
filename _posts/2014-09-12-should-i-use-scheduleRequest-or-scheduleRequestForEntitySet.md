---
layout: post
title: "Should I use -scheduleRequest:, or -scheduleReadEntitySet:?"
quote: Which `SODataStoreAsync` method should I use, and how can I use blocks to simplify the code?
image: /media/cover-images/at&t-day.jpg

---


#Should I use `scheduleRequest:` or a higher-level API like `scheduleReadEntitySet:` in SODataStoreAsync?

We had a great discussion internally about whether to use `scheduleRequest:` to send requests on an `<SODataStoreAsync>`, or one of the higher-level API's, like `scheduleReadEntitySet:`, or `scheduleReadEntityWithResourcePath:`.  My recommendation generally is to use `scheduleRequest:`, since it allows you to have a single request method abstraction for READ, CREATE, UPDATE, DELETE, PATCH.  

The other important benefit of `scheduleRequest:` is that it takes a `SODataRequestParam` parameter that is unique, so we can track it through the delegate flow easily.

My [example code](https://mylink.com/broken) uses this approach in **DataController.m**, such that:

  - Developer invokes `scheduleRequestForResource:WithMode:withEntity:withCompletion:` for all requests,
  - which invokes `scheduleRequest:completionHandler:` internally,
  - which invokes `scheduleRequest:delegate` internally.

You don't necessarily need this much abstraction, but I included logic in the middle layer for ensuring the store is open, and that logon is complete, so there's a bit more going on under the covers.  That's one reason why using a single interface is convenient.  

But, the primary reason for this approach is that it is *Collection-agnostic*.  That means that there is no Collection-specific logic in the request implementation.  

Let me show how that is important.

##Handling specific responses:  Delegate callbacks
There are a number of ways that you can handle unique asynchronous responses with delegate callbacks:  create custom tags, stringify the pointer name, inspect the `response.request.customTag` the `request.options[value]`, etc.  

If you opt to program directly against the delegate callbacks, you need to implement Collection-specific logic in your response delegate, so that your model is correctly updated with the response.  For example, let's say you have two tabs that show *Accounts*, and *Opportunities*.  You schedule two requests on your `SODataOfflineStore`, and 20ms later, the `requestDidFinish:` callback is fired with a `<SODataRequestExecution>` response.  What do you do with the response:  how do you distinguish between the result sets?

###The wrong way
The worst approach, in my mind, is to solve this problem by replicating network functionality throughout the app, in order to reduce the number of response types that arrive at a response delegate callback:  implementing multiple `<SODataRequestDelegate>`'s:  one for each view controller. 

With only one Collection type per view controller, you can implement the request delegate in your view controller, and you know exactly what the response content will be.  After calling [`myStore scheduleRequestForResource:resource delegate:self]`, the response comes back to `-requestServerResponse:`, and you refresh the UI with the result.  Simple, but doesn't scale across multiple view controllers.  

If you have two Collection types per view controller, you need to handle the two in the request delegate.  The lowest-level approach for mapping the request type to the response object would be to keep track of the uniqueId's of the `<SODataRequestExecution>` object, and maintaining them as properties on the view controller. *(Don't do this)*

In **MyViewController.m**:

{% gist c4cf362f0d63710fc14a %}

Or, you could add custom tags to the `<SODataRequestExecution>`.  This at least has the virtue of keeping the information about the request within the request itself (rather than maintaining it in the view controller).  There's a race condition between the setter and the delegate response, but it's unlikely that you'd lose even if reading from the local database in OfflineMode.  I'm not a fan of this approach, but it's probably fine over a network for online requests.  

In **MyViewController.m**:

{% gist 0b287e50a44cfdeee81b %}

Note:  the `-scheduleReadEntitySet:` method includes an `options:(NSDictionary *)options` parameter.  When using the `SODataOnlineStore`, it is possible to create a tag as a dictionary key/value, and pass the dictionary to the `options` parameter.  Then, you can retrieve the tag in `-requestServerResponse:`:
	
{% gist 829461bd69a22b6bf809 %}

However, as of SDK 3.0 SP5PL01, this method will not work for the SODataOfflineStore, as the `options` dictionary is not copied into the `request` object. 

##NSNotifications...
One way to keep the Collection-specific code out of the request delegate, is to put it in the application itself.  This is not a terrible way to accomplish things in some cases, though I'll show how it is not yet a complete solution.  

In this approach, your view controller listens for relevant notifications, containing either:

1.  A general notification about the Collection or resource (e.g.:  `@"TravelagencyCollection"`)
2.  A notification about a specific request id (e.g.:  `@"requestFinished.request00001"`)

###Part-way there
This approach allows you to have a single implementation of the `<SODataRequestDelegate>` that is shared by all collections.  When a response comes in, the job of the callback implementation is to determine which notification identifier should be emitted, then set the generally-agreed-upon payload as the notification's object.

In **MyViewController.m**:

{% gist 650f41e21d8cc38a7b8a %}

In **MyRequestController.m**:

{% gist ad8eaab56fa0c30c47a1 %}

This approach is definitely the most flexible thus far:  it allows you to maintain some central logic for the networking, while allowing multiple view controllers to observe updates to the model--particularly useful for READ operations.  You can also start to centralize the parsing of the `requestExecution` response object.

In **MyRequestController.m**:

{% gist 21b7ebe34379c95bb626 %}

> My caution against this approach is that NSNotifications can get surprisingly unwieldy in complex applications, which is really frustrating to debug.  You can get similar dynamic UI updates from a decoupled model, with a much more robust and debuggable framework, by using KVO and a central model. I would strongly recommend this alternative, rather than maintaining model properties on each view controller, and updating them directly via NSNotifications.

##Give Blocks a try!
I can't urge you strongly enough to give blocks a try, if you have not yet adopted them in your standard programming model.  

The most important principle of using blocks, is that you can maintain your memory context in-line, for asynchronous operations.  So, if we can construct our request in such a way that the collection type and/or uniqueId is implicit, we can eliminate Collection-specific code from the network stack, *and* handle specific request responses in-context.

The method is to combine the two approachs we walked through above:  use implicit tags for the requests, and NSNotifications, plus a new block-based wrapper interface to the `scheduleRequest:` interfaces.  

###The modern (preferred) approach

Creating a block wrapper around the delegate callbacks allows you write methods like this:

{% gist fec22c714dae93791b77 %}

The beauty of an asychronous method with a completion block, is that you can still program against a local context in-line, instead of maintaining state at the parent level with properties and instance variables.  

Imagine the complexity of calling the above `-createEntity:` method, and handling from a delegate callback.  Most likely, you want to refresh your UI related to the entity's collection when it completes.  If you have multiple UI's reading from the collection, you'll probably want to refresh the common model, instead of just adding an entity into the local view controller's list of objects.  

So, you could add an NSNotification to the `-createEntity` request delegate callback, which always kicks off a refresh request for a collection when an entity is updated.  But maybe you're also segueing between screens, so a new request is going to be sent anyway.  Or maybe you're viewing an entity set that is filtered--which request should be sent in this context?

Having the completion block on the end of the `-createEntity:` method allows you to control the behavior in the specific context from which the operation was started:  
    
{% gist 2c646c85966ddb5a9f71 %}

####Creating the wrapper method
Creating a wrapper method works like this: 

1.  You create a method, that you call directly from your application.  It can have a completion block, that runs at the end of the asynchronous operation, with a set of memory values from the result of the operation

2.  Inside that method, you create a unique identifier, that can be derived from the request itself.  This is where the `-scheduleRequest:` method is superior, since the `SODataRequestParam` can be used for the unique id

3.  Add a listener to `[NSNotificationCenter defaultCenter]` for this unique id.  Use the block version of the `-addObserver:` interface, so that you can add context-specific code to be executed on the response

4.  Invoke the original delegate version of the method

5.  In the `SODataRequestDelegate` methods, reconstruct the unique id for the notification from step (3) from the response object, and post a notification with the response as the payload

6.  Handle the response in the NSNotification completion block from step (3).  Typically, this means parsing the response

7.  Call the completion block for your wrapper method.  Note that the 3 parameters in the `completion()` match the parameters in `completionHandler()` in step (1)

{% gist 074f97695ba11bd8f48f %}

####Profit
You can then put this wrapper block inside a Collection-specific request, so that instead of tagging requests and tracking them into the delegate, or adding NSNotification listeners all over the application, the response to your specific request can be handled directly in the context from which you sent it!(!).

I'll leave you with a final example, that uses the wrapper method you just created to build a clean `fetch:` method:

{% gist 2ed88505c006f56d9d94 %}