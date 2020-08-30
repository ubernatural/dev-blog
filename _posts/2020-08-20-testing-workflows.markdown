---
layout: post
title:  "Testing Workflows"
date:   2020-08-20 08:00:00 -0700
categories: workflow
comments_id: 4
---
Usually when we write Workflows we end up testing them all at once at the end, when they are complete. For small Workflows this integration testing isn't so bad but it is much harder with large Workflows. When Workflows are broken down into more manageable sub-Workflows, however, it makes testing a lot easier.
  
Let's consider an example. Amanda wrote a nice example Workflow that replicated the core functionality of the v4 Workflow's 'Get Map Service Info' - the one that takes a map service ID and a layer name and gives you back the layer ID and the layer URL. I took that Workflow and modified it slightly to accept the map service ID and layer name as inputs so it can be used as a sub-Workflow. That modified Workflow is [here](https://latitudegeo.maps.arcgis.com/home/item.html?id=fe0b5c9c715d4b79ad701df6839e1b2d).
  
Now that we have a sub-Workflow, we need a way to test it independent of the master Workflow. We can write a basic 'Test Runner' Workflow that will invoke the sub-Workflow with any inputs you like and print the resulting outputs for you. I've created a Test Runner for the Get Map Service Info sub-Workflow [here](https://latitudegeo.maps.arcgis.com/home/item.html?id=e42e5fa893a348729ece416947ce951a). The Test Runner has only a Display Form activity, plus an event handler Workflow on button press (which is where the `RunWorkflow` activity is).
  
Finally, I configured the Test Runner Workflow on the toolbar of my GVH viewer. In many cases your sub-Workflows are independent enough to test right in the sandbox - in this case the Get Map Service Info sub-Workflow is specific to GVH so won't work properly in the sandbox environment. For a recent project I actually added all of the Test Runner Workflows to a Testing toolbar tab in the main GVH site so I could easily run any of them on demand.
  
You can open up those Workflows referenced above to get a good idea of how this all works, but here are some screenshots as well to illustrate it a bit more clearly.
  
### Launching the Test
Here is the Test Runner configured as a tool on a new 'Test' tab.  

![]({{ '/assets/images/LaunchTestRunner.png' | relative_url }})

### The Test Runner Form
Here is the form as it initially displays. I put it in a modal just because I didn't want to bother with the narrow side panel view. It is pre-configured with some valid sample values that can be changed to test other scenarios.  

![]({{ '/assets/images/TestRunnerForm.png' | relative_url }})

### The Results
When you click `Test Workflow`, the event handler will invoke the sub-Workflow with the inputs, and write out the outputs as JSON at the bottom of the form. Note that the outputs of the `RunWorkflow` are of course also visible in the browser console if you prefer to look there (which can be good for more complex outputs).

![]({{ '/assets/images/TestResultsGood.png' | relative_url }})

### Exposing Errors
It becomes really obvious when you have cases that you haven't covered, such as when a layer is not found.

![]({{ '/assets/images/TestResultsBad.png' | relative_url }})

One of the most compelling things for me about this type of solution is that it opens the door for more automated tests of sub-Workflows. Instead of manually typing in different inputs, we could feed this Test Runner a JSON configuration file with arrays of inputs and expected outputs, and have the Test Runner loop through all of them quickly and show a total summary. Adding new cases would be simple, and re-running all the tests for regressions would be also very simple. An automated Test Runner solution may be a blog post for a future time, who knows?