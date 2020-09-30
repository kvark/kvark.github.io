---
layout: post
title:  "Platform wars"
date:   2020-09-30 11:11:11 -0500
categories: web platform
---

Back in [Defining the Web Platform]({% post_url 2016-03-21-web-platform %}) I elaborated on the future directions of WebGPU API. Later I realized how wrong I was, and came to understanding of the Web platform a lot better. Let's now talk about the place of the Web in a wider space.

## Sunset of the Native

Today we can observe a clear separation between different personal computing platforms:
  - Microsoft platform based on Windows OS, DirectX graphics, and more, working with a wide range of hardware.
  - Apple platform based on Darwin OS, Metal graphics, and Core APIs. It has a strong top-down integration, and runs on a limited range of hardware.
  - Linux (or GNU/Linux?) platform is the most open and the least... defined as a platform. There is a bunch of distributions, a number of different package repositories or application stores, lots of protocols, etc. It's an anarchy of a platform, and that's probably why we are still waiting for the year of Linux desktop.

Each of these platforms sit directly on top of hardware. They feature ways to discover and run applications. They expose a number of system APIs to allow developers to get access to that hardware. This artificial environment constitutes a digital platform.

For a long while, if you needed to do any productive activity, you had to find and run the specific application to do so. For example, editing documents was done in MS Office or OpenOffice. For programming you'd use MS Visual Studio or Geany, and so on. For gaming, you'd install games and run them. Good applications, generally, were either good, or portable, but rarely both. For examples, Java apps were portable, but they were neither pretty or fast. Qt applications were fast, but cross-platform UI never reached a level of usability that native UI had.

## Sunrise of the Web

One of these applications was a browser, which was somewhat similar to a document viewer, but for remote addresses (URLs). We taught browsers to execute general-purpose code - JavaScript, which is where the new platform was born. Applications were documents on the Internet, and the APIs allowed basic interaction with the page and the browser. Portability story was pretty good, given the collaboration between browser vendors on standards.

Quickly, developers became comfortable in this Web-by environment, and started to want more. They demanded access to audio, video, canvas, local storage, database, geolocation, payment, and more! Browsers borrowed OpenGL ES for the Web to run fancy hardware-accelerated graphics. In 25 years since JavaScript was born, the platform's roots grew wide and far, reaching much of the capability previously thought to be native only.

MS Office suite is now online, trying to compete with Google Docs. Nobody is editing documents in applications any more. Developers massively migrate to Visual Studio Code, which is a Web application in disguise. Games are perhaps the last bastion of native platforms (since WebGL isn't powerful enough), and that is being challenged by emerging new WebGPU standard.

## Fight for users

Platform holders want applications and users. This allows them to sell products and services. Platform is their space to breath and grow. If everybody just kicks off a browser after booting a PC and never leaves the Web, why would they need a native platform? They don't, and that's the basis for ChromeOS and FirefoxOS development. It's a liberation from the native platforms. The fact that there is still a Linux-based OS running behind a browser changes nothing: it's not exposed, not a part of the ecosystem, and can easily be replaced, which is what Fuchsia will do at some point.

Platforms fight for users. For you! The Web is one of the platforms, quite different from the others.

All this rhetoric about open standards and collaboration, which is often mentioned when accusing Apple or Microsoft, is distracting from the main idea: that Web is a competitor. Strategic decisions are made with respect to that stance. Avoiding some of the Web APIs, for example, may skew developers towards using more native APIs, such as Metal Framework on iOS.

Interestingly, the native platform holders are active in collaborating on Web standards, which may sound like a conflict of interest. They want the Web surfing experience to be great for the users, but they'd not want the Web as a platform to compete with native APIs they expose. There is also a little bit of "keep your friends close; keep your enemies closer".

## Trajectory

Ok, so the Web is growing to be more capable, every day. What does this path lead us to? The world of portable and secure applications, separated from the hardware by numerous layers of abstraction and virtualization. The work has been ongoing in the browser space to optimize the JS engines, to introduce WASM bytecode, and lower the overhead. But it's nowhere near native applications, and unlikely ever be. My first computers back in 90x were in some ways snappier than the current machinery burdened by these layers. (stopping myself to elaborate further)

And funnily, the Web applications are not even consistent in terms of visual styles and controls placement! This is a step back from native UIs, where all applications across the system are written with the same guidelenes. They look and feel the same, in a good sense.

The active growth of the Web is also its curse. There is no deprecation on the Web. APIs are supported almost forever, and are just piling up on top of each other. We've seen a similar development in such things as OpenGL and C++, both with high focus on backwards compatibility. The former got bypassed by DirectX, which quickly shaped up into a beautiful API with D3D10, thanks to the ability to break things. C++ also got bypassed... but I'd rather not sidetrack here :)

Having worked on a browser engine for a while, I see the implementation complexity being orders of magnitude higher than what you'd expect as a user. Old standards, new standards, lots of catchy edge-cases, and existing content - all contributing to that. The costs are raising faster than the search-based income from the Web.

## Alternative

Why can't we have ~~good things~~ these properties, namely portability and security, in native applications?
Imagine if Microsoft, Apple, and Linux Foundation came together to develop truly cross-platform *native* APIs? Crazy! Snappy and pretty applications, which work on all platforms, based on well-tested open APIs? Who needs that?..

That last question has a caveat: the answer is both obvious and not. Short answer is: we, consumers, direly need that. Long answer: nobody is willing to *pay* for that. There is no money in native, no advertising or brainwashing, unlike the Web space. The needs of a market went into an opposite direction from the needs of consumers.

Fortunately, not all is lost. There is a number of technologies that show promise. One of them is [WASI](https://wasi.dev/). It's technically not a native API, I see it as somewhere in the middle between the native and the Web. Perhaps, it's the best of both worlds?

Another idea is - more initiatives like [Vulkan Portability](https://www.khronos.org/blog/fighting-fragmentation-vulkan-portability-extension-released-implementations-shipping) and [WebGPU on native]({% post_url 2020-05-03-point-of-webgpu-native %}), which bridge the gap between platforms, supported by proper specification and tests. If we have a critical mass of high-quality cross-platform libraries, perhaps we'd see more companies willing to choose this path instead of Electron.

P.S. this is a personal opinion, and doesn't reflect the views of my employer
