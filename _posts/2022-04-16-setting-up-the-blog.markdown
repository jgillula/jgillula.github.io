---
layout: post
title:  Setting Up the Blog
date:   2022-04-16 16:37:20 -0700
tags: setup jekyll github ruby
description: "Step 1 in documentation: set up the documentation."
---


# Goals

I wanted to set up a nice-looking blog to record the documentation of the various custom systems I set up. Turns out that's harder than it sounds, though not *that* hard.

I decided to do it using [GitHub Pages](https://pages.github.com/), but using a [custom theme](https://github.com/streetturtle/jekyll-clean-dark).

# 0. Setup the repo

Per [the instructions](https://pages.github.com/), I created a new repo named ```jgillula.github.io```.

I then followed [the instructions here](https://bitbra.in/2021/10/03/host-your-own-blog-for-free-with-custom-domain.html) to set up the site. Note that in the **Github Action** section, in the code block for ```.github/workflows/github-pages.yml```, you need to replace the ```token: $``` line with ```token: ${% raw %}{{ secrets.CUSTOM_GITHUB_TOKEN }}{% endraw %}```.

# 1. Update the repo

To add new posts, just copy an existing post in the ```_posts``` directory, edit it, and then push to Github. The Github action will take care of updating the blog, though it may take ~10 minutes.