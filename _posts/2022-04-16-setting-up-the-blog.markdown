---
layout: post
title:  Setting Up the Blog
date:   2022-04-16 16:37:20 -0700
tags: jekyll github ruby
description: "Step 1 in documentation: set up the documentation."
---


# Goals

I wanted to set up a nice-looking blog to record the documentation of the various custom systems I set up. Turns out that's harder than it sounds, though not *that* hard.

I decided to do it using [GitHub Pages](https://pages.github.com/), but using a [custom theme](https://github.com/streetturtle/jekyll-clean-dark).

# 0. Setup the repo

I started by forking [this repo](https://github.com/streetturtle/jekyll-clean-dark) and naming the repo `jgillula.github.io` per [the instructions](https://pages.github.com/).

I then followed [the instructions here](https://bitbra.in/2021/10/03/host-your-own-blog-for-free-with-custom-domain.html) to set up the site. Note that in the **Github Action** section, in the code block for [`.github/workflows/github-pages.yml`](https://github.com/jgillula/jgillula.github.io/tree/main/.github/workflows/github-pages.yml), you need to replace the `token: $` line with `token: ${% raw %}{{ secrets.CUSTOM_GITHUB_TOKEN }}{% endraw %}`.

Also, since this theme didn't automatically come with a Gemfile, I had to make one.

# 1. Customizations

I made a few edits to the site's CSS in [`assets/css/theme.scss`](https://github.com/jgillula/jgillula.github.io/tree/main/assets/css/theme.scss).

I also got pagination working, but by the time I got to write this documentation I forgot how. [`_includes/pagination.html`](https://github.com/jgillula/jgillula.github.io/tree/main/_includes/pagination.html) is a good place to start.

Originally with this theme you could enable a table of contents for a post by setting `toc: true` in the front matter section of a post. I decided I'd rather always have the TOC on the side, so in [`_includes/sidebar.html`](https://github.com/jgillula/jgillula.github.io/tree/main/_includes/sidebar.html), I added code to detect if the layout was a post or not, and show the table of contents if so.

I also wanted to automatically generate links to every header, so I changed [`_includes/header.html`](https://github.com/jgillula/jgillula.github.io/tree/main/_includes/header.html) to include a slightly edited script [from here](https://darn.es/adding-heading-links-to-your-jekyll-blog/) to automatically insert links, and again edited the css in [`assets/css/theme.scss`](https://github.com/jgillula/jgillula.github.io/tree/main/assets/css/theme.scss).



# 2. Update the repo

To add new posts, just copy an existing post in the [`_posts`](https://github.com/jgillula/jgillula.github.io/tree/main/_posts) directory, edit it, and then push to Github. The Github action will take care of updating the blog, though it may take a minute.