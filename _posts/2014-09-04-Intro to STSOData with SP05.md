---
layout: post
title: "Intro to STSOData"
quote: A best-practice template for bootstrapping your application with SAP Mobile SDK 3.0 SP05
image: /media/cover-images/lighthouse.jpg

---

#Intro to STSOData
We've been working with the new 3.0 SP05 API set for several months now, and some common usage patterns have developed around the core application components, best practices for MVC, etc.  I've encapsulated these in a helper template called **STSOData**, and am making it available both as an example, and also as a reusable framework to app developers using the SDK.  The STSOData code is not a standard product framework, but it does a lot of the bootstrapping of the core application components with the SDK that you're going to need anyway.  So, it should really speed your development, and let you take advantage of some great features without learning too much about the lower level details of the core APIs.

I want to stress that use of this framework is *not* mandatory with the SP05 version of the SDK, and you can feel free to modify or throw away parts or all of it, as your application architecture requires.  I also want to point out that this code is not indicative of gaps in the API, but is really a stylistic enhancement around principles of MVC, blocks, reactive programming, etc. that are great for iOS developers. 

##Getting STSOData
To add STSOData to your project from github.wdf.sap.corp, download the [repository](https://github.wdf.sap.corp/i826181/sap-odatasdk-template.git), and drag-and-drop the **Template/Framework** folder into your application.  Or, if you're using cocoapods, add https://github.wdf.sap.corp/i826181/sappods.git as a private repository, and then use `STSOData` pod *(recommended)*. See simple 2-step instructions [here](https://github.wdf.sap.corp/i826181/sappods#setting-up-for-private-podspecs).

##Getting Started with STSOData
<iframe width="854" height="510" src="http://www.youtube.com/embed/m4O_PalvIE0" frameborder="0" allowfullscreen></iframe>

##STSOData contents
STSOData is not very large, and it consists of implementations of some of the standard application components that are necessary in the SDK.  

###ODataStores and protocol
You'll see an OnlineStore and OfflineStore, which are implementations sub-classing SODataOnlineStore and SODataOfflineStore, respectively.  Both of the store classes contain the implementation of their constituent protocols.  Stores are discussed in more detail in other posts--for this introduction, just know that they are lower-level containers for the networking/database connections, and for the parsing of OData results.  

One of the key premises for the framework is to enable block interfaces for the developer, so the ODataStore protocol provides block wrappers on some of the delegate-styled interfaces (`openStore:`, and `flushAndRefresh:`).  You'll use the `flushAndRefresh:` frequently in offline apps to sync the local database and backend source, but the `openStore:` method is called internally by the DataController, so you won't need to invoke this directly.

###LogonHandler and its categories
The **LogonHandler** is an update to the **LogonHelper** that has commonly been used in samples over the past year.  This is a core component of the application that handles all of the MAFLogonDelegate implementation, registering with the server and getting connection settings, setting up networking for the application, obtaining and storing credentials, etc.  There's not very much that you need to do with LogonHandler directly.

There are also some categories on LogonHandler, that implement **Usage**, **Logging**, and **E2ETrace** features.  These share a networking stack with LogonHandler, and are initialized during the Logon procedure.  You may interact with the methods on E2ETrace and Logging for starting/stopping, and uploading these log documents.  For Usage, you should interact with the Usage API's directly, since the records are put into a database and uploaded automatically, when the device connects via wifi.  

####Usage

Developer-facing API for logging timers, qualitative, and quantitative data, for analytic analysis.  Requires HCP Mobile Services server.

####Logging

Developer-facing API for error & debug-style logging.  Requires Admin enablement for a particular app instance, and requires integrated Solution Manager system.

####E2ETrace

Internal instrumentation for SAP Passport trace analysis.  Requires Admin enablement for a particular app instance, and requires integrated Solution Manager system.

###DataController and its categories
The **DataController** is really doing the heavy lifting in the application.  It handles the life-cycle of creating the data store, specifying whether it should be running in online or offline mode, and then implementing the request sending & response parsing, is all happening here.  The core concept behind localizing all of this data management into a central controller is to enable us to adhere to MVC, and a core principle that we should never implement and make network requests from within view controllers.  Instead, we trigger network requests (or in the offline case, database requests) from the view controllers, but the response handlers put the data into a central Model; the view controllers subscribe to the Model for their data.

The DataController has an Online and Offline store attached.  The decision for which store a request will be routed through is determined by whether a *defining request* exists for the OData collection referenced in the request.  Defining requests are used when initializing the local database under the OfflineStore, and all requests on collections which exist in the local database will be routed locally.  All others, and function imports, will be sent over the network, via the OnlineStore.  This routing takes place internally in DataController, and you will not need to handle for these cases, beyond determining the scope of your defining requets.

The Create, Update, Delete requests are implemented as a category on DataController.  These implementations are totally collection-agnostic, so you should never need to modify these files.  (They take the information from the entity in the API to derive their resource paths).

The GET requests will obviously be unique to your application.  As a best practice, I recommend you also implement these in your own FetchRequest category(s) on DataController.  STSOData includes a sample of this category, with some example fetch requests for OData resources.  You can copy/paste these examples into your own FetchRequest category; the only change you need to make is to change the target model  from `[DataController shared]` to your own Model. 

###Your additions
To get going, you'll add two files to your project:  

  1. **MyModel.{h, m}**

    Add a singleton model controller to the project.  The Model should include properties, that your view controllers will reference for KVO, etc.  The type of the properties is up to you:  the standard is `NSArray`, but you might opt for `NSMutableArray`, or even `SODataEntitySet`, if you want to parse the `<SODataRequestExecution>` property in the `-scheduleRequestWithResource` callback in your **FetchRequests** implementations.

  2. **DataController+FetchRequests.{h, m}**

    Add your `fetch:` methods for any/all resources you'll be GET-ting in your application.  Feel free to break these into multiple categories, etc.  You can copy/paste the implementation from the FetchRequestsSample, and just need to change the target model to your MyModel.

##Integrating in your project
See the embedded video clip above, at [6:34](https://www.youtube.com/watch?v=m4O_PalvIE0#t=394).



