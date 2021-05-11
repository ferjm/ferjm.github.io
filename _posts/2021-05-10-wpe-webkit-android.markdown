---
layout:     post
title:      "WPE WebKit for Android"
date:       2021-05-10
summary:    Introducing WPE WebKit for Android
categories: ["wpe", "webkit", "android"]
author: "Fernando Jiménez"
---

WPE WebKit is the official WebKit port for embedded and low-consumption computer devices. It has been designed from the ground-up with performance, small footprint, accelerated content rendering, and simplicity of deployment in mind.

It brings the excellence of the WebKit engine to countless platforms and target devices, serving as a base for systems and environments that primarily or completely rely on web platform technologies to build their interfaces.

WPE WebKit’s architecture allows for inclusion in a variety of use cases and applications. It can be custom embedded into an existing application, or it can run as a standalone web runtime under a variety of presentation systems, from platform-specific display managers to existing window management protocols like Wayland or X11.

Today, we ([Igalia](https://igalia.com)) are happy to announce initial support of WPE for Android.

This effort was initiated back in 2017 by my colleague [Žan Doberšek](https://www.igalia.com/igalian/zdobersek), who fully implemented a [WPE backend](https://github.com/Igalia/WPEBackend-android) for Android along with the required pieces to get rendering and basic input work. The work was paused for quite some time until the beginning of this year, when I joined Igalia and took over his work. Since then, I have been heads down working on it, trying to make it more usable thanks to [Cerbero](https://github.com/Igalia/cerbero/tree/wpe-android) and a [WebView](https://developer.android.com/reference/android/webkit/WebView) based Java API.

# How it looks

An image is worth a thousand words. This is how it currently looks running on an Android phone:

<div style="text-align:center;">
    <video controls src="/content/videos/2021/05/wpeandroid_may.mp4"></video>
</div>

As you can see, we have the basic set of functionality enough to implement a simple multi-tabs web browser with progress report, navigation controls and IME support.

Support is not limited to mobile devices though. Thanks to the wide range of architectures and devices that support Android
we can now run WPE WebKit on an even wider set of devices. Like a pair of XR glasses. This is a video of a port of
[Firefox Reality](https://mixedreality.mozilla.org/firefox-reality/) using [WPEView](https://github.com/Igalia/wpe-android#wpeview-api)
instead of [GeckoView](https://mozilla.github.io/geckoview/):

<div style="text-align:center;">
    <video controls src="/content/videos/2021/05/wpeandroid_fxa.mp4"></video>
</div>


## Building blocks

# Cerbero build system

WPE WebKit has a very long list of dependencies. Cross compiling all these dependencies manually can be quite cumbersome,
so in order to ease the development process I focused my first weeks of work on setting up a more usable build system.
We decided to use [Cerbero](https://github.com/Igalia/cerbero/tree/wpe-android), GStreamer's cross compilation system,
which already had *recipes* - this is how Cerbero
names its build scripts - for many of the required dependencies. I wrote all the missing Cerbero recipes and integrated it
into WPE Android's build system, to the point that building everything requires a single `python3 scripts/bootstrap.py --build`
command.

For now the only supported architecture is arm64. There are plans to support other architectures soon.

# WPEView API

WPEView wraps the WPE WebKit browser engine in a reusable Android API.
WPEView serves a similar purpose to Android's built-in WebView and tries to mimick
its API aiming to be an easy to use drop-in replacement with extended functionality.

Setting up WPEView in your Android application is fairly simple.

First, add the WPEView widget to your [Activity layout](https://developer.android.com/training/basics/firstapp/building-ui)

```xml
<com.wpe.wpeview.WPEView
        android:id="@+id/wpe_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".MainActivity"/>
```

And next, wire it in your Activity implementation to start using the API, for example, to load an URL:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)

    var browser = findViewById(R.id.wpe_view)
    browser?.loadUrl(INITIAL_URL)
}
```

To get a better sense on how to use WPEView, check the code of the *MiniBrowser* demo in the
[examples](https://github.com/Igalia/wpe-android/tree/main/examples/minibrowser) folder.

# Process model

In order to safeguard the rest of the system and to allow the application to remain responsive even if
the user loads a web page that infinite loops or otherwise hangs,
the modern incarnation of WebKit uses a multi-process architecture.
Web pages are loaded in its own WebProcess.
Multiple WebProcesses can share a browsing session, which lives in a shared NetworkProcess.
In addition to handling all network accesses, this process is
also responsible for managing the disk cache and Web APIs that allow websites to store
structured data such as Web Storage API and IndexedDB API.

Given that Android forbids the fork syscall on non-rooted devices, we cannot directly spawn child processes.
Instead, we use Android Services to host the logic of WebKit's auxiliary processes.
The life cycle of all WebKit's auxiliary processes is managed by WebKit itself.
The Android layer only proxies requests to spawn and terminate these processes/services.

In addition to the multi-process architecture, modern WebKit versions introduce the PSON model
(Process Swap On Navigation) which aims to improve security by creating an independent WebProcess
for each security origin. This is currently disabled for WPE Android, although partial support is already in place.

# Browser and Pages

The central piece of WPE Android is the [Browser](https://github.com/Igalia/wpe-android/blob/main/wpe/src/main/java/com/wpe/wpe/Browser.java)
top level Singleton object. This is somehow the equivalent
to WebKit's UIProcess. Among other duties it:
* Manages the creation and destruction of `Page` instances.
* Funnels `WPEView` API calls to the appropriate `Page` instance.
* Manages the Android Services equivalent to WebKit's auxiliary processes (Web and Network processes).
* Hosts the UIProcess thread where the [WebKitWebContext](https://wpewebkit.org/reference/wpewebkit/2.23.90/WebKitWebContext.html)
instance lives and where the main loop is run.

A [Page](https://github.com/Igalia/wpe-android/blob/main/wpe/src/main/java/com/wpe/wpe/Page.java)
roughly corresponds to a tab in a regular browser UI.
There is a 1:1 relationship between WPEView and Page.
Each Page instance has its own [gfx.View](https://github.com/Igalia/wpe-android/blob/main/wpe/src/main/java/com/wpe/wpe/gfx/View.java)
and [WebKitWebView](https://wpewebkit.org/reference/wpewebkit/2.23.90/WebKitWebView.html) instances associated.

# WPE Backend

The common interface between WPEWebKit and its rendering backends is provided by [libwpe](https://github.com/WebPlatformForEmbedded/libwpe).
[WPEBackend-android](https://github.com/Igalia/WPEBackend-android) is our Android-oriented implementation of the libwpe API, bridging the gap between the WebKit architecture and
the internal composition structure on one side and the Android system on the other.

# gfx.View

[gfx.View](https://github.com/Igalia/wpe-android/blob/main/wpe/src/main/java/com/wpe/wpe/gfx/View.java)
is an extension of [android.opengl.GLSurfaceView](https://developer.android.com/reference/android/opengl/GLSurfaceView?hl=en) living
in the UI Process. It manages the life cycle of a [Surface Texture](https://developer.android.com/reference/android/graphics/SurfaceTexture),
which is some sort of buffer consumer, that is handed off to the Web Process
through Android's IPC mechanisms, where the actual rendering happens.

It is also in charge of relaying input events to the internal WebKit input-methods.

This part is currently being significantly changed by Žan to use
[Native Hardware Buffers](https://developer.android.com/ndk/reference/group/a-hardware-buffer).

## Future work

There are still plenty of things to do and, we have a growing [list of issues](https://github.com/Igalia/wpe-android/issues) in the main repository.
The next steps will be towards extending support for other architectures - so far only arm64 is supported.
Multimedia support is also on the list of immediate plans. Along with the big rendering engine refactor that
Žan is working on.

## Try it yourself

If you want to try the current prototype, you can follow the instructions in the
[README](https://github.com/Igalia/wpe-android/blob/main/README.md#setting-up-your-environment)
of the main repo.

We welcome contributions of all kinds. Give it a try and [file issues](https://github.com/Igalia/wpe-android/issues/new) as you encounter them.
And if you feel encouraged enough, send us patches!

## Acknoledgements

* I would like to thank [Igalia](https://igalia.com) for giving me the time and space to work on this project.
* Huge thanks to [Žan Doberšek](https://www.igalia.com/igalian/zdobersek) for his amazing work and continuous guidance.
* Kudos to [Philippe Normand](https://www.igalia.com/igalian/pnormand) and [Thibault Saunier](https://www.igalia.com/igalian/tsaunier) for their recommendations and support around Cerbero.
* Many thanks to [Imanol Fernandez](https://www.igalia.com/igalian/ifernandez) for his contributions so far and the XR demo.

