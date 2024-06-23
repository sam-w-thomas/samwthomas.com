---
title: Content website validation with Vale
date: 2024-06-17T16:30:00+01:00
slug: content-website-vale-validation
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

# 1.0 Introduction

[samwthomas.com](https://samwthomas.com) is hosted using an application called [Hugo](https://gohugo.io/getting-started/quick-start/), this is an application which converts content files, i.e. Markdown, into a collection of (useable) HTML and CSS files; with a selection of themes available. The benefit of this is that new content can be added easily, and managed through version control.
However, many spelling and grammar checks are Markdown unaware.

[Vale](https://vale.sh/) enables you to bring "linting" to Markdown-enabled documentation and integrate it into a pipeline. For example, if you manage a documentation platform, you could reject any pull request with spelling errors *and*, via Hugo, deploy your documentation site with no user involvement.

Vale delivers many benefits, these include:
* Spelling and grammar checks
* Prose checks
* Enforce styling guides (for example, if I dare use this example, instead of "VMWare", you enforce all writers to use "VMWare by Broadcom") 

The combination of combing Hugo *with* Vale is something I think which offers powerful benefits with minimal cost, and forms the content of this post.
<br>
<br>
<br>

## 2. Setting up Hugo
Install Hugo -  [https://gohugo.io/installation/](https://gohugo.io/installation/)

Create your website directory, 
`hugo new site {project_name}`

Save yourself the hassle and turn the (empty) Hugo structure into a GitHub repository,  
`git init`

Install the base of a theme you'd like to use, I used [https://github.com/colorchestra/smol](https://github.com/colorchestra/smol).
Themes are installed into the themes folder in your website structure:  
![Hugo Themes](/hugo_themes.PNG)

I wanted to customise my theme, here is when my first problem arose; Having a git repository within another get repository. There's a few different options, I'd recommend [browsing the Hugo forum for inspiration](https://discourse.gohugo.io/t/adding-a-theme-as-a-submodule-or-clone/8789). 
I'd had limited experience with [Git Submodules](https://www.git-scm.com/book/en/v2/Git-Tools-Submodules) but decided to give that a go. Unfortunately, I just could not get it working with GitHub so that Cloudflare Pages (more on that later) would also pull in the themed repository into the build. Therefore, I just added the theme as a set of files not associated with a GitHub repository - dirty, yes, quick, yes.

Now, to add content to your website. Simply add .md files to the "content" folder in the root of your directory; these are then converted into pages when the website is built. Examples of content .md pages can be found on [this websites GitHub repository](https://samwthomas.com).
![Hugo Content](/hugo_content.PNG)  
A few points on the specifics of content:
* Take note on the `draft: false` option. If this is `true` pages shall only show if Hugo runs in development mode (`hugo server -D`)
* Categories and Tags dynamically build pages containing all pages which have those categories and tags, this is one of the benefits of a static website generator
* Setting `<base href="{{ .Site.BaseURL }}">` appeared to be required to enable images to be easily added via `![Basic graph](/graph.PNG)`. Where `graph.png` is located in the `static` folder in Hugo's root folder. 
![Hugo Content](/hugo_content_md.PNG)

Once you've added content, now time to customise the Hugo `toml` file.
* To add custom menu items, use the `[menu]` option, as shown [this websites GitHub repository](https://samwthomas.com)
* Some themes have custom parameters, for example this website has a `subtitle` parameter which sets the "Technical engineering, plus anything which interests me" message
![Hugo Content](/hugo_toml.PNG)

Once you've selected your theme, added content, and modified your TOML, you can now see how it looks:
`hugo server [-D]`   (Adding -D includes draft files)

If you're happy with how it looks, build the website into static HTML/CSS files:  
`hugo`

The website is now created and located in the `public` folder. 

<br><br>

### 2.1 Customising Hugo Layouts
This is for anyone wanting to add a little extra flare to your page.

Themes are (essentially) a collection of partials; you can edit any of these to change sections of the rendered website.
Alternatively, you can also edit the root `layout\partials` partials. 
Note - a sites root layout files will override any theme layouts. For example, if a custom `head` partial exists - this will override all theme partials of the same type. 

I'd recommend reading Hugo's official website of [template partials](https://gohugo.io/templates/partials/). 

<br><br><br>

## 3. Integrating Hugo with Cloudflare Pages
If you followed the steps above, you should already have your site on GitHub.

Cloudflare Pages is a *free service* which shall build the website on every push/pull request (your choice).  
  
Hugo already has a [good guide](https://gohugo.io/hosting-and-deployment/hosting-on-cloudflare-pages/) on this aspect, I'd recommend following this.

The only caveat, make note of the [version problem discussed here](https://community.cloudflare.com/t/hugo-site-does-not-build-correctly/441168). This impacted my initial build.

<br><br><br>

## 4. Setting up Vale
Now, onto the interesting parts; adding Markdown prose validation with Vale. 

First, I'd recommend [installing Vale](https://vale.sh/docs/vale-cli/installation/) locally. It enables you to use the `vale {markdown_folders}` to test locally. 

Once you've installed locally you need to setup two things:
* `.vale` file
* `.vale` folder

These are setup in the *same directory* as your Hugo root directory:
![Types of Vale files](/hugo_vale_files.PNG)  <br>

Your .vale files provides several options, key examples of these are:
```
StylesPath = .vale/styles
MinAlertLevel = suggestion
Packages = write-good
[*]
BasedOnStyles = Vale, write-good
```

Key parts are:
* StylesPath - location of specific `writing` styles. [write-good](https://github.com/btford/write-good) is a popular, generic, style. Think of this as the configuration files of the packages below.
* MinAlertLevel - when you type `vale` what is the minimum alert to appear.
* Packages - external packages to install. These are the actual "implementation" of the styles.
* BasedOnStyles - which specific packages do you want to use when running `vale`

<br>
Your .vale folder provides a repository for the different styles. It's unlikely you'll have to go and edit a specific style initially. 

To install specific packages/styles:
1. Add package to packages (`Packages = {package}`)
2. Run `vale sync`
  
Vale PackageHub - [https://vale.sh/hub/](https://vale.sh/hub/)  

<br><br><br>

## 5. Adding Vale to GitHub Actions
Now, you've setup Vale, tested locally, and are ready to integrate into GitHub Actions.

GitHub Actions enable a _basic_ form of pipelines. That is, actions triggered when you make an action - such as a Pull Request or a Push. [Jenkins](https://www.jenkins.io/) or [JetBrains Team city](https://www.jetbrains.com/teamcity/) are examples of other pipeline tools.  
  
Once your site is setup on GitHub, adding a workflow is as simple as adding a `.github/workflows` folder in the Hugo root directory -  Following these steps below:
1. Determine how you want to validate your content. Do you want to validate content categories separately? If so, create a separate workflow for each. Below demonstrates this.
2. Create `{action_name}.yml` file in `.github/workflows`
![GitHub Actions workflows](/vale_seperate_categories.PNG)  
3. Populate the action file. Below gives guidance. 
![GitHub Action Content](/vale_seperate_categories.PNG)  
4. Add, Commit and Push your content to GitHub. I push to main (currently) so I require no additional pull-requests following this.

Now, looking at the actions tag in GitHub, you should see each action being run. In the case of samwthomas.com - we split out Posts/Books so there will be two actions per push. Clicking a workflow gives further information on what triggered the success/fail. `ReviewDog` provides the granular level of detail:
![Reviewdog](/reviewdog.PNG) 

<br><br>

### 5.1 Adding a Workflow Status badge to your site
Want this?
![GitHubActionsDemo](https://docs.github.com/assets/cb-16218/mw-1440/images/help/repository/actions-workflow-status-badge.webp)  

Follow this [https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/adding-a-workflow-status-badge](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/adding-a-workflow-status-badge)

You can add this to your actual Hugo site by modifying the themes partials.

For example, I modified `samwthomas.com\themes\smol\layouts\partials\header.html` in for this site.  

<br><br><br>

## 6. Improvements
* Currently, whenever I push to main, it updates the website itself. You can setup CloudflarePages and GitHub actions to use separate branches. Therefore, I plan to create a `dev.samwthomas.com` site; and push directly to this. Then, once I am ready to publish - pull request from the dev to main branch. 
* Integration with other GitHub actions, such as [upptime](https://github.com/upptime/upptime), to improve the sites functionality

<br><br><br>

## 7. Limitations
* I found it frustrating that I couldn't create a workflow status for individual pages. However, it would require a separate action for each page - which, from what I could see, is beyond the simplicity of GitHub actions at scale. Implementing a pipeline tool with greater depth/customisation may achieve this result.
* Looking at it from a documentation perspective, sometimes documentation isn't always that "hierarchical" layout which a tool like Hugo presents. You may have documentation relevant to all teams, some sub-sets of teams etc. Demonstrating that level of complexity in a tool like Hugo would be challenging, and ultimately futile, I believe. 
* Whilst writing blogs in Markdown has many benefits. Sometimes the formatting "prettiness" requires a bit of work. 

<br><br><br>

## 8. Use Cases
* Technical, validated, documentation for commercial products and services. For example, a team of 5 technical writers may benefit from enforcement of prose which is user defined. Furthermore, the version control aspect of GitHub enables industry-standard reviews
* Technical, validated, documentation for personal products and services. For example, I create a commercialised business idea and to spin up a documentation platform with minimum effort and cost.
* Quick, and easy, site hosting for personal blogs. Previously, when I hosted personal websites, there was maintenance involved. Now, I could leave [samwthomas.com](https://samwthomas.com) for 5 years and it would still be there