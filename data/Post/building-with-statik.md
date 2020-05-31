---
title: Built With Statik
slug: built-with-statik
header: img/headers/code.png
header-alt: A computer screen with CSS
summary: This shiny new blog was built with statik. Here's the basics.
author: piper
---

It's been a wild week. On Monday I decided to make a new website. I had already
taken a look at statik as a possible tool for the job, and set out to try. Less
than 48 hours I launched.

It's not feature complete yet, but it's a really solid start.

## Why _ANOTHER_ Static Site Generator

When you get into my specialties as a developers, I get opinionated. No
surprise, we all do. So it should come as no surprise that as a dev who got my
start building static websites in HTML and CSS, I have opinions on how I work
with them.

Most static site generators are single purpose: Jekyll makes great pages and
blogs, but building any other taxonomy is annoying. Lots of other ssgs have this
problem, too. Sphinx is _great_ for docs, but I wouldn't want to build a web
page with it.

This limitation was one I'm never happy with, because I almost always want to
run multiple things off the same site.

The other thing I dislike about most static site generators is that they're
opinionated about their HTML. They have specific ideas about how you should go
about laying out your templates, and how you invoke them. It's never sat well
with me.

## I Could Build My Own

It's a running joke that there's [as many static site generators as there are
developers](https://twitter.com/cfactoid/status/1245424014669090817). I was
ready to go that route. After all, I knew a bunch of tools, it wouldn't be too
difficult to glue them together in a way I could tolerate.

But in that thread, someone mentioned [`statik`](https://getstatik.com/).

Models? Relationships? And you just point at a template in your view definition?

This was promising.

## 48 Hours Later

I went through the tutorial and got my first ugly version running inside an
hour. From there, it was a matter of working out the various bits I knew I
wanted:

* It had to be responsive.
* Needed to tell folks who I was.
* Had to have a blog
* Bonus if I could make it look good.

I dug in. Since statik supports Jinja templates, I used all that power afforded
me: used includes to build with components. An about summary, and article, an
author callout, the main header, and the main nav. Tiny, self-contained
components I could use in my pages.

My pages are your typical Jinja fare: extensions to a base.html with specialized
layouts.

All my styles are plain old CSS.

Just to make sure I'm clear here: I was building with a static site generator
more or less how I wrote my earliest websites. I got the template working with
[some dummy data](https://piper.thunstrom.dev/blog/2020/05/25/debug-post/), and
moved on to media queries. (Yes, I also like to hand roll my responsive
websites.)

This all happened over about 48 hours. It went incredibly fast, and it's been
really nice to work on one or two features a day as I go.

## The Bad Parts

This isn't to say that this process was entirely without frustration. The
errors from statik are often entirely opaque with not enough detail of what is
being processed to debug effectively. I've dug into my installed version to add
breakpoints to add that context.

The other downside is the activity on the project. The last commit was back in
October, 2019. I'm not judging for the silence, but I'd love to see some work
on the user friendliness of the project.

## The Good Parts

The tutorial only included the basic blog functionality: Throwing a page up on
the web. I added the header image and the author block below. Because it uses a
database and that database is ephemeral from build to build, making a change
like adding an avatar or header image only took about an hour. I spend way more
time finding or writing content than adding the basic functionality for a new
feature. (I then end up spending an hour testing the new feature at multiple
breakpoints, but you could probably just use your favorite CSS framework.)

The next time you're thinking about a new static site generator, maybe give
statik a look?
