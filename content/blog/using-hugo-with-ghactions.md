---
title: "Hugo Static Site Deployment with  Github Actions"
date: 2020-04-10T10:34:33-05:00
draft: false
description: "This documents outlines the steps involved hosting a static website built using Hugo, on GitHub and deploying using Github Actions."
tags: 
    - Github Actions
    - Hugo
    - Static Website
    - Github Pages
categories:
    - Static Website Deployment
author: Param V V
ReadingTime: 10 min
---

__Outlining the steps below to able to deploy a Hugo based Static Site to Github Pages using Github Actions__

## Repository creation

* The first repository will host the website. It needs to be named as: `<github_username>.github.io`. 
    For example, my website repository is named as `parameswaranvv.github.io`
* The second repository will host the site in development. So, it can be named in any way.
    For example, my site content repository is named as `param-site`.
    
## Site content development 

* Clone the `param-site` repository locally and within the directory, we will develop our content for the website. I used Hugo to generate my website content and I customized it to my needs.
* Clone the other repository `parameswaranvv.github.io` as a submodule within the root of the current repository. We will build our code in the current repository, and deploy it via the submodule to the other repository.

## Github Actions Pipeline script

* Within the `param-site` directory, create a `.github/workflows/main.yml` which will contain the pipeline script for building and deploying our website to Github Pages in our website repository `parameswaranvv.github.io` .
* For building the hugo site, we use the `peaceiris/actions-hugo@v2` action, and use the Hugo extended version to build. This is to enable processing of SCSS files for CSS.
* For deploying the site to the `<username>.github.io` or `parameswaranvv.github.io` in my case, use the `peaceiris/actions-gh-pages@v3` with the options turned on to deploy to an external repository. Forcing orphan ensures a clean slate deployment always.

### Sample Pipeline Script    
```yaml
# This is a basic workflow to help you get started with Actions

name: Deploy Static Site

# Controls when the action will run. Triggers the workflow on push or pull request
# events but only for the master branch
on:
  push:
    branches: [ master ]

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  deploy:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - name: Checkout Code
        uses: actions/checkout@v2
        with:
          submodules: true

      - name: Cleanup cache
        run: |
          rm -rf node_modules/gh-pages/.cache

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.64.1'
          extended: true

      - name: Build
        run: hugo -v

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.GHPAGESTOKEN }}
          external_repository: parameswaranvv/parameswaranvv.github.io
          publish_dir: ./parameswaranvv.github.io
          publish_branch: master
          force_orphan: true
          user_name: 'Param'
          user_email: 'parameswaranvv@gmail.com'
```

## Handling Github Security Tokens while deploying across repositories.

* Create a SSH Public and Private Key using your email address locally on a UNIX machine.
* Configure the SSH Private Key as a Github Secret Token named `GHPAGESTOKEN` on the `param-site` repository, where in the pipeline is hosted.
* Configure the SSH Public Key as a Deploy Key on the other repository `<username>.github.io` or in my case `parameswaranvv.github.io`.

Congratulations!! You have a static website hosted on github and with full CI/CD capabilities. In fact, you care currently reading this on my Github Pages hosted static website built on Hugo and CI/CD managed by Github Actions.
