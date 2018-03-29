---
layout: post
title: "Introduction to the HttpConversationManager"
quote: An overview of the new networking API in SAP Mobile SDK Version 3 SP05
image: /media/cover-images/el-capitan.jpg

---

This is the first in a series of posts on using SAP Mobile SDK 3.0 SP05, using modern features in iOS7 and iOS8, and 3rd-party libraries like ReactiveCocoa.

The iOS SAP Mobile SDK version 3 service-pack 5 has some truly awesome new features, ranging from a major upgrade to the networking layer, to a new "harmonized" API for working with OData services online and offline, to a Usage collection API, to a new cloud-hosted 'Discovery Service', that allows mobile apps distributed through public app stores to discover their connection settings with an end-user's email address.  I'll cover a number of these features for the app developer, and provide an out-of-box framework for getting a new application up and running with the push of a button!

##Topic 1:  The HttpConversationManager

Successful mobile app development in the enterprise, particularly on-premise, really hinges on highly-usable solutions for networking and authentication.  To this end, I'm very proud to share the `HttpConversationManager`, the new networking API for the mobile SDK in Release 3 SP05.  

###Past history
Earlier generations of the mobile SDK networking on iOS depended on networking layers implemented on the `CFNetworking` API set, commonly known as `SDMRequesting`, `SDMConnectivity`, or simply `<Requesting>`.  The core libraries implemented support for basic, mutual-auth, and session-token (including CA Siteminder(c)) authentication types, and were instrumented for traceability with SAP Passport.  

The limitations of the API demonstrated themselves in two main areas:  ease-of-use, and extensibility.  When the API's were first developed, in 2010 and 2011, the concept of standardizing on Apple's native API's for lowering the developer learning curve really hadn't taken hold in the industry; the methodology of joining NSURLConnection and NSOperation (a la AFNetworking) wasn't fully fleshed-out, So, `-requestDidFinish:` and `-requestDidFail` were relatively straightforward implementation choices.  But, as iOS 5, 6, 7, 8 evolved, Apple built up the stack on the `NSURL` API set, including powerful tools like `NSURLProtocol`, NSURLSessionDataTask, completion blocks, Kerberos support, etc.  It was only natural that we should upgrade our communication stack to take advantage of this wide variety of features.

###Development concepts
The basic concept for the HttpConversationManager is the NSURLSession, and the input for the request is a standard `NSMutableURLRequest`!  This was one of the core design principles:  *"*reuse*, then *invent*."*  In fact, a side-by-side comparison shows that the HttpConversationManager and NSURLSession request interfaces are quite similar:

