This post attempts to illuminate the methods that Geocortex Workflow (GXWF) Designer uses to store its Workflows, where URLs to on-prem GXWF servers and other things like custom GXWF activity packs are configured, and how to manually update these when the need arises. Buckle up!
  
All Workflows are saved as items in the ArcGIS portal you signed in to GXWF Designer with. For client-side Workflows everything is self-contained within the portal item. Server-side Workflows are a bit more complicated but we'll talk about those in detail below.
  
A bit confusingly, Workflows are stored as 'Web Mapping Application' items in portal. ArcGIS does not allow custom portal item types so Workflows just use the closest thing that makes sense (though a custom 'Workflow' item would seem logical if it were allowed). A convenient thing with 'Web Mapping Application' portal items is that they have a built-in way to launch the application from a URL.

Insert screenshot

Incidentally, Workflow takes advantage of this by launching GXWF Designer with the Workflow already loaded in when you click the 'View Application' button. We'll talk about how it does this a bit later on - it's important to know since you may need to manually update this Designer URL value at some point.

Portal items are identified by a non-deterministic random GUID, unlike something like git which creates a GUID from the hash of a fiie's contents. This means that whenever you create a new Workflow it will have a unique URL. This includes when you import a previously-exported Workflow using GXWF Designer. This leads us to the following important characteristic of Workflows:
  
> Any time a Workflow is created or imported, it is given a globally unique URL
  
While the statement above may sound obvious to those who have worked with ArcGIS portals, it has implications when you need to migrate Workflows from one server to another such as when deploying to a client environment or moving from a development to a production Portal instance.
  
All portal items have metadata that can be retrieved using a portal REST API call. This is the portal item itself, and is represented as a JSON object. This is how Workflows store their name, description, tags, and other such information.
  
Certain portal items, such as Web Mapping Applications, also have the ability to store text or binary data alongside. This is referred to as the itemData, and is where GXWF stores the actual configured Workflows themselves. These are stored as JSON objects and can be easily inspected using a tool like ago-assistant or by making a REST call to retrieve the itemData directly in a browser.