---
title: Jekyll, Chirpy and the Github Pages
date: 2022-04-24 12:54:00 +0600
categories: [Blogging, Hosting]
tags: [ruby, bundle, blog, blogging, host, hosting, jekyll, chirpy, theme, jekyll-theme, github, github-pages]
---

> **Github Pages lets you host Jekyll sites and Chirpy makes it even easier.**

![Desktop View](/assets/img/bg/florian-bernhardt-CRbTJ5eFy8U-unsplash.jpeg){: w="700" h="400" }
_Photo by <a href="https://unsplash.com/@floww?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Florian Bernhardt</a> on <a href="https://unsplash.com/s/photos/ravine?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>_

## Prerequisites
Make sure to have Ruby installed.

**On Mac:**
[moncefbelyamani](https://www.moncefbelyamani.com/how-to-install-xcode-homebrew-git-rvm-ruby-on-mac/#start-here-if-you-choose-the-long-and-manual-route)'s tutorial.

**On Windows**
[https://rubyinstaller.org/](https://rubyinstaller.org/)

## Steps

1. Create a new repository from the [Chirpy Starter](https://github.com/cotes2020/chirpy-starter/generate) and name it `<GH_USERNAME>.github.io` if you want to host with Github Pages at `https://<GH_USERNAME>.github.io`.

    > If you want to host at `https://<GH_USERNAME>.github.io/<REPO_NAME>`, then just name the repo whatever you want.
    {: .prompt-info }

2. Cloned the repo locally. Installed dependencies with command `bundle` from both windows and mac.
3. Also ran:
```shell
bundle lock --add-platform x86_64-linux
```
4. In `_config.yml`{: .filepath}, 
    * updated `baseurl` to `<REPO_NAME>`
    * updated `url` to `https://<GH_USERNAME>.github.io`
5. Then pushed to `main` branch. That triggered the `/.github/workflows/pages-deploy.yml`{: .filepath} and the branch `gh-pages` on github was automatically created and site build was also successful.
6. Then went to `repository on GitHub > Settings > Pages` and as `Source`, chose `gh-pages` as the branch and `/(root)` as the root folder. Shortly afterwards, the site became accessible.
