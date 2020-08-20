This post attempts to illuminate the methods that Geocortex Workflow (GXWF) Designer uses to store its Workflows, where URLs to on-prem GXWF servers and other things like custom GXWF activity packs are configured, and how to manually update these when the need arises. Buckle up!
  
All Workflows are saved as items in the ArcGIS portal you signed in to GXWF Designer with. For client-side Workflows everything is self-contained within the portal item. Server-side Workflows are a bit more complicated but we'll talk about those in detail below.
  
All portal items have metadata that can be retrieved using a portal REST API call. This is the portal item itself, and is represented as a JSON object. This is how Workflows store their name, description, tags, and other such information.
  
Certain portal items, such as Web Mapping Applications (which is what Workflows are stored as), also have the ability to store text or binary data alongside. This is referred to as the `itemData`, and is where GXWF stores the actual configured Workflows themselves. These are stored as JSON objects and can be easily inspected using a tool like ago-assistant or by making a REST call to retrieve the `itemData` directly in a browser.
  
The actual Workflow itself - all of the activities, their inputs, and the logic stringing them together are stored as a JSON object in the portal item's `itemData`.
  
When you export a Workflow using Workflow designer, it downloads a JSON file to your computer. This JSON file is identical to the content of the `itemData` that you can see in ago-assistant.