---
layout: post
title:  "How are Workflows Stored?"
date:   2020-08-19 08:00:00 -0700
categories: workflow
comments_id: 3
---
This post attempts to illuminate the methods that Geocortex Workflow (GXWF) Designer uses to store its Workflows, where URLs to on-prem GXWF servers and other things like custom GXWF activity packs are configured, and how to manually update these when the need arises. Buckle up!
  
### Client Workflows
All Workflows are saved as items in the ArcGIS portal you signed in to GXWF Designer with. For client-side Workflows everything is self-contained within the portal item. Server-side Workflows are a bit more complicated but we'll talk about those in detail below.
  
All portal items have metadata that can be retrieved using a portal REST API call. This is the portal item itself, and is represented as a JSON object. This is how Workflows store their name, description, tags, and other such information.
  
Certain portal items, such as Web Mapping Applications (which is what Workflows are stored as), also have the ability to store text or binary data alongside. This is referred to as the `itemData`, and is where GXWF stores the actual configured Workflows themselves. These are stored as JSON objects and can be easily inspected using a tool like [ago-assistant](https://ago-assistant.esri.com/) or by making a REST call to retrieve the `itemData` directly in a browser or with Postman.
  
The actual Workflow itself - all of the activities, their inputs, and the logic stringing them together are stored as a JSON object in the portal item's `itemData`.
  
When you export a Workflow using Workflow designer, it downloads a JSON file to your computer. This JSON file is identical to the content of the `itemData` that you can see in ago-assistant. Note that the portal item content (name, description, etc.) is **not** included as part of the JSON file you export.
  
### Server Workflows
Server workflows are stored on the server that Geocortex Workflow on-prem is installed on. That is to say, the **actual** Workflow is stored there - there is still an item saved in portal but this is effectively just a pointer to the Workflow on the server.
  
You can't create a server Workflow with the SaaS designer (at apps.geocortex.com). The point of server Workflows is to get access to internal network resources like databases and file systems - so even if there were a place to store your Workflows in SaaS it wouldn't make a lot of sense (edge cases aside).
  
The Workflows are stored (typically) in `C:\ProgramData\Geocortex\Workflow\Workflows`. The Workflows are named based on the ID of the portal item that points to them. They also have a numeric suffix that increments each time the Workflow is saved in designer. It keeps a record of all versions of the Workflow that have been saved - designer doesn't yet do anything with this but conceivably in the future it could support something like 'roll back to previous version'. In the meantime, if you have access to the Workflows directory you can grab any older version you'd like and revert manually (Don't know how? Ask me!).

![]({{ site.baseurl }}/assets/images/WorkflowFileList.png)
  
If you look at the `itemData` for a server Workflow in ago-assistant, you will see that there actually is a Workflow there. What the heck? I just said that server Workflows are stored on the Workflow server, not in portal! The Workflow you see in `itemData` here is the pointer - if you were to open it in Designer you'd see that it just collects all of the inputs supplied to it, and invokes your actual server Workflow with a `RunWorkflow` activity.
  
`RunWorkflow` is special in that it first retrieves the portal item (the metadata) to see if the Workflow is client or server. If it's server, it doesn't bother retrieving the `itemData` at all, and just makes a direct REST call out to the Workflow server. Technically you could have a server Workflow, **delete** the `itemData`, and still be able to execute the server Workflow using a `RunWorkflow` activity.
  
So, why is the 'pointer' Workflow needed at all? Couldn't the execution engine in the Workflow clients handle that directly? Not sure, but my guess is that it made things a lot cleaner to centralize that server interaction code (with polling due to async execution) in one spot. I'll edit this post if that turns out to be not correct!
  
### You Have the Power
With this knowledge of how Workflows are stored, you can do things like the following:
- Export a Workflow without needing to have access to Workflow designer by copying the `itemData` using ago-assistant
- Update a Workflow from a dev server to another server (test, prod) without changing the Workflow's URL by coping the `itemData` from one to the other. The export/import dance works well too, but it'll always import the Workflow as a copy, not overwriting the target while keeping the URL the same. (_edit: there's now a detailed post on that [here]({{ '/2020/08/30/deploying-workflows-to-new-server/' | relative_url }})_)
    - This requires that you manually edit the Workflow license URL, or at least open and save the target Workflow in designer once it's copied over (this automatically resets the license URL)
    - You may have to edit other URLs such as to custom activity packs, other `RunWorkflow` activities, but you'd have to do that anyway.

### And With Great Power...
Know that you can mess things up if you go digging in the Workflow portal items and on the on-prem file system. As long as you're comfortable with this and you've got backup plans for when things go wrong it's OK. But doing something such as 'oh, I'm just going to set this client Workflow to be a server Workflow by manually toggling the `isServerWorkflow` flag to true!' may not do what you think it will (Left as an exercise for the reader, but hint...you can recover even from this!)