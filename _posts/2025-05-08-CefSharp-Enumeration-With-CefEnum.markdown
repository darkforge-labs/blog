---
layout: post
title:  "CefSharp Enumeration With CefEnum"
date:   2025-05-21 09:00:00 +0200
categories: CEF CefSharp CefEnum Thick-Client .NET
---

## TLDR
[CefSharp](https://github.com/cefsharp/CefSharp/) is widely embedded in .NET-based Thick-Clients, often without proper hardening or awareness of its security implications. For researchers and red teamers, this creates opportunities for stealthy exploitation, persistence, and even RCE if misconfigurations are identified.

In this post, we explore common misconfigurations and attack vectors encountered during testing. We’re also releasing a new enumeration tool to help you quickly identify and fingerprint CefSharp instances in your engagements.

**Download CefEnum**: [https://github.com/darkforge-labs/cefenum](https://github.com/darkforge-labs/cefenum)


## CEF
Before we discuss CefSharp, let's look at [CEF](https://github.com/chromiumembedded/cef) (Chromium Embedded Framework). 

> Chromium Embedded Framework (CEF). A simple framework for embedding Chromium-based browsers in other applications.
> CEF is a BSD-licensed open source project founded by Marshall Greenblatt in 2008 and based on the Google Chromium project. Unlike the Chromium project itself, which focuses mainly on Google Chrome application development, CEF focuses on facilitating embedded browser use cases in third-party applications. CEF insulates the user from the underlying Chromium and Blink code complexity by offering production-quality stable APIs, release branches tracking specific Chromium releases, and binary distributions.

- [https://github.com/chromiumembedded/cef](https://github.com/chromiumembedded/cef)

## CefSharp
![CefSharp Logo](/assets/images/04fe27e7a409e3c463dec186a49254d49f4d1ea2.png){:width="300px"}

What exactly is [CefSharp](https://github.com/cefsharp/CefSharp/)? 

> CefSharp lets you embed Chromium in .NET apps. It is a lightweight .NET wrapper around the Chromium Embedded Framework (CEF) by Marshall A. Greenblatt. About 30% of the bindings are written in C++/CLI with the majority of code here is C#. It can be used from C# or VB, or any other CLR language. CefSharp provides both WPF and WinForms web browser control implementations. 

- [https://github.com/cefsharp/CefSharp/](https://github.com/cefsharp/CefSharp/)

## Web Based Thick-Client
Much like Electron, CefSharp enables developers to build desktop applications using web technologies, tailored specifically for the Windows and .NET ecosystem. Its real power lies in its ability to expose internal .NET objects to client-side JavaScript, creating a bidirectional bridge between the client frontend and the user’s system.

This architecture can allow web pages to interact with privileged system functions, something that, when misconfigured, becomes a valuable entry point for security researchers. It can lead to file access, method invocation, or even command execution directly from the browser context. 

## Attack Scope
Finding client-side vulnerabilities like XSS in a Thick-Client may seem unconventional, especially since users don’t interact with the app like a typical browser. But as we've seen repeatedly with Electron-based apps, these flaws can lead to severe consequences—and CefSharp is no different. When XSS is combined with CefSharp’s JavaScript bridge to exposed .NET objects, a persistent XSS can quickly escalate into RCE. 

Before crafting any payloads, though, you need to understand which objects are exposed to the client/browser front-end via CefSharp.

## Finding Exposed Objects, With Access To The Client Source Code
This is straightforward. According to the CefSharp wiki, there are two methods to register your object with `browser.JavascriptObjectRepository`. In both examples below, the exposed object is named `boundAsync`, referencing the `BoundObject` class. (Note: bindable object names will generally follow camelCase, not PascalCase.)

```cpp
//For async object registration (equivalent to the old RegisterAsyncJsObject)
browser.JavascriptObjectRepository.Register("boundAsync", new BoundObject(), true, BindingOptions.DefaultBinder);
```

And:

```cpp
browser.JavascriptObjectRepository.ResolveObject += (sender, e) =>
{
	var repo = e.ObjectRepository;
	if (e.ObjectName == "boundAsync")
	{
		BindingOptions bindingOptions = null; //Binding options is an optional param, defaults to null
		bindingOptions = BindingOptions.DefaultBinder //Use the default binder to serialize values into complex objects
		bindingOptions = new BindingOptions { Binder = new MyCustomBinder() }); //Specify a custom binder
		repo.NameConverter = null; //No CamelCase of Javascript Names
		//For backwards compatability reasons the default NameConverter doesn't change the case of the objects returned from methods calls.
		//https://github.com/cefsharp/CefSharp/issues/2442
		//Use the new name converter to bound object method names and property names of returned objects converted to camelCase
		repo.NameConverter = new CamelCaseJavascriptNameConverter();
		repo.Register("boundAsync", new BoundObject(), isAsync: true, options: bindingOptions);
	}
};
```
From here you simply locate the "BoundObject" class, and inspect what methods are available.

## Interact with the client
If a CefSharp client retrieves a malicious page, that page can bind to the exposed object and invoke its methods directly. We've created a vulnerable test application called BadBrowser for you to try this out yourself:

* [https://github.com/darkforge-labs/badbrowser](https://github.com/darkforge-labs/badbrowser). 

In the code below, we use `CefSharp.BindObjectAsync("customObject")` to bind to `customObject` and execute the `WriteFile()` method with `window.customObject.WriteFile("test.txt")`.

```javascript
CefSharp.BindObjectAsync("customObject");
window.customObject.WriteFile("test.txt");
```

or 

```javascript
function bindAndRun(){CefSharp.BindObjectAsync("customObject").then(function(n){window.customObject.WriteFile("test.txt")}).catch(function(n){console.error("Failed:",n)})}window.addEventListener("load",function(){setTimeout(function(){bindAndRun()},2e2)});
```

Now, when we view a page containing the above JavaScript in BadBrowser, a file named test.txt is written to `C:/tmp/`.

## Delivering the Payload (Getting the Client to Load Your Page)
If you've discovered a persistent XSS vulnerability in the client’s portal, you can embed your payload into the page, launch the client, and trigger the XSS as it renders.

If direct navigation is restricted and custom URLs can't be entered, there’s a common workaround: drag a link to your test page from another browser and drop it into the client window.

![Drag Link Across](/assets/images/28eb4850e5fc5953bd5ad52b663689356606ce33.png)

## Finding Exposed Objects, without access to the source code
If you're testing a client without access to its source code, you can use [CefEnum](https://github.com/darkforge-labs/cefenum). This simple tool helps detect CefSharp-based clients and identify .NET objects exposed to JavaScript.

When you launch CefEnum, it starts an HTTP listener on port 9090 (configurable via the -port flag). Upon a client connection, CefEnum first delivers a wordlist to the client, which is stored in the front-end. These words are then used to fuzz and guess exposed object names. The tool includes a default wordlist based on [PortSwigger's param-miner](https://github.com/PortSwigger/param-miner). 

After that, CefEnum checks whether the connecting client is running CefSharp.

![Client connected](/assets/images/743b5ddf541b740b3fe0d7f46e45a2d0c9f4f1e9.png){:width="500px"}

You can interact directly with the client from the command line. CefEnum will pass messages to the clients over a WebSocket. You can either type JavaScript directly into the console, or use the built-in helper functions to do the work for you.

![Client connected](/assets/images/76b81351e6640094863a96e68eb51d7e0ed55819.png){:width="500px"}

## Fuzz
This is the core feature of CefEnum: it attempts to bind to every object name in the wordlist. For each entry, the page executes `CefSharp.BindObjectAsync("ObjectName")` and checks whether the binding was successful using `CefSharp.IsObjectCached(ObjectName)`. This process runs at roughly 2,000 attempts per second, and you can expand it by supplying a larger wordlist.

CefEnum also supports common suffixes, which are automatically appended to each wordlist entry and tested as well. These suffixes include:

```javascript
var commonSuffix = ["Api", "Object", "Class"];
```

![object fuzz](/assets/images/6cba0ae35d27b05f23223171df494dbb588be540.png){:width="500px"}

You can use your own wordlist by specifying it at startup with the `-wordlist=./wl.txt` flag.

## Brute
You can also brute-force object names with the `brute` command [a-zA-Z] This uses the same detection method as `fuzz`, but it's not recommended as performance drops off quickly, and the process becomes impractical beyond five-character names.

## Bind
Once you've discovered an object, use the `bind` command to instruct the client to enumerate all available methods/functions via introspection:

![object bind](/assets/images/7b6b0cf85237943f18ae4f47c3073638eea327a6.png){:width="500px"}

## Run within the client
You can also conduct all these attacks from within the client/browser itself if your page is rendered within the client. 

![Testing within the client](/assets/images/8e0f05bd24bf0445c2222af229c2aa06b21f1179.png)

## Interact with the binding
Once you've identified an object and successfully enumerated its methods, you can invoke them using:

```javascript
window.objectName.MethodName("myparameter")`
```

Just replace objectName and methodName with the appropriate values. As seen for BadBrowser's `cusomObject` with the WriteFile() method:

```javascript
window.customObject.WriteFile("test.txt");
```

## Prevention
If your portal is not intended to load content from external domains, implement a strict allowlist of trusted origins. Deny any requests attempting to load resources from outside this list. (Note: This might sound like configuring the CSP, but this list must be enforced within the client’s C# code, not the web server.)

This, however, won’t prevent all attacks. For example, if the portal hosting the backend site contains XSS vulnerabilities, attackers can simply embed their attack payload into the portal itself.

Carefully review the classes you expose to the browser and ensure that only those with minimal, tightly scoped methods are bound.

If you’re unsure or would like a second opinion, book a session with us at DarkForge Labs, we’d be happy to help.

## Final Words
[CefSharp](https://github.com/cefsharp/CefSharp) is an excellent project, maintained by a great community. It remains a popular choice for enterprises developing in-house thick-clients.

If there’s a faster or more effective way to enumerate object names that we’ve missed, feel free to submit a PR. Thanks for reading!