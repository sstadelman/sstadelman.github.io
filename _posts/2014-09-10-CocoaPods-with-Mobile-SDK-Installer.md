---
layout: post
title: "Use CocoaPods with SAP Mobile SDK Installer!"
quote: How to use CocoaPods to bootstrap your SAP Mobile SDK app, instantly
image: /media/cover-images/yosemite_entrance.jpg

---

#How to use CocoaPods to bootstrap your SAP Mobile SDK app, instantly

Oh snap! This is awesome.  

##Use CocoaPods
Those of you who know me, know that I'm a big fan of the CocoaPods tool and community for discovering new open-source components for iOS development and library/dependency management.

I've been pushing for broader adoption in the SAP community, but have been somewhat stiemied, due to the fact that our iOS SDK isn't distributed via github.com, or an endpoint accessible to the command-line tools.  I've been hosting a copy of the SDK on our internal github, and had a blog post queued up describing how to do the same in your environment, with a podspec you could modify to link to your repo.  But it was just a couple too many steps... I wasn't optimistic about the adoption.

All that can be thrown out.  

##Seriously, right now.
I was trying to work out how to submit a public podspec for my STSOData framework, which depends on the SAP Mobile SDK, and I ran into this [blog by the team at Gaslight](https://teamgaslight.com/blog/using-local-libraries-with-cocoapods).  It didn't solve the problem I was initially looking at, but I came back to it a few days later, and found this:

**You can just copy [this](https://github.com/sstadelman/NativeSDK-podspec) `NativeSDK.podspec` document into the NativeSDK folder of your installation directory, add this line to your project's Podfile:  `pod 'NativeSDK', :path => "~/SAP/MobileSDK3/NativeSDK/"`, and CocoaPods will read from your local installation.**

Ah.  Yes.

##Check it out
<iframe width="854" height="510" src="//www.youtube.com/embed/sI4nDiSX9Gc" frameborder="0" allowfullscreen></iframe>

##Try it.  Now.

1.  Copy the [`NativeSDK.podspec`](https://github.com/sstadelman/NativeSDK-podspec) file into the **/NativeSDK** directory in your Mobile SDK 3.0 SP05 installation.
2.  Create a new iOS project in xCode
3.  Navigate to the project directory in Terminal, and run `pod init`
4.  Copy this text into the generated Podfile:

        platform :ios, "7.0"

        target "MyTestApp" do
        inhibit_all_warnings!

        pod 'NativeSDK', :path => "~/SAP/MobileSDK3/NativeSDK/"

        end

    Modify the path to match the location of **/NativeSDK** on your own machine.  

    > Note:  if you are referencing NativeSDK headers in the project **\*.pch** file, you must add `pod 'NativeSDK'` to the **Tests** target also.

5.  Run `pod install`
6.  Open the .xcworkspace file, instead of the .xcodeproj file.  The workspace is now your regular working area.
7.  Profit

##I'll wait..
Interestingly, when you add a pod using the `:path` prefix, it gets added to the workspace in the **Development Pods** folder in the **Pods** xcodeproj (instead the **Pods** folder, with the pods downloaded remotely ).  As you can see in the bottom left of the screengrab, the headers are all correctly linked.  If you scroll down in the **Native SDK** folder, you'll also see all the Mobile SDK libs in the **Frameworks** folder, and the bundles and storyboards in the **Resources** folder.  

![Newly created xcworkspace](https://raw.githubusercontent.com/sstadelman/sstadelman.github.io/master/media/blog-images/Native-SDK-podspec1.png)

##For extra credit:
Add an additional pod to your project!  Check out [CocoaControls.com](https://www.cocoacontrols.com/platforms/ios/controls?cocoapods=t&sort=rating), and pick a pod to try out.  The [JASidePanels](https://www.cocoacontrols.com/controls/jasidepanels) pod has 2750 stars on github, and 580 forks, so it looks like a pretty robust project.  

![JASidePanels page on CocoaControls.com](https://raw.githubusercontent.com/sstadelman/sstadelman.github.io/master/media/blog-images/cocoacontrols-jasidepanels.png)

The pod info is in the middle of the page:  copy the `pod 'JASidePanels', '~>1.3.2'` text, and paste into your podfile (below `pod 'NativeSDK'`...).  From the command line, run `pod update`.  The JASidePanels pod will be added to your workspace!

To remove a pod, just delete that line from your Podfile, and run `pod update`.

*CocoaControls.com and JASidePanels are not affiliated with SAP, and rights may be reserved by their respective owners.*
