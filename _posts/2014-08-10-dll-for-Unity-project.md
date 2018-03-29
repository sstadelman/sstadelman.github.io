---
layout: post
title: "Developing C# .dll for use with Unity in Visual Studio"
quote: Setup instructions for creating dll for use in Unity project
image: /media/cover-images/chicago-met-mus-art.jpg

---

#Developing C# .dll for use with Unity in Visual Studio

1.  Create new project, for type 'Class Library'

2.  Add **UnityEngine.dll**ference to your ClassLibrary.  

	+ Right-click on ClassLibrary, and select *Add Reference*
	+ Go to *Browse*, and navigate in Unity install directory to find UnityEngine

	    C:\Program Files (x86)\Unity\Editor\Data\Managed\UnityEngine.dll

3.  In **Class1.cs**, delete the `namespace` block in the generated class template

	    namespace ScenarioEngine
		{
		    class MyUnityClass
		    {
		    }
		}

4.  Add `using UnityEngine;` to class imports

5.  Add class declaration, using `MonoBehavior` type

	    public class MyUnityClass : MonoBehaviour
		{

		} 

6.  Change the Target Framework to **.NET Framework 3.5**

	Right-click on ClassLibrary, and select `.NET Framework 3.5` under *Target framework*

	> .NET Framework 3.5 does not install by default, select *Install other frameworks* in the drop-down to be linked to download page on MSDN.  

	> You will need to restart Visual Studio after installing

7.  Specify **Release** build profile in Visual Studio for the dll

8.  **Build** the Solution

9.  Open a Unity Project, and add dll.  The dll must be added as an Asset into the **Plugins** folder.  

	+ The Plugins folder is a default member of the *Project* file structure.  
	+ Right-click on Plugins folder, and select *Import New Asset*.
	+ Locate the dll location in the project `\bin` folder

	    C:\Users\Stan\Documents\Visual Studio 2013\Projects\ScenarioEngine\ScenarioEngine\bin\Release

10. Open a new or existing C# script in MonoDevelop, and develop normally
     
    Initialize objects of type `MyUnityClass`, call functions by dot notation.

    > Auto-complete should include `MyUnityClass`

Additional YouTube reference & credit [Robert Banister](https://www.youtube.com/watch?v=bTQ0WOPOhtY)

