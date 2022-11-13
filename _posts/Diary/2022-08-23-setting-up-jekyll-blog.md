---
layout: posts

# title: "Setting up Jekyll Blog"
# excerpt: "excerpt"

categories: diary
# tags: diary

date: 2022-08-23 23:20:00 +09:00
last_modified_at: 2022-08-23 23:20:00 +09:00

# published: true

# read https://apit.dev/jekyll/minimal-mistakes-side-bar/
# https://mmistakes.github.io/minimal-mistakes/docs/layouts/#custom-sidebar-navigation-menu


---

## Setting up Jekyll Blog

### Installation and Initialization

I created a `paulsohn.github.io` repo which remote is on github.
Necessary installations are:

```
$ sudo apt install ruby
$ sudo apt install gem
$ gem install jekyll bundler
```

It takes some time. After that, initialize jekyll blog.

```
$ jekyll new --skip-bundle .
```

Once they're done, I deployed it on my local machine.
```
$ bundle install
$ bundle exec jekyll serve
```

Publishing is easy - just git push it and github will do the rest of the work for you.

### Configurations
I decided to apply [minimal mistakes](https://mmistakes.github.io/minimal-mistakes/) theme on my blog. In `_config.yml`, comment out `theme` and add below:

```
remote_theme: "mmistakes/minimal-mistakes@4.24.0"
plugins:
  - jekyll-feed
  - jekyll-remote-theme
  - jekyll-include-cache
```

I wanted pages to see posts category-wise and tag-wise, so I added this:
```
category_archive:
  type: liquid
  path: /categories/
tag_archive:
  type: liquid
  path: /tags/
```
and this is my `category-archive.md`. `tag-archive.md` is similar.
```
---
title: "Posts by Category"
layout: categories
permalink: /categories/
author_profile: true
---
```

## References
* https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll
* https://theorydb.github.io/envops/2019/05/03/envops-blog-github-pages-jekyll/
* https://theorydb.github.io/envops/2019/05/04/envops-blog-posting-prose-io/