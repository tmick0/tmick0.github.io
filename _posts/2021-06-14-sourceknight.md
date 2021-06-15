---
layout: post
status: publish
published: true
title: Dependency and build management for sourcemod with sourceknight
author: Travis Mick
date: 2021-06-14
---
Though I haven't been playing with it for long, I've found that compiling plugins for sourcemod is a huge
hassle, mostly due to how dependencies are managed -- that is, they really aren't.

For those who don't know what sourcemod is, you probably have no interest in reading this article. But,
if for whatever reason you want to read it anyway, [sourcemod is a mod for Source engine game servers](https://www.sourcemod.net/about.php)
that provides support for custom scripts and plugins.

The Source engine is fairly old, dating back to Half-Life 2, but many games based on it such as
Team Fortress 2 and Counter-Strike: Global Offensive still have a thriving community with active
custom servers. Many of these community servers rely on sourcemod to provide various features as well as
support for custom game modes.

Unfortunately, the state of sourcemod plugin development is largely "download a bunch of files,
put them somewhere, hopefully have them organized correctly, and pray." I aimed to fix that through the
creation of the tool I'm writing about today: [sourceknight](https://github.com/tmick0/sourceknight).

<!-- more -->

The core concept of sourceknight is to allow you to specify the dependencies of your project with a simple configuration file:

```yaml
project:
  name: myplugin-example
  dependencies:
    - name: sourcemod
      type: tar
      version: 1.10.0-git6503
      location: https://sm.alliedmods.net/smdrop/1.10/sourcemod-1.10.0-git6503-linux.tar.gz
      unpack:
      - source: /addons
        dest: /addons
  root: /myplugin
  targets:
    - example
```

Then, all you have to do is type `sourceknight build`, and your plugins will magically be compiled.

Maybe you aren't writing your own plugins, but manage a server and hate the process of updating and compiling
all the plugins you use? Well, sourceknight can still help -- a sourceknight project doesn't need to contain
any scripts of its own, and can also just be used to compile plugins you specify as dependencies.

I did find a few other projects which alleged to help manage sourcemod based projects, however I believe
sourceknight, at least in vision, provides a much more streamlined experience. As of today, sourceknight
is mostly a proof of concept -- however, it is certainly capable of managing dependencies and compiling
plugins. I'm writing this post mostly in order to measure interest, find deficiencies, and identify what
features are desired moving forward. But nonetheless, hopefully someone will find it useful.

If you missed the link above, the GitHub repository is here:
[https://github.com/tmick0/sourceknight](https://github.com/tmick0/sourceknight)
