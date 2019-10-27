---
layout: post
status: publish
published: true
title: Information-centric networking for laymen.
author: Travis Mick
date: 2016-10-08
---

The design of the current Internet is based on the concept of connections between "hosts", or individual computers. For example, when you visit a website, your computer (a host) always connects to a particular server (another host) and retrieves content through a session-oriented pipe. However, the amount of content hosted on the Internet and the number of connected devices are both growing. This is a crisis scenario for the current Internet architecture -- it won't scale.

<!-- more -->

Several proposals for Next-Generation Network (NGN) architectures have been proposed in recent years, aimed at better handling immense amounts of traffic and orders of magnitude more pairwise connections. Information-Centric Networking (ICN) is one NGN paradigm which eschews the concept of connections entirely, removing the host as the basic "unit" of the network and replacing it with content objects.

In other words, the defining feature of an ICN is that instead of asking the network to connect you to a particular server (where you may hope to find a content you desire), you instead ask the network for the content itself.

Several distinct ICN architectures have been proposed, however for the remainder of this article I will focus on Named-Data Networking (NDN) and Content-Centric Networking (CCN), the two most popular designs in recent literature. NDN and CCN both share the core concept of consumer-driven communication, wherein a consumer (or client) issues an Interest packet (a request) for a content object and hopes to receive a Data packet in return. Interest and Data packets are both identified by a Name, which is in essence a immutable, human-readable name for a particular content object.

Whereas current Internet routers rely on only one lookup table (i.e., a forwarding table) in order to route packets toward a destination, NDN/CCN routers use three main data structures in order to locate content objects. A Pending Interest Table (PIT) keeps track of outstanding requests, a Content Store (CS) caches content objects, and a Forwarding Information Base (FIB) stores default forwarding information.

When a router receives an Interest, it will first check its CS to see if it can serve the content immediately from its cache. If it is not found there, then the router checks its PIT to see if there is an outstanding request for the same content; if there is then the request does not need to be forwarded again, since the data for the previous request can satisfy both that request and the new one. Finally, if an existing Interest is not found, the router checks the FIB for a route toward the appropriate content provider; once a route has been identified, the Interest is forwarded and the PIT is updated.

Though ICN requires routers to store more state and make more complicated forwarding decisions, it is still expected to reduce the overall network load by virtue of Interest aggregation and content caching. Caching in particular also benefits the end-user, since the availability of content nearby reduces download time. Since content downloads are independent of any particular connection, ICN also allows multi-RAT (Radio Access Technology) communication to be exploited by mobile devices, further improving the user's QoE (Quality-of-Experience).

Last week, I presented a collaborative caching scheme for NDN at ACM ICN 2016, the leading conference in the ICN domain ([slides](http://conferences2.sigcomm.org/acm-icn/2016/slides/Session3/mick.pdf), [paper](http://conferences2.sigcomm.org/acm-icn/2016/proceedings/p93-mick.pdf)) which is able to satisfy up to 20% of Interests without leaving the home ISP's network. Additionally, we published an article in IEEE Communications Magazine about the advantages of ICN for mobile networks ([paper](/papers/COMG_20160901_Sep_2016.pdf)). These works, as well as those of the larger ICN community, have the potential to influence the acceptance of ICN as the foundation of the future Internet. Only with continued research will we find a holistic solution for scalability in the face of billions of connected devices and billions of terabytes of traffic.

