---
layout: post
title: "Should I use SODataOnlineStore or SODataOfflineStore, or both?"
quote: What are the differences between Online and Offline stores, how should I use them?
image: /media/cover-images/lighthouse.jpg

---

#Should I use `SODataOnlineStore` or `SODataOfflineStore`, or both?

The OData API for scheduling requests on the `<SODataStoreAsync>` protocol is common for both online and offline stores, which means that you don't need to modify your request/response interface for using either mode.  However, there are two important distinctions in the internal behavior of the stores that you should be aware of.  These should guide your decision to use one, the other, or both.

##Distinctions between `SODataOnlineStore` and `SODataOfflineStore` behavior

1.  The `SODataOfflineStore` is *only* populated by it's *defining requests*.  Requests outside their scope return no records

	The SODataOfflineStore store takes a dictionary of "defining requests" when it is initialized for the first time, which are used by the MobiLink service to fetch data from the back end, and load to the client database.  

	All requests made against the store are then read/writes to the client database.  So, if a request attempts to read a resource from a filter outside the scope of the defining requests, no entities will be returned.

	For example, if the defining request = `"TravelagencyCollection"`, a request for `/TravelagencyCollection`, or `/TravelagencyCollection?$filter=city eq 'Rome'` will return all travel agencies, and all travel agencies in Rome, respectively.

	However, if the defining request = `"TravelagencyCollection?$filter=country eq 'US'"`, a request for `/TravelagencyCollection?$filter=city eq 'Beijing'` will return no entities, even if travel agencies in Beijing exist in the back end system.  Likewise, a request to a separate collection for which there is no defining request, e.g. `/FlightCollection?$filter=fldate gt datetime'2013-09-16T00:00'` will return no records.

	`SODataOnlineStore` sends all requests to the back end, and does not have defining requests.

2.  OData Function Imports are not supported by `SODataOfflineStore`

	Requests to the the SODataOfflineStore are filtered by the `$metadata` document, and requests which are not related to a collection are rejected with an error.  So, Function Imports, even if they reference collections which are in the scope of the defining requests, are not allowed on the SODataOfflineStore.

	Function Imports are supported by the `SODataOnlineStore`.

##Using both `SODataOnlineStore` and `SODataOfflineStore`

There are certainly use cases for pure 'online' apps, which make every request to the back end.  In this case, just use SODataOnlineStore.

There are also a few use cases for pure 'offline' apps, (though many fewer, I think), where the use case is highly transactional, but the relevant data is expected to be fixed during the usage (no live data is expected from the back end).  For example, if the application is designed to enter customer orders, and all the product data is constant, a defining request can be added for the product info, and it will always be read locally, while the customer orders are written and edited locally.

However, if the application has some data which should be offline-enabled (like customer orders), but you also need access to real-time pricing or inventory information on products, then you should use SODataOnlineStore and SODataOfflineStore together, for a 'mixed mode'.

##An example implementation

I always work with a central **DataController** singleton in my apps, which handles the configuration of the online and offline stores, and the request/response delegates.  That way, I can write a `-fetch:` request, without needing to worry about where the request is going.

I accomplish this by creating both a `localStore` (offline store) and `networkStore` (online store) on the DataController.  The defining requests for the offline store are set by the developer, from the AppDelegate.  Then, all my CRUD requests in the app invoke the `-scheduleRequestForResource:withMode:withEntity:withCompletion` method.

{% gist e1a2714f2442b295d35a %}

When the request is being handled in the implementation of `-scheduleRequestForResource:...`, an internal method `-(id<ODataStore>)storeForRequestToResourcePath:` is invoked.  Here, I test to see whether the `localStore` or `networkStore` should handle the request.  


{% gist e609a71b8e6c86bd8088 %}

First, I do a check of the `workingMode`.  The default value in my framework is `WorkingModeMixed`, which means that both Online and Offline stores are used in the app.  But, if only one is available, that one is returned.  (In this framework, the developer should set an alternate working mode `WorkingModeOnline` or `WorkingModeOffline` from the AppDelegate `-applicationDidBecomeActive:`, if desired).  Also, if the app is in Mixed mode, but there are no defining requests, then the request should go over the network.

If the app is in Mixed mode, and there *are* defining requests, then I check to see if the collection of the incoming request matches that of one of the defining requests.  If so, then it should read from the local database.  If not, or if the request happens to be a Function Import, then the comparison will fail, and the online store is selected.

> Note:  in the above code snippets, the `<ODataStore>` protocol is my own interface, which adds a block-wrapper to the `openStore:` method.