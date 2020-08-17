---
layout: post
title:  "Converting a Complex v4 Workflow to GXWF"
date:   2020-08-16 14:19:00 -0700
categories: workflow
comments_id: 2
---

We (Services) recently converted a very complex Essentials (v4) Workflow into 5-Series (GXWF). *Converted* isn't really the best word to use because there was nothing that could be reused, so it's better to say it was re-written. This will be the case for pretty much all v4 Workflow conversions because the underlying technology is different.
  
To get a sense of the complexity of the original Workflow, the client had 4 other Workflows in their site. These were average complexity, nothing special. The file sizes of these other Workflows varied between about 25kb to 52kb. The Workflow we re-wrote was ~740kb. So, 15-20 (or more) times larger than an average, 'basic' Workflow.
  
We had 80 hours budgeted to complete the Workflow, and there were two developers on the project (Nick Allen and me). The complexity of the Workflow meant that there were going to be some challenges:
- How could we both work in parallel on the same Workflow?
- How will we test this when it's finished, since the number of paths through the Workflow is so large?
  
## Our Solution
Our solution was to break the Workflow into sub-components, no different than factoring a massive function into smaller, more manageable ones. Each sub-component became its own Workflow which was called from a master orchestration Workflow. The orchestration Workflow was responsible for controlling the flow and communicating error messages back to the user.
  
Some sub-Workflows included user interaction, such as prompting for an address and zooming the map to the location. Other sub-Workflows were server-side, and others were just client-side Workflows that handled a specific task. In all cases the sub-Workflows were passed a limited set of input arguments and returned one or two outputs.
  
## Limiting Context
It was important that we did not make the sub-Workflows reliant on the entire context of the orchestration Workflow. The reason is that we needed to be able to work on different sub-Workflows independent of each other. By defining a rigid contract for each of the sub-Workflows we could work on them in isolation. In fact, using this pattern we wrote all of the sub-Workflows before even starting on the main orchestration Workflow. We just needed to ensure that the sub-Workflows we wrote satisfied the API contract that was designed when the original Workflow was broken down into its constituent parts.
  
Testing was significantly easier using this approach. For each sub-Workflow we wrote a 'test harness' or 'invoker' Workflow, usually a simple `Display Form` that we used to supply just the inputs that the sub-Workflow needed. This allowed us to quickly change the inputs to test multiple scenarios and inspect the output (usually just written to a `Text` element in the `Display Form` itself). Being able to test individual components independently without running through the entire master Workflow was a huge time saver.
  
## Result
We ended up writing 11 Workflows - the main orchestration Workflow plus 10 other sub-Workflows. To keep track of all of these Workflows we used the following naming convention:  
`CLIENT - Workflow Name - Subroutine Name`  
We managed the Workflow items in the Latitude ArcGIS Online organization, creating an AGOL group to hold everything together. When it came time to deploy to the client's organization we dropped the `CLIENT` part of the name and kept the rest.
  
Supporting the Workflow we wrote two simple custom Workflow activities (both client-side), one GXR report template, one 'scratch' polygon layer in AGOL, and one Web Map to support the report. This was deployed in a GVH site, but there is very little code that's GVH-specific so it can be ported to GXW in the future without much difficulty (save for an Essentials Feature Map we had to use).

We hit the 80 hour budget target for development of the Workflow though we overran it during deployment for various reasons, most of which were out of our control (as is tradition).
  
## Tips
We learned a few things along the way that we'd like to share.  
### Supply sub-Workflow URLs to the Orchestration Workflow as Arguments
You invoke a sub-Workflow using the `Run Workflow` activity, which takes a URL of the Workflow you want to run. If you hard-code this URL in the `Run Workflow` activity it will be difficult to modify when you deploy to the client environment (as well as all the other times that the URL of your Workflow changes).
### Enclose All Run Workflow Activities in Try/Catch
You typically want your orchestration Workflow to be the only one responsible for displaying error messages to the user. When something goes wrong in a sub-Workflow the orchestration Workflow often needs to clean up markup and other things before exiting (if the error is non-recoverable).
### Understand Where and How Workflows are Stored
This deserves a blog post on its own (forthcoming). Sometime during the course of development and client deployment you are likely to run into errors where your Workflow is trying to load a URL that it shouldn't be loading. This is particularly true when you have more than one developer collaborating on a single Workflow. In these cases it can be invaluable to know exactly where all of the Workflow code and descriptions are stored to easily track down and resolve the source of the issue. Often this is done using a targeted strike with ago-assistant, Postman, or a similar tool. Grokking Workflow in this manner can literally save hours on a project.