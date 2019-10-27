---
layout: post
status: publish
published: true
title: Integrating GitLab and Google Calendar.
author: Travis Mick
date: 2017-04-25
image: /images/gitlab-calendar-screenshot.png
---
Zeall, like many other software startups, uses [GitLab](https://about.gitlab.com/) for version control and issue management. We also use the ever-popular [Google Calendar](https://calendar.google.com/) to handle meetings, reminders, and deadlines. For several months, we've been looking for a way to automatically push GitLab issue deadlines into Google Calendar, and until now it seemed impossible. Only after a recent migration from our own private mailserver to [G Suite](https://gsuite.google.com/) did we find a solution -- or rather, figure out how to feasibly build one.

<!-- more -->

![visualization of a gitlab issue and corresponding calendar entry](/images/gitlab-calendar-screenshot.png)

GitLab's webhooks make it pretty easy to obtain information about new and modified issue deadlines, however the problem has always been pushing that information into Google Calendar. I had previously explored the [Google Calendar API](https://developers.google.com/google-apps/calendar/) and ultimately decided that implementing our own solution wasn't worth it, simply because authorization would be a nightmare -- there was no easy way to share a calendar with our team's members. However, once we got a G Suite domain, I found it was possible to use a domain scope for [Calendar ACLs](https://developers.google.com/google-apps/calendar/v3/reference/acl). I simply created a [service account](https://developers.google.com/identity/protocols/OAuth2ServiceAccount) for our integration, set it up to grant read access to our entire domain, and implemented some basic logic to map GitLab issues to calendar events.

As far as I can tell, I'm the first person to have successfully integrated GitLab issues into Google Calendar; however, I'm sure I'm not the first to *want to* do so. Therefore, to spare others from sharing in my misery I've decided to make the fruit of my labor [available on GitHub](https://github.com/tmick0/gitlab-calendar) for anybody to use. Try it out, and be sure to let me know if it breaks.

