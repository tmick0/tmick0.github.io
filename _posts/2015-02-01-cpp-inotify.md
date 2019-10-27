---
layout: post
status: publish
published: true
title: A C++ encapsulation of the Linux inotify API
author: Travis Mick
date: 2015-02-01
---
The inotify API allows you to monitor a file or directory for various events such as file creation, modification, and deletion. It is part of the Linux kernel and the glibc userspace library, however its C API can be cumbersome to use in a C++ application. A [C++ binding of inotify does exist](http://inotify.aiken.cz/?section=inotify-cxx&page=doc&lang=en), but it still requires the application developer to write an unsightly wait-and-handle loop. My goal for this project was to create an asynchronous event-driven API through which filesystem events can be processed.

<!-- more -->

My inotify encapsulation, [Inotify\_cpp](https://github.com/tmick0/Inotify_cpp), hides inotify's read loop behind a simple system of event callbacks. A programmer who seeks to monitor the filesystem only needs to write an event callback (which can be as simple or as complicated as desired) and attach it to a watch. It will then receive events as they occur, all while allowing the main application thread to attend to more important matters. An [example of the library's ease of use](https://github.com/tmick0/Inotify_cpp/blob/master/app/main.cpp) is included.

Currently, Inotify\_cpp implements all the functionality of the underlying inotify API, only with the exception of cookie handling for move events. However, this will be implemented soon, and I even plan to extend my library beyond the capabilities of traditional inotify. Most importantly, I will add support for recursive watching -- one of the most sought-after features inotify lacks.

Inotify\_cpp is hosted on Github. You can clone or fork it here:

[https://github.com/tmick0/Inotify\_cpp](https://github.com/tmick0/Inotify_cpp)
