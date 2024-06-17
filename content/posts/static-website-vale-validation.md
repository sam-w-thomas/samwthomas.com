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
Install Hugo -  [https://gohugo.io/installation/](https://gohugo.io/installation/)

Create your website directory, 
`hugo new site {project_name}`

Save yourself the hastle and turn the hugo (empty) structure into a GitHub repostiory,  
`git init`

Install the base of a theme you'd like to use, I used (https://github.com/colorchestra/smol)[https://github.com/colorchestra/smol]
Themes are installed into the themes folder in your website structure:  
![Hugo Themes](/hugo_themes.PNG)

I wanted to customise my theme, here is when my first problem arose. Having a git repo within another get repo. There's a few different options, I'd recommend [browsing the Hugo forum for inspiration](https://discourse.gohugo.io/t/adding-a-theme-as-a-submodule-or-clone/8789). 
I'd had limited experience with (Git Submodules)[https://www.git-scm.com/book/en/v2/Git-Tools-Submodules] but decided to give that a go. Unfortunately, I just could not get it working with GitHub so that Cloudflare Pages (more on that later) would also pull in the themed repository into the build as well. So, I just added the theme as a set of files not associated with a GitHub repository. 

Now, to add content to your website. Simply add .md files to the "content" folder in the root of your directory; these are then converted into pages when the website is built. Examples of content .md pages can be found on [this websites GitHub repository](https://samwthomas.com).
![Hugo Content](/hugo_content.PNG)  
A few points on the specifics of content:
* Take note on the `draft: false` option. If this is `true` pages shall only show if hugo runs in development mode (`hugo server -D`)
* Categories and Tags dynamically build pages containing all pages which have those categories and tags, this is one of the benefits of a static website generator
* Setting `<base href="{{ .Site.BaseURL }}">` appeared to be required to enable images to be easily added via `![Basic graph](/graph.PNG)`. Where `graph.png` is located in the `static` folder in Hugos root folder. 
![Hugo Content](/hugo_content_md.PNG)

Once you've added content, now time to customise the Hugo `toml` file.
* To add custom menu items, use the `[menu]` option, as shown [this websites GitHub repository](https://samwthomas.com)
* Some themes have custom paramters, for example this website has a `subtitle` paramter which sets the "Technical engineering, plus anything which interests me" message
![Hugo Content](/hugo_toml.PNG)

Once you've selected your theme, added content, and modified your TOML, you can now see how it looks:
`hugo server [-D]`   (Adding -D includes draft files)

If you're happy with how it looks, build the website into static html files:  
`hugo`

The website is now created and located in the `public` folder. 

### Customising Hugo Layouts
This is for anyone wanting to add a little extra flare to your page.

Themes are (essentially) a collection of partials; you can edit any of these to change sections of the rendered website.
Alternatively, you can also edit the root `layout\partials` html partials. 
Note - a sites layout files will override any theme layouts. For example, if a custom `head` partial exists - this will override all theme partials of the same type. 

I'd recommend reading Hugo's official website of [template partials](https://gohugo.io/templates/partials/) to get more knowledge. 

## Integrating Hugo with Cloudflare Pages
If you followed the steps above, you should already have your site on GitHub.

Cloudflare Pages is a *free service* which shall build the website on every push/pull request (your choice).  
  
Hugo already has a [good guide](https://gohugo.io/hosting-and-deployment/hosting-on-cloudflare-pages/) on this aspect, I'd recommend following this.

The only caveat, make note of the [version problem discussed here](https://community.cloudflare.com/t/hugo-site-does-not-build-correctly/441168). This impacted my initial build.
  
## Setting up Vale
Now, onto the interesting parts.  


## Adding Vale to GitHub Actions

## Limitations