**NSURLSession**

    - (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler

**HttpConversationManager**

    -(void) executeRequest:(NSMutableURLRequest*)urlRequest completionHandler:(void (^)(NSData* data, NSURLResponse* response, NSError* error))completionHandler;

Note that the NSURLSession method returns a NSURLSessionDataTask; the HttpConversationManager `-executeRequest:` method creates a NSURLSessionDataTask internally, by invoking the above method.  Both have similar completion handler interfaces, with the HttpConversationManager also returning the raw NSData of the reponse.

If we are so committed to reusing standard iOS API's, why implement the HttpConversationManager at all?  In fact, our initial prototype was implemented as a category on NSURLSession directly (`NSURLSession+HttpConversationManager.h`).  The answer goes to the point of why we chose this moment to do the update:  support for SAML2 (and OAuth2) authentication, for working with **SAP HANA Cloud Platform**, and the upcoming **SAP HANA Mobile** release.

###Support for SAML2
The NSURL delegate APIs are robust and well-known for handling authentication challenges over HTTP/s, and the NSURLProtocol is an elegant, if lesser known (and minimally documented) method for globally over-riding the URL loader behavior.  But, SAML2 support for native applications presented a challenge:  SAML2 is really designed around interactions taking place in a browser or webview--*not* in a native client.  

Specifically, the HTTP Web Post Profile, which is standard on SAP ID Service, relies on the browser to load HTML content with an embedded `onLoad()` function to redirect to the SAML identity provider ("IdP").  Unfortunately, this HTML content doesn't come back as a 401 authentication challenge:  it returns with a 200!   And, once the redirect to the SAML IdP occurs, the authentication challenge often comes as a HTML form.  So, there is a major benefit in using a browser or webview to handle this interaction, and some filter needs to be available to discover responses which happen to be SAML authentication challenges, create a webview, and then manage the interaction with the IdP culminating in the redirect to the original service provider, where the SessionID token is obtained.  

The solution we adopted requires some configuration on the server (these are already implemented by SAP ID Service, and HANA Mobile, so no additional action is required):

- Service Provider should emit a unique header when responding with a 'SAMLRequest' HTML body
- Service Provider should implement a 'redirect' service, which results in a redirect with a unique query parameter 

 > For additional info on integrating a corporate SAML2 IdP, see [here](https://help.hana.ondemand.com/help/frameset.htm?dc618538d97610148155d97dcd123c24.html).

This info is set to the HttpConversationManager by registering and implementing a SAML2 Configuration Provider.  See in the following example how a SAML2ConfigProvider is added to the `CommonAuthenticationConfigurator`, which then configures the HttpConversationManager itself.  We'll go into detail on the CommonAuthenticationConfigurator in the next section, but in this case, just think of it as a sort of NSURLSessionConfiguration for the HttpConversationManager.

{% gist 00bd9c42cff68ae2829e %}

The end-to-end flow in the HttpConversationManager is then:  

1.  A filter on the response (`ResponseFilter`) looks for the presence of the `responseHeader` (e.g. `com.sap.cloud.security.login`, then the HTML content with "SAMLRequest" string.  This indicates that SAML2 authentication sequence is required.
2.  The ResponseFilter presents a UIWebView, and attempts to load the URI of the `finishEndPoint` (e.g. `/SAMLAuthLauncher`).  This is GET, and should not be a modifying request.  
3.  The Service Provider responds again with the HTML content, with the `responseHeader`, and SAMLRequest body.  This time, the HTML is not intercepted, but is loaded normally in the UIWebView.  This allows the `onLoad()` function in the page to execute, which POSTs the SAMLRequest blob to the SAML IdP.  
4.  The SAML IdP responds with an authentication form, and/or mutual auth challenge.
5.  In the simple authentication form case, the end user enters credentials, and submits the form to the IdP.  In the mutual auth case, the challenge is handled by a NSURLProtocol which is registered by the HttpConversationManager, specifically for the lifetime of the UIWebView.  The NSURLProtocol invokes a `Provider` to obtain the SecIdentityRef, which will be described shortly.
6.  Once the UIWebView has authenticated against the SAML IdP, the IdP responds with a 'SAMLResponse' HTML page, which contains a similar `onLoad()` function to POST the SAMLResponse body to the original Service Provider.
7.  The Service Provider accepts the SAMLResponse assertion, and allows the request to the `finishEndPoint`, which attempts to redirect the UIWebView to a URL with the `finishParameters` query parameter(/s) (e.g.:  `finishEndpointParam`).
8.  The ResponseFilter tests the current URL of the UIWebView, and on detecting the `finishParameters`, determines that the authentication was successful.  It dismisses the UIWebView.
9.  The SessionID token from the Service Provider is now in the Cookie Jar, so the original request is completed (retried) by the HttpConversationManager, and the callback returns to the original request `completionHandler:`.  


###The Configurator, Filters, and Providers
A few components were introduced in the above description, so let's take a look at they way they work together:

- `CommonAuthenticationConfigurator`
- `RequestFilter`
- `ResponseFilter`
- `ChallengeFilter`
- `CredentialProvider`
- `ConfigurationProvider`

In brief, the heirarchy is: 

    + HttpConversationManager

    	- is configured by:

    	+ CommonAuthenticationConfigurator

    		- containing:

        		+ RequestFilters
        		+ ReponseFilters
        		+ ChallengeFilters

        			- which may rely on: 

        				+ CredentialProviders
        				+ ConfigurationProviders


So, after the HttpConversationManager is configured, you may see something resembling the following properties set on the object.  (This is the default set which is provided by the `logonConfigurator` factory method on the `<MAFLogonNGDelegate>` protocol).

<img src="https://raw.githubusercontent.com/sstadelman/sstadelman.github.io/master/media/blog-images/request,%20response,%20challenge%20filters.png">

Note the logic of the implementation:  OAuth2 authentication requires checks at the start of a request (does bearer token exist?), and in the absence of the token, management of a multi-step procedure with checks on the response from the auth token provider.  So, OAuth2 support is implemented with both a RequestFilter *and* a ResponseFilter.

SAML2 authentication (as described above) requires checks on 200-type HTTP responses, to detect payloads related to redirects to the SAML2 IdP.  That logic is encapsulated in a ResponseFilter.  The configuration for the SAML2 interaction is supplied by a `ConfigurationProvider`.  

Basic and Client Certificate authentication can both be handled within the regular HTTP authentication challenge framework.  The NSURLSessionDelegate invokes `-didReceiveChallenge:`, passing the `NSURLAuthenticationChallenge`.  The `UsernamePasswordChallengeFilter` and `ClientCertChallengeFilter` execute on the challenge, and supply the correct credential provider:  If the challenge is for basic auth, the `UsernamePasswordProviders` will be tried; if for client certificate, then the `ClientCertificateProviders` will be tried.  Providers are tried sequentially in their containing NSArray.

In short, you have a very flexible framework around the NSURLSession for managing the authentication, and any custom behavior you need to attach to your networking.


####Out-of-the-box, and Bespoke variants

For adding this to your project, there is an out-of-the-box method, and the bespoke approach.  As mentioned, the LogonManager component has a factory method for auto-generating a complete set of these filters and providers.  The generated providers automatically obtain the credentials from the DataVault, SAP Afaria libraries, and the traditional locations for Mobile SDK apps. (You can see these in the challenge filters above, labeled "LogonAuthenticationConfigurator.)  This option expects the LogonManager component in your app.

If you are developing a very targeted solution (an in-house app with known authentication landscape requirements, a consumer-facing app, etc.), and don't need to work with LogonManager's full feature set, you can implement any subset of the auth filters and providers necessary to your case.  



####`CommonAuthenticationConfigurator`
The verbosely-named CommonAuthenticationConfigurator is like a high-level NSURLSessionConfiguration:  it bootstraps the standard request, response, and challenge filters, and aggregates providers to be set to the HttpConversationManager.  There are two basic cases for the CommonAuthenticationConfigurator.

1.  Use the standard configurator generated by LogonManager factory method

	The `<MAFLogonNGDelegate>` protocol exposes a property named `logonManager`, which can generate a prefab configurator.  If using this prefab configurator, you must wait until the Logon procedure finishes.  (This ensures the DataVault is open, etc.).  So, the best approach is to initialize immediately after `-logonFinishedWithError:` is called on the MAFLogonNGDelegate.

{% gist 39d9348b8b8dad77ed76 %}


2.  Generate a new CustomAuthenticationConfigurator, and set your own providers; add and/or remove filters

	A CommonAuthenticationConfigurator initializes with the SAML2, OAuth2, Basic, and ClientCertificate filters added by default--but lacking providers.  So, it is up to you to implement your own necessary providers (the SAML2AuthConfigProvider for SAML2 auth, the UsernamePasswordProvider for basic auth, etc.).  

	You can optionally remove any filters that you know you won't need (e.g., eliminating SAML2 or OAuth2 filters if you will authenticate only via client certificate). However, these filters will not be invoked unless explicitly instructed 





