---
title: "Thoughts on Hugo"
date: 2018-05-26T20:24:56-04:00
draft: false
---

This domain has been registered for quite a while, but I didn't bother to create a website, until now.  This is some thoughts on using Hugo.

I'm quite familiar with the Javascript ecosystem (Node.JS, React, Sass/SCSS, Typescript, etc.), but wanted something simple without sacrificing too much flexibility.  I picked Hugo as the static site generator since it's a single executable that you have to download, not a Node.JS module with 6,725 dependencies to go along with it.

It's relatively nice to work with once you've gone through the documentation, but that's the rub.  The documentation looks nice at a cursory glance (and don't get me wrong, the documentation is great and thorough), but you'll quickly fall into a brain-overload of terminology that's being thrown at you every which direction.

If you stick with just making content and don't necessarily care about how the site looks, that's no problem.  Type in `hugo new blog/thoughts-of-the-day.md` and you'll be up and running in no time.  My problem, however, is I want to look under the hood and tweak everything from the theme's layout to the HTML libraries.

Searching for `hugo theme tutorial` will bring up [this page](https://gohugo.io/themes/creating/), which lays out a couple high-level concepts and how to create a new theme.  So far that's good, but how do you actually start creating the theme?  At the bottom, you'll see a link called `Directory Structure`.  That must be a good starting point!

So you go there and see some more high-level concepts about Hugo's directory structure... still nothing about themes.  Hm, what about this [layouts link](https://gohugo.io/templates/)?  Okay, they're templates.  Let's look at the [introduction](https://gohugo.io/templates/introduction/).  Okay, it uses Go's html/template package.  If you're not familiar with Go, or it's templating system, you're going to be in for a world of hurt.  Thankfully, I'm already familiar with it.

The template introduction page lays out more than what you'd want just so you can get started.  And going through all the links in the navigation bar aren't exactly helping you figure out how to get started, either.

Eventually, I just downloaded a theme and studied the source to figure out how everything fits together.  The main problem with going through Hugo's documentation is there aren't any step-by step tutorials that start with the basic, core concepts (like list pages, single pages, index pages, etc), and slowly add on more concepts from there.

That's my only criticism about Hugo's documentation.  Once you've understood the core concepts, the documentation becomes much more useful.  

The only other problem I have with Hugo is using Go's templates.  They're fine for simple tasks, but once you want to do some basic logic, they become clunky and a real eye-sore, very fast.