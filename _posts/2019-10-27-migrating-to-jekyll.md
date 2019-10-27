---
layout: post
status: publish
published: true
title: Migrating to Jekyll
author: Travis Mick
date: 2019-10-27
---

The idea that my blog has been running on Wordpress for all these years has been
haunting me. In the back of my head, I've been thinking about how not only is
the CMS itself known for being full of vulnerabilities, but it's also built with
PHP, which itself isn't really the most secure. The platform also has too many
moving parts for me to be comfortable with -- maintaining a VPS with Apache,
PHP, and MySQL installations is relatively straightforward, but still has its
moments of suffering.

<!-- more -->

I decided long ago that I would eventually migrate to a static generated site;
the obvious tool for a blog following such a paradigm is
[Jekyll](https://jekyllrb.com/). Everyone seems to be using it these days, so it
was my first choice.

From the start, I identified several benefits of Jekyll over Wordpress:

* No dynamic code in production
* No need for a database
* Generation from Markdown, rather than a WYSIWYG nonsense editor or handwritten
  HTML
* The ability to version-control the entire site
* Easier customization of the theme
* Support for stuff like syntax highlighting and LaTeX without installing and
  configuring numerous plugins
* Free hosting from [Github Pages](https://pages.github.com/)
* Lower degree of platform lock-in
* Easy workflow for writing and development via the `bundle exec jekyll serve`
  incantation

I found that a [Wordpress-to-Jekyll migration script](http://import.jekyllrb.com/docs/wordpress/)
was supported, and got started. However, the process was not entirely painless.
My first endeavor was to get the Gem installed, and thus I was introduced to the
hellscape that is the Ruby package ecosystem.

Several dependencies had to be built from source, requiring me to install
numerous `-dev` packages that were identified one at a time, only after each
subsequent build had failed for a new reason.

When I thought this process was nearing completion, I was then told that the
Ruby interpreter on the system (the ancient Ubuntu installation which had been
running Wordpress) was too old to proceed. I then had to install a newer
interpreter from a PPA and rebuild all of the dependencies. 

Once I had gotten the dependencies set up, I quickly found a bug in the
migration Gem itself. Apparently, someone submitted [this patch](https://github.com/jekyll/jekyll-import/commit/41c97bd0a6c8ec98556ab2dd474c36cd6c6cbc31)
as a bugfix, however it would seem that for my particular multisite
configuration, it was incorrect. Instead of going through the pain of rolling
back to an older version of the Gem, I just decided to revert this patch by hand
in the installed `.rb` file.

At this point, I was able to successfully extract all of my posts from the
Wordpress database. However, the result was not immediately usable. Each object
from my Wordpress site had been stored as an `.html` file, copied verbatim from
the database. This meant that all embedded images and links between posts would
be broken. In addition, the author metadata placed in the front matter of each
post was not rendering correctly, and likely would not in any sane template.

This meant that I had to edit every post by hand. I had to manually download
every image in every post and alter the URLs where they were included. I had to
convert every instance of a link to another post to an incantation involving the
`post_url` macro supplied by Jekyll's templating engine, Liquid. This also did
not immediately work, due to strangeness in the way Jekyll handles generating
URLs for posts which are assigned to categories. To remedy this, I ended up
stripping the categories from all of my posts entirely.

In this manual process, I decided to bite the bullet and translate everything
from HTML to Markdown. While this was quite nice in general, reducing the syntax
overhead in my posts, I found that some of my posts had content that Markdown
could not natively represent. For example, [this one]({% post_url 2017-09-08-reverse-engineering-sysex %})
contained lists within tables, which is not part of any sane Markdown dialect.
Fortunately, Markdown gives you the freedom to keep embedded HTML around in
these circumstances.

After converting all of my content, I tested out various themes until I found
one that I liked and suited my preferences in regard to customizability. That
theme ended up being [beautiful-jekyll](https://deanattali.com/beautiful-jekyll/).

One thing that I'm not satisfied with is excerpt generation -- with Wordpress,
my post excerpts on the front page supported rich content including code blocks,
LaTeX, and images; I'm not sure whether to blame Jekyll itself, its default
configuration, or the theme I chose, but I am currently limited to plain text in
post excerpts. However, it does support specifying an `image` in the front
matter, and with a little CSS customization, I got this to do something close
enough to what I'd like. For a few posts, I also had to manually specify an
`excerpt` instead of allowing one to be generated with use of the
`<!-- more -->` tag.

My next step was to upload my site to Github to activate hosting via Github
Pages. This was relatively straightforward, simply involving forking the theme
repo and copying my content in. Of course, the next thing I wanted to do was
move it to my custom domain -- I added a CNAME file to my repository and
repointed my domain to my `.github.io` URL. After a few minutes (actually
several -- this took a pretty damn long time), everything was working, with the
exception of TLS. Allegedly, Github Pages will automatically acquire a Lets
Encrypt cert after some time (up to 24 hours?). ~~As of writing this post, it
hasn't happened yet; this downtime is less than ideal, but I suppose I can
survive.~~ *Update: immediately after publishing this, my certificate seems to
have gone live. It's nice that this process was relatively painless. Thanks,
Gitlab and Lets Encrypt!*

Overall, I expect that my life will be less painful moving forward. With a very
straightforward writing workflow (make a file, put stuff in it, `git push`) and
nothing else to worry about, I think this is something I can get used to.

