---
title: Jekyll, Chirpy and the Github Pages
date: 2022-04-24 12:54:00 +0600
categories: [Guides, Hosting]
tags: [guide, ruby, bundle, blog, blogging, host, hosting, jekyll, chirpy, theme, jekyll-theme, github, github-pages]
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

## Launch locally
```shell
bundle exec jekyll serve
```

## Custom domain
GitHub Pages can be configured with a custom domain. GitHub's documentations are pretty comprehensive and easy to follow, but let's go over some of the key steps that GitHub will make you perform in order to configure a custom domain.

1. **Verify domain**: GitHub will want to know that you really own the custom domain that you claim to own. This is accomplished by GH providing you with a `TXT` record details that you must create on your DNS provider.
2. **Add domain to repo in GH**: On your GH repository page, go to "Settings" > "Code and automation" > "Pages" > "Custom domain", type your custom domain, then click Save. Since I was publishing my site from a branch, this created a commit that added a `CNAME` file directly to the root of my source branch (gh-pages).
3. **Create A records on your DNS provider**: Time to create some `A` records on your DNS provider. These `A` records will point your apex custom domain (YOURDOMAIN.COM) to GitHub Pages server IP addresses. Be sure to delete any existing `A` records that might point to some other IP addresses. Check if it worked:
```shell
dig YOURDOMAIN.COM +noall +answer -t A
```
4. **Redirect subdomain**: You can redirect a subdomain like `www` by creating a `CNAME` record on your DNS provider so that the subdomain points to `<GH_USERNAME>.github.io`. Check if it worked:
```shell
dig WWW.YOURDOMAIN.COM +nostats +nocomments +nocmd
```

    > If you want to host directly at `https://YOURDOMAIN.COM`, then no need to rename the repo to `<GH_USERNAME>.github.io`. Only in `_config.yml`, set `baseurl` to `''`.
    {: .prompt-info }
