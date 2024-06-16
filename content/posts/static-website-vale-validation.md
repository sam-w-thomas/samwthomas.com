---
title: "Static website validation with Vale"
date: 2024-06-17T16:30:00+01:00
slug: static-website-vale-validation
type: posts
draft: false
categories:
  - automation
tags:
  - automation
  - documentation
  - pipelines
  - github
  - github_actions
  - vale
  - linting
---
<base href="{{ .Site.BaseURL }}">

# Introduction

[samwthomas.com](https://samwthomas.com) is hosted using an application called [Hugo](https://gohugo.io/getting-started/quick-start/), this is a "static website" application which converts basic files, into a richer onlne version. For example, in our case it converts a series of Markdown documents in a folder, into a website with pages, navigation and styling. 
The benefit of this is that new content can be added easily, and can be managed through established version control procedures. 
However, many spelling and grammer checks are not aware of markdown; furthermore, version control systems are often integrated into pipelines, but automated documentation review (spelling and grammer, styling, use of specific langauge) is something I rarely see in these documentation.

[Vale](https://vale.sh/) enables you to bring programmtatic "linting" to markdown and integrate it into a pipeline. For example, if you manage a documentation platform, you could reject any pull request with spelling errors.

Vale delivers many benefits, these include:
* Spelling and grammer checks
* Prose checks
* Enforce styling guides (for example, if I dare use this example, instead of "VMWare", you enforce all writers to use "VMWare by Broadcom") 

The combination of combing Hugo *with* Vale is something I think which offers powerful benefits with minimal cost, and forms the content of this post. 


## Setting up Hugo

## Setting up Vale

## Adding Vale to GitHub Actions

## Limitations

