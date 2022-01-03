---
title: "Publishing Subdirectory to Github Pages"
date: 2020-03-12T22:14:58+08:00
draft: false
tags:
  - github
---

I'm using Hugo + Github Pages as my personal blog platform. A Hugo site yields the following directory structure, where the `public/` subdirectory stores the generated static pages:

```
├── archetypes/
├── config.toml
├── content/
├── data/
├── layouts/
├── public/
├── resources/
├── static/
└── themes/
```

How do I publish the `public/` subdirectory, instead of the root directory, to Github Pages?

<!--more-->

We need two branches to achieve this. Assuming your default publishing source is the `gh-pages` branch (check out [this link](https://help.github.com/en/github/working-with-github-pages/about-github-pages#publishing-sources-for-github-pages-sites) for more information about default publishing source), we commit the entire Hugo site to the `master` branch, and commit only the `public/` subdirectory to the `gh-pages` branch. Here are the steps:

1. Commit `public/` to `master` branch. You need first make sure it isn't ignored by Git.
2. Push only `public/` to `gh-pages` branch: `git subtree push --prefix public origin gh-pages`.

This should do the trick.
