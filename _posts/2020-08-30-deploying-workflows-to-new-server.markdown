---
layout: post
title:  "Deploying Workflows to a New Server"
date:   2020-08-30 08:00:00 -0700
categories: workflow
comments_id: 5
---
There are a few different reasons why you may need to move a Workflow (GXWF) from one server to another. On the professional services team we frequently develop Workflows in our own development environment, migrating them to the client environment once they've been built and tested. Another common scenario is having two ArcGIS portal instances - one for development and one for production (and possibly others for test and QA). Workflows developed on one portal need to be copied up to the other when promoting to production.

Even when you have only one portal instance (on-prem Portal or AGOL) that you use for both development and production, you often need to overwrite your production Workflow with your development Workflow. This post will give you some ideas about how to do that.

The most straightforward way of copying a Workflow to another server is to export it using Workflow designer, then import it when you've connected to the target Workflow designer. This automatically takes care of updating the Workflow license and server URL (for server-side Workflows). This is almost always the best way to perform an initial Workflow migration, when there is no existing Workflow on the target server that you're using already.

What happens though when you have a Workflow already in production (or UAT even) and you just want to push some changes you've made in a dev Workflow? If you follow the export/import sequence you will end up with another copy of the Workflow in your target environment, with a new URL. To get your production (or UAT) application to use this new URL you will also need to update the application configuration, requiring more steps and causing disruption. You also need to remember to clean up the old Workflow when you're done to avoid having old copies lying around. It's much cleaner to overwrite the existing version with the new changes.

There is a [post] I did earlier about how Workflows are stored - I'd encourage you to read through that if you haven't already. This post sort of assumes you've got a decent handle on all of that.

I'll start with a simple case - a client-side Workflow with no custom activities. The steps we're going to follow are:
- Export the Workflow
- Using [ago-assistant](https://ago-assistant.esri.com), replace the item data in the target portal
- Open up the Workflow in the target designer to automatically update the license URL

### Export the Workflow
You can export the Workflow in a couple of ways. You can use the Export option in Workflow designer:
![]({{ site.baseurl }}/assets/images/ExportWorkflow.png)
or you can use ago-assistant to copy the item data instead. The two methods will give you the same result, but the ago-assistant method will format your JSON for you automatically which is nice.

The Workflow export method gives you a JSON file that you download - you will need to open that JSON file in a text editor and copy all the content to your clipboard. The ago-assistant method isn't as common but it's handy. First, launch [ago-assistant](https://ago-assistant.esri.com) and log in to your portal where the source Workflow lives.
![]({{ site.baseurl }}/assets/images/ago-assistant.png)
Then, choose `View an item's JSON` from the 'I want to...' menu (I like that 'I want to...' menu idea, wonder where they came up with that? :)
![]({{ site.baseurl }}/assets/images/ViewJson.png)
Find the Workflow in the left-hand panel and click on it. If you have lots of items you can use the search bar.
![]({{ '/assets/images/ItemDetails.png' | relative_url }})
Scroll down to the `Data` section in the right panel and click `Copy JSON`
![]({{ site.baseurl }}/assets/images/CopyJson.png)

### Replace the Item Data
