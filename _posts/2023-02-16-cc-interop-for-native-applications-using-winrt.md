---
title: "C++/C# interop for native applications using WinRT"
date: 2023-02-16T20:46:00.002Z
slug: cc-interop-for-native-applications-using-winrt
draft: false
layout: post
---

When you search C/C++ C# interop on google you find a lot of articles about it, but many of them (if not all) are focused on P/Invoke interop, which in the end is a "C" style interop, not a C++ one.

NOTE : the repository with the code for this article is here: [https://github.com/QbProg/CppCSInterop](https://github.com/QbProg/CppCSInterop) --

There is a number of tools (like [CppSharp](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwj57pen3v78AhVmSfEDHXjHAm8QFnoECBIQAQ&url=https%3A%2F%2Fgithub.com%2Fmono%2FCppSharp&usg=AOvVaw2Dw8MEcJvaSevKH4Wy3oCN), [SWIG](https://www.swig.org/), or others that can be found online) which are able to generate bindings from C++ to C#, even in a cross platform way.

On Windows, another common option is to use C++/CLI, which works well in both directions. Unfortunately [it's becoming abandoned](https://developercommunity.visualstudio.com/t/vs-2019-ccli-support-c20concept/1274321) as it doesn't support C++20 and probably never will.

Still on Windows, there's another possibility for seamless interop : COM interfaces. This works without many troubles specially if one uses only the COM-ABI aspect (which I [recently found being referred as nano-COM](https://devblogs.microsoft.com/oldnewthing/20230130-00/?p=107761))

In Windows 10, finally, there's yet another option which works for native applications too and, thanks to higher level libraries is almost easy to integrated and implement, without getting lost in all the COM details which made it cumbersome to implement previously. The technology is [WinRT](https://learn.microsoft.com/en-us/windows/uwp/cpp-and-winrt-apis/intro-to-using-cpp-with-winrt), the same which originally powered UWP, and now can be used from Win32/desktop code too.  

In this article I will show you my jurney for a interop system between .NET (core) and native C++ desktop code using WinRT.

Since WinRT and it's various projections are officially the de-facto technology used for all the new Windows API developments (e.g. WinUI 3), the technologies used here are well documented in the Microsoft documentation.  
Though, all the examples and tutorials are much more focused on the user/consumer side that on the authoring side; and gluing them all together was indeed a little complex and required much trial-and-error, so I'm hoping that this article could be of help for someone.

Any feedback and improvement is welcome!  

## **Motivation and requirements**

In my projects, I'm looking specifically for way to replace C++/CLI.  
My applications are native Win32 C++ applications (some even using Qt), and I have a number of components which are implemented using C# that I need to talk to; on the other side, I'd like to expose some C++ classes to the C# components too, so bidirectional interaction is needed.  

This is the configuration and requirements of the project we will be using:  

* Native Windows desktop application target  
  A complex unpackaged native C++ Win32 application composed by many executables and DLL's;  

* Run on all Windows 10 versions (at least starting from 1607)  
  As today the oldest Windows LSB version will be supported until 2026, and unfortunately I have customers using it, so this was an hard requirement.

* CMake as build system for c++ (using Visual Studio generators)  

* Bidirectional communication  
  This means that I want to call C++ code from C# and C# from C++.

* Bidirectional activation (i.e. create C++ objects in the C# side and vice-versa)  
  In addition of **passing** objects I want to be able to **instantiate** objects on both sides.

* Avoid too much code-bloat  
  Adding interfaces and functions should be simple, possibly done only once, and without too much complexity.  

* Internal interop only  
  Generated interop code will be used internally only by the application. We'll skip everything about versioning, compatibility, etc...  

* App-local deployment  
  I want to deploy all(\*) the dependencies in the application directory.

* Registration-free activation and manifest-free activation  
  Since the interop is done for interal use, I want to absolutely avoid registration the components; in addition, I dont't want to even manually create a manifest declaring all the classes exposed in C# or C++.  

The jurney to fullfill all these requirements was kind of tricky but in the end I succeeded and I'll show you all the steps here.  
The repository of the code, with a working example, is to be found here in github.  

## **About WinRT**

WinRT technology was introduced on Windows 8 to support UWP, and it was initially only available to UWP application developers.  
Later on, on Windows 1809, Microsoft made the technology available to desktop applications too (Desktop Bridge).  

In this article I take for granted some WinRT introductory knowledge and I will focus on the interop mechanism. I'm not really a WinRT expert on my side too. 

From an interop perspective, WinRT and it's projections provide a simple way to declare and pass primitives and structured data, and all the gluing is done by the system.  

Note that WinRT types are really required for interop purposes only (e.g. to pass strings and arrays) and the usage of Windows UWP API's can be almost totally avoided. In my tests I just used native C++ functions together with some Windows.Forms functions. 

On the other side, nobody forbids to pass around UWP or WinUI objects and data and do interop in a full modern WinUI application.

#### **About .NET**

These examples work with .NET 6. It seems to me that C#/WinRT don't support .NET framework 4.8; I may be wrong though, but the tooling is so well integrated in CS/WinRT that I didn't bother to check older technologies.  

## **Tools used**

In this article a number of tools were used:  

* Visual Studio 2022, focused only on the x64 platform
* CMake (using the latest versions)
* [nuget command line tool](https://www.nuget.org/downloads)
* [cppwinrt](https://github.com/microsoft/cppwinrt)
* [CsWinRT](https://github.com/microsoft/CsWinRT) on .NET 6.0
* [Microsoft xlang (undockedRT)](https://github.com/microsoft/xlang) for Windows 1607 support

Note that Visual Studio 2022, does not need UWP support installed. Just make sure that you have the correct Windows SDK versions, else the C# compilation will fail.  

cppwinrt and CsWinRT will be added using nuget packages. Note that the nuget command line tool is not distributed with Visual Studio or the .NET SDK, so it has to be downloaded manually.  

#### **Setting up the repository**

You'll find the repository here: [https://github.com/QbProg/CppCSInterop](https://github.com/QbProg/CppCSInterop)

Just clone it and you'll find all the referenced material. Run <setup.bat>: the cmake scripts automatically downloads a bunch of dependencies and source code around the web.   
Please make sure to have the proper dependencies installed, specially the correct version of the Windows SDK. If not, just add it using the visual studio installer.

 **The starting point**

Microsoft has two tutorials for the creation of C# components and C++ components and consuming on the other side. These work for native desktop applications too.

[https://learn.microsoft.com/it-it/windows/apps/develop/platform/csharp-winrt/net-projection-from-cppwinrt-component](https://learn.microsoft.com/it-it/windows/apps/develop/platform/csharp-winrt/net-projection-from-cppwinrt-component)

[https://learn.microsoft.com/it-it/windows/apps/develop/platform/csharp-winrt/create-windows-runtime-component-cswinrt](https://learn.microsoft.com/it-it/windows/apps/develop/platform/csharp-winrt/create-windows-runtime-component-cswinrt)  

The code contained in these projections is good but there are a few drawbacks:

* They use nuget to expose the package to the other side. This is a great choice for library authors, but a little too much for internal interop purposes.  

* They use Visual Studio and not CMake for C++ projects.

* They use reg-free but not manifest-free activation.

To use CMake to develop cppwinrt projects I found this great starting point:

[https://github.com/fredemmott/cmake-cpp-winrt-winui3](https://github.com/fredemmott/cmake-cpp-winrt-winui3)

Thanks to the author for it. This repository is focused on WinUI 3 but gives a starting point to build a cppwinrt cmake project and,most importantly, it contains a workaround for a blocking cmake+cppwinrt compilation bug.

## **Creating a c++ WinRT component**

So, after this long intruduction, let's start by creating a C++ WinRT component. In the repository, the reference is the CppComponent project.  

cppwinrt is the official native WinRT projection; it is proposed as nuget packages and it basically provides the cppwinrt compiler which is capable to generate the boilderplate code for C++ users to implement the exposed classes.  

The process of adding a c++ class (and any of it's functions) gets simply as

\- creating a midl file with the class and it's functions (Class.idl)  
\- adding to CMake and building  
\- implementing the functions in the Class.cpp file  

the library generated can be automatically be used in C# and all of the types declared will be available there.  

In our example we created a cpp midl here:  

```midl
namespace CppComponent  
{  
    [default interface]
    runtimeclass Class  
    {  
        Class();
        String Hello();  
    }  
}
```

and the implementation looks like this:

```cpp
#include "pch.h"  
#include "Class.h"  
#include "Class.g.cpp"  

namespace winrt::CppComponent::implementation  
{
    hstring Class::Hello()  
    {  
        return L"Hello World";  
    }  
}  
```

Nothing really fancy. Just note the usage of cppwinrt projected types (hstring), which can also be constructed from standard types using helper functions. 

After building we get a CppComponent.dll which only depends on the visual c++ runtime and that be used in standalone way.

The CppComponent.dll contains the implementation of the c++ part, plus some generated cppwinrt code (the activation factory etc...)

In addition there is a CppComponent.winmd metadata file which will be used to generate projections from other languages.  

In the IDL definition of the c++ part it's possible to add interfaces only that can be later used in C# to implement functionalities without having to expose a WinRT component on the C# side too. This may be useful if the scope of interop is just exposing the cpp object to C#.

### Notes on the CMakeLists.txt file

The CMake file for the Cpp component has a few addition in respect of a standard C++ project.

First of all, in the main CMakeLists.txt file, nuget gets downloaded with the cmake download functions, and the tool is stored in the "tools" directory.  

Then nuget is used to restore cppwinrt nuget packages.  

In the project CMakeFiles we then define a regular c++ target.  

Since cppwinrt auto-generates the module definitions, we add the auto-generated file as a cmake dependency so it correctly recognizes it. In addition there are some global Visual Studio generator define that basically set the visual studio project to work correctly with the winrt options.

```cmake
add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/CppComponent.dir/$<CONFIG>/Generated Files/module.g.cpp" COMMAND echo )  
set(ADDITIONAL_FILES packages.config "${CMAKE_CURRENT_BINARY_DIR}/CppComponent.dir/$<CONFIG>/Generated Files/module.g.cpp")
```

Finally, we copy the generated binaries to a common "distrib" directory; this step is explained below.  

### DLL Naming conventions

Since we'll want to use manifest-free activation it's important that the name of the DLL matches the name of the root namespace of our components. So in this case the namespace is CppComponent and the DLL is called with the same name. Nested namespaces and classes can follow this rule too.

## **Generating a C# projection for the Cpp component**

To use any WinRT component from C# a CsWinRT projection assembly needs to be created. Once created, this assembly can be referenced from other C# projects like a regular assembly.

Unfortunately there seems to be no alternative to creating a separate C# projection assembly for each C++ WinRT component used, so we'll stick to this rule for the moment.  

In the main CMake file I added the C# projection project as an external project. The C# project is a SDK-style project based on .NET 6.  

```cmake
INCLUDE_EXTERNAL_MSPROJECT(CppComponentProjection "${CMAKE_CURRENT_SOURCE_DIR}/CppComponentProjection /CppComponentProjection.csproj" PLATFORM "x64")
```

This is a regular C#/WinRT project, created with Visual Studio. A few considerations to look at:

* The target framework has an explicit reference to windows (see the tutorial)  
  `<TargetFramework>net6.0-windows10.0.22000.0</TargetFramework>  
  <Platforms>x64</Platforms>  `
  Make sure you have the exact windows SDK version installed, or the compiler will complain  

* Added the C#/WinRT reference to the project  
  `<PackageReference Include="Microsoft.Windows.CsWinRT" Version="2.0.1" /> `

* We referenced c++ project using a relative path. Be careful of this if you change the directory structure.  
  `<ProjectReference Include="..\\build\\CppComponent\\CppComponent.vcxproj" />`

* We referenced the CppComponent in the CsWinRT property group.  
  `<CsWinRTIncludes>CppComponent</CsWinRTIncludes>  `

In the end we got our C# projection of the component, that we can use in any other C# project like a regular assembly.  

Note that, despite the Microsoft tutorial suggests, I didn't create a nuget package for the projection. If you are a component designer it is surely the way to go, but I just want to use these classes as an interop mechanism for my application.  

#### Using the C# projection

In the CMake example I added a test C# executable, which uses the projection to instantiate the component.  
Note that there were two issues in using the projection from the C# test executable:

* The C# project needed to reference CsWinRT too; this was a bit unexpected but was required.
* The projection's "ProjectReference" needed a special attribute `SkipGetTargetFrameworkProperties="true" `

I don't really know what's going on here, if anyone has an idea please let me know.

## **The other way around: creating a C# WinRT component and using it in a native C++ project**

The process described in the previous section is particularly useful if you have a C# application and want to integrate C++ code or libraries.  
For native C++ applications the scenario may be the exact opposite: since C# has a number of great libraries and components, and UI programming is indeed more easy and intuitive to do in a managed environment, one could start integrating C# classes in an existing C++ native application (this was my case indeed).

So, let's create a C#/WinRT component for comsumption. 

In this case the Microsoft tutorial is a good starting point too. The creation of the C# component is easy, and I added it to CMake using the same external project method.

In our source the component is **CSComponent** .

In addition of just adding a C# class, I started using the classes from the previous C++ component projection, to show that mixing different component in different languages is possible.  

In a C#/WinRT component every public class is exported in the metadata (winmd) and will be usable from the C++ side. The projection system will automatically handle the marshaling and type matching for you.  
There may be attributes to control the details of marshalling and export, [so please refer to the CsWinRT docs in this regard.  

```csharp
namespace CSComponent  
{
    public sealed class CSClass  
    {         
        public string Hello()  
        {  
            CppComponent.Class c1 = new CppComponent.Class();  
            return c1.Hello();  
        }  
    }  
}  
```

We still used the convention that the root namespace should match the assembly name. This is really important in C# too, as we'll see later.  

The C# component generates the CSComponent.dll assembly and it generates a number of other DLL's too, namely WinRT.Host.dll and WinRT.Host.shims.dll too.

These additional values are crucial for the usage in a manifest free activation, explained below.

## **Using the C# component from C++**

Native c++ projects can use winrt directly without having to resort to any special UWP attribute or setup.

The main thing here is to point winrt to use the winmd file generated by the C# application; cppwinrt will generate the C++ interfaces and activation code for you.

From the user point of view it's just a matter of allocating and using the class in a typical cppwinrt way:

```cpp
CSComponent::CSClass ex;  
std::wcout << ex.Hello().c_str() << std::endl;  
```

In my case I wanted to use the C# component not from the main EXE application, but from a DLL. This scenario is a slightly more complex, but exposes a number of critical issues in using the C# component in native code.  

In our scenario we created a DLL (CSComponentClient.dll) and with a single export (dll\_main) we called it from the main EXE. This is not really different than calling it from the main EXE.

If you follow the official tutorials, you see that you need to add the C# component entries in the main EXE manifest. In our case, we still want to follow the manifest-free activation route, and this is where we found some issues: if you run the code without modifications you'll see that it does not work.  

Before solving the issue, let me write a quick recap of how COM/WinRT activation works on the C++ side.  

### **Activation (instantiation) of C# WinRT classe****s**

The activation system in winrt is kind of complex, and evolves from the venerable COM system.  

When you want to create a COM/WinRT class, you generally need what is called an _Activation Factory_, i.e. a special factory class that instantiates concrete classes objects.

In COM, each class being instantiated needed to be registered in the Windows registry (remember the infamous regsvr32 tool?); this means that to use a COM component the installer/deployer of the component would have to register the component itself in the register. The registration entry lets the COM system know in which DLL the activation factory of a class is implemented and where to find such DLL when an object of the class is requested.  

Later on (in the Windows XP era IIRC) [reg-free COM activation](https://learn.microsoft.com/en-us/previous-versions/dotnet/articles/ms973913\(v=msdn.10\)?redirectedfrom=MSDN) was introduced; this allowed component users to distribute components locally in the application without having to register them, and most importantly to avoid clashes with different component version installed globally at system level.   
The main idea was to just declare where to find all the activation factories inside the manifest file of the main EXE application instead of registering them in the registry.

Unpackaged WinRT apps usually follow the same approach. You can use WinRT components in your application as long as you declare these in the main manifest EXEs.  

If you look at the tutorial step, you'll see a manifest like this in the native EXE console application:

```
<?xml version="1.0" encoding="utf-8"?>
<assembly manifestVersion="1.0" xmlns="urn:schemas-microsoft-com:asm.v1">
    <assemblyIdentity version="1.0.0.0" name="CppConsoleApp"/>
    <file name="WinRT.Host.dll">
        <activatableClass
            name="AuthoringDemo.Example"
            threadingModel="both"
            xmlns="urn:schemas-microsoft-com:winrt.v1" />
    </file>
</assembly>
```

You see a few things here:

* The manifest is declared on the EXE file
* The referenced DLL is **WinRT.Host.dll** and surprisingly not CSComponent.dll
* Each different class exported by CSComponent need to be put here

Turns out that each C# winrt component generates two additional DLLs containing the activation factory code; these DLLs are named WinRT.host.dll and WinRT.host.shim.dll  
Some more details can be found [here](https://github.com/microsoft/CsWinRT/blob/master/docs/hosting.md).  

These Host DLLs contain native code to 

1. Activate the .NET core engine if required 
2. Load the CSComponent assembly. 
3. Export the native DLL entry points for the winrt system use them.

This approach works, but for requirements it has a number of issues:

* For each class exposed an update to the client (EXE) manifest will be needed.  

* If case of multiple components, the WinRT.host.dll name clashes, as each component should have it's own unique version.  

* Which manifest is to be updated if it is not the EXE to call the C# component but a different DLL?  

As already explained above, in our case the solution adopted is to a**void manifests entirely** using manifest-free activation.

### Manifest free activation for our C# component

Like the C# counterpart, cppwinrt supports manifest-free activation too. The issue here is that the rules used by the cppwinrt version don't match the (default) C# DLL naming convention. In particular, the cppwinrt activator won't look for _WinRT.host.dll_ name, so that DLL should be renamed. If you looked at the [previous link](https://github.com/microsoft/CsWinRT/blob/master/docs/hosting.md) 's information, there are three potential naming conventions to use. We choose to rename the host DLL and keep the main assembly name intact, because in the end it's a more intuitive approach.

By default cppwinrt will look for a DLL with the same name(s) of the root namespace. The same logic applyes for nested namespaces. So in our case the loader will look for the _RoGetActivationFactory_ function inside the **CSComponent.dll** 

Unfortunately the CSComponent.dll is a C# assembly and does not export any such function. The function is indeed contained in the auto-generated WinRT.Host.dll

As the winrt manifest-free activation logic cannot be customized and there's no way to tell the cppwinrt system to load the WinRT.Host.dll instead. This is a hard blocker for our scenario ([which I reported here](https://github.com/microsoft/CsWinRT/issues/1281)), but it seems it was a non-intended use-case.  

Fortunately, there's a detour on the winrt default activation function, probably implemented for testing purposes, that can be hacked to implement the correct logic. It's just a matter of setting a function pointer in winrt and the custom activation logic will be used in addition of the standard one.

To do this we set the detour once per process (in the EXE main or in the DLL main or any other utility function) and there we implement our custom DLL matching logic. 

```cpp
winrt_activation_handler = custom_winrt_activation_handler;  
int32_t __stdcall custom_winrt_activation_handler(void * classId, winrt::guid const & guid, void* result) noexcept
```

**note:** the signature of this function could be cppwinrt version-specific  

The detour code is copyed straight from cppwinrt source, the only logic that changes is that it tryes one more DLL with the .Host.dll suffix  
See the WinRTCustomActivation.cpp for the sample code.  

So in our case it will look for `<root\_namespace>.Host.dll` , and it will only do so after the standard DLL matching has not failed.  

`CSComponent.dll -> fail  
CSComponent.host.dll -> success  `

In the C# side of the project, we'll implement a custom post-build event which copies the local WinRT.Host.dll to a more specific name. We'll end with these names in the output directory:  

`CSComponent.Host.dll  
CSComponent.Host.runtimeconfig.json  `

## **So many DLL's , so many files**

At this point the example should run, but it does so because there are post-build events that put the DLL's in a specific "distrib" folder.

Without these steps the application would fail since the manifest-free loaders won't be able to load the DLL's in the application path. This is of course similar of how regular EXE and DLL's work.

So if you look in the cmake and csproj files you'll see that there is some custom post-build logic to:

\- Copy the CppComponent DLL's in the application directory  
\- Copy the CppComponent projection in the application directory  
\- Rename the CSComponent WinRT.Host.dll to CSComponent.Host.dll  
\- Copy the CSComponent files to the application directory  

After the copy, the EXE file need to be run from the application directory and not from the build directory.  
This is still done in CMake by setting the proper visual studio debug options at configure time.  

Also note that the C# components require the .runtimeconfig files to propertly run, dont forget to deploy these too!  

## **Applocal deployment**

Up to this point the application will correctly run in a developer workstation. To run it in a standalone ("clean") machine, two things are required:

* **The Visual c++ redistributables**  
  These can be copied directly in the application directory structure to avoid the system installation. A configure/install step can be used for this. In our example we copy these from the visual studio directory.  
  
  Note: somewhere in the UWP days there was a need for VCRT forwarders, but I found it's not the case anymore.  

* **The .NET 6.0 re-distributable installed**  
  I could not find to avoid the system-wide installation. Surely a dotnet self contained publish could work, but I didn't find a straightforward way to do this for a mixed C++/C# project. Any contribution is welcome on this regard :)  

## **Supporting versions prior Windows 10 - 1809**

All these examples run smoothly up to Windows 10 versions 1809, which introduced Desktop Bridge and UWP API's usage for unpackaged applications.

Fortunately enough, Microsoft made available the [xlang project repository](https://github.com/microsoft/xlang) , which contains the winrtact.dll that,when loaded inside one process, will detour the WinRT activation APIs and make the whole system work in windows versions up to 8.1.  
I only tested this up to windows 1607, which was my minimal system requirement. Finding a virtual machine for it was already hard enough :)

To get that library company one has to download the repository and build it by hand using the Visual Studio solution provided.  
Then, to enable detouring, one has to link the winrtact.dll by forcing a symbol on it (or just use LoadLibrary). The library is smart enough to auto-detect when it's needed so this can be used in all Windows versions.  

Since I was using a cmake project system, I quickly built a cmake script (only for the x64 version of that library) and build it directly in the main project structure.  
Since I don't want to mess with the Microsoft source licenses, the real sources are downloaded on the fly by the cmake configure script and copied in the source directory.  
The winrtact.dll is then copied to the application directory with a post-build event.

We then use the library as a link target for the main executable and force a linker symbol on it.

I didn't find many users of this library in the web, it seems only winget uses it, but I'm not sure. It will be quickly outdated as these older versions of Windows get replaced by newer ones.

## **Wrapping up**

Finding a solution for all the issues has taken a long time, but in the end I got a template project that I can use in my production code.

All of this may seem complex, but once you have an example working it can be quickly re-used and the actual integration time is near zero.

Some steps could be further automated: for instance the C++ -> C# projection projects could be auto-generated in the cmake-configuration steps using a configuration template.

Of course the C++ and c# projects can be built separated too, and possibly use different build systems. In this case I preferred a single solution to quickly work on both languages.

I hope this is just a shortcut for developers doing the same thing and finding the same roadblocks.

Please provide me any feedback if any of these instructions is wrong or if there's a better way to do some things. 

As next steps, I will try to do a proper interop project in my production projects and report here any issue found.