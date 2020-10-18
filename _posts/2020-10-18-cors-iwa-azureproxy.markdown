---
layout: post
title:  "CORS, IWA, and the Azure App Proxy"
date:   2020-10-18 08:00:00 -0700
category: general
tags: troubleshooting network azure iwa cors proxy
comments_id: 6
---
We had a challenging (meaning very interesting) troubleshooting session last week that was difficult due to a specific network and ArcGIS configuration mix. This is a summary of that troubleshooting session - you may not encounter the exact same situation as we did but it's good to understand the moving pieces.

### The Back Story
We started with a requirement to expose a GVH site and internal ArcGIS server to external users. The external users should still have to sign in, but they should not need a VPN or some other way of being on the client's internal network. The GVH and ArcGIS servers are humming along nicely on the internal network already.

The typical way that we've seen to allow this is to configure a reverse proxy on the client's network. This is a server (or other appliance) that sits in an isolated network space (DMZ) that exposes ports to the big bad Internet (:443 for example). The internal network has a port opened to the DMZ (again, :443) so that the proxy can forward Internet requests through the firewall in to the internal servers. In our case however the client chose to use the Azure Active Directory Application Proxy.

### Azure AD Application Proxy
If you are familiar with reverse SSH tunneling, think of the Azure AD Application Proxy as reverse SSH tunneling for Windows and Azure. If you're __not__ familiar with reverse SSH tunneling, it's awesome. Take a machine on your internal network and make an __outbound__ connection to a server on the Internet. That server you've connected to is then able to maintain this persistent connection back to the original internal server and feed data to it. If that server on the Internet is configured to accept HTTP requests, it can then forward those requests to the internal server on that persistent connection. The internal server then can be configured to forward those HTTP requests further along to any other internal server it has access to. It's exposing internal resources to the internet without having any incoming holes poked in the firewall. Sound like a security risk? Well yeah, it totally could be if you're not careful. But sometimes with great responsibility comes great power (a central theme of this blog), and this is an established pattern.

>Side note: this is called the Azure AD Application Proxy instead of just the Azure Application Proxy because it is only available if you have a subscription to Azure AD, and is a subcomponent of that subscription. I don't know why.

The Azure AD Application Proxy requires a _connector_ that you install on an internal server, then you log in to the connector using Azure AD application admin credentials. This connector functions similarly to the originating SSH client in the reverse tunneling scenario. Once logged in, the connection is established with Azure so you can log in to the Azure portal and declare that _this external URL/domain_ should redirect to _this internal URL_ please. As long as the internal URL can be accessed from the Azure AD App Proxy connector server, the forwarding should work just like a regular reverse proxy.

Thus set up, with geocortex.client.com pointing to the internal GVH server and arcgis.client.com pointing to the internal ArcGIS server, our troubleshooting begins.

### IWA and the GVH Proxy
The first issue was that the ArcGIS map services were protected using Integrated Windows Authentication. This is fine if someone is hitting the ArcGIS endpoints directly from their browser and can log in. In the case of GVH though, by default the XmlHttpRequests are routed through the GVH proxy to bypass CORS issues. It is therefore the proxy that makes the request to ArcGIS Server, not the user, and so is not successfully authenticated.

Mike in support came through immediately with the recommended solution for this, which was to reconfigure GVH (directly in the index.html page) to whitelist the ArcGIS Server URL to bypass the GVH proxy. GVH and the client's ArcGIS Server negotiated the CORS requests just fine, so the proxy was not necessary and the user could authenticate to the ArcGIS Server successfully.

### Cross-Origin Resource Sharing
A brief note about CORS - this is a mechanism enforced by the browser that allows scripts (XHR) to connect to certain cross-domain resources on other domains. GVH on geocortex.client.com has scripts that load Map Service information from arcgis.client.com. Look at [the Wikipedia CORS page](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) for a good summary of what's going on there. CORS will come up later in our troubleshooting journey, with a twist.

### Odd Behaviour
This seemed odd to us at first, but of course now that we know what's going on it makes total sense. After whitelisting the ArcGIS Server we found that we could launch the site just fine - all the maps loaded and we thought it was AOK. However, the client could not load the site - they were getting a popup in GVH on load that said that all of the map services were currently unavailable. When we tried launching in an incognito window we saw the same thing. We would then go to the network tab in the Chrome debugger, grab the offending URL to the ArcGIS Service, and paste it in. The page loaded just fine for us after we logged in. Once we loaded the ArcGIS URL in the browser once, we could launch the GVH site and all the maps would load.

The fact that the viewer worked at all suggested that CORS wasn't the issue. There were a couple of things that ended up being red herrings:
- The SSL certificates for Geocortex and ArcGIS caused errors in the browser (expired)
- The ArcGIS Server was negotiating NTLM authentication, so a username/password prompt was coming up in the browser
  - This is still an outstanding, but unrelated issue

One thing that we found unusual that would end up being a valuable data point was the fact that when we launched GVH we were prompted to sign in using our domain credentials; however, we noticed that we were in fact _not_ signed in to the viewer once it loaded.

### CORS Again
We noticed in the Chrome dev tools that when we had previously loaded and logged in to the ArcGIS Server the network requests were fine. If we loaded the viewer _before_ having logged in to ArcGIS Server we saw errors in the console suggesting that the requests were blocked by CORS policies. Why that?

Here is where a key bit of information about CORS requests connects with that bit of information about the weird logged in / not logged in behaviour mentioned above. Add to that a little more information about the Azure AD Application Proxy and we can understand what's happening.
- Key info 1: CORS requests will fail if the response from the server is a 3xx redirect to a server on a different domain (with some exceptions).
- Key info 2: The Azure AD App Proxy is configured by default to preauthenticate all users entering the proxy, regardless of whether the resource at the other end is secured.
- Key info 3: The Azure AD App Proxy authenticates by redirecting users to login.microsoftonline.com.

### What Was Happening
Based on the key information listed above you may be able to figure out what was happening on your own. Here is the sequence for the normal (failing) launch:
- The user loads the viewer URL in their browser.
  - The URL is to the Azure AD App Proxy, which sends a redirect to login.microsoftonline.com (Azure AD login page).
- The user logs in to the Azure AD App Proxy using their domain credentials.
  - Importantly, here the user is logging in to the _proxy_, not to the resource behind the proxy.
- The user is redirected back to the viewer URL with an `AzureAppProxyAccess` cookie attached. The viewer loads.
- During the load process, an XHR request is sent to arcgis.client.com to get the map service info. This is a CORS request.
  - The URL is also to the Azure AD App Proxy, but the `AzureAppProxyAccess` cookie is __not__ sent because it's for just the geocortex.client.com domain.
- The Azure AD App Proxy responds to the XHR request with a 302 request to redirect to login.microsoftonline.com.
- Being on a different domain than arcgis.client.com, the CORS request fails with extreme prejudice.

Now, if I load the ArcGIS URL on its own beforehand, my browser redirects me to the Azure AD login page where I can log in to the proxy. Now that I've logged in to the proxy I have the `AzureAppProxyAccess` cookie for arcgis.client.com, and the subsequent XHR requests are __not__ redirected to login.microsoftonline.com. Thus, the CORS requests are just fine from then on.

Incidentally, I don't even need to load the ArcGIS URL itself to get it to work, as long as I load a URL that hits the App Proxy. For example, instead of loading `https://arcgis.client.com/arcgis/rest/services/servicename/MapServer`, I could just load `https://arcgis.client.com/foo` (and get a 404). This is because the App Proxy is forwarding everything from that base domain along to the ArcGIS Server.

### How to Resolve This?
It's nice to know what's going on, but how can we resolve this? We can't just reinstate the GVH proxy (and forget about the CORS requests) because the proxy will not be able to authenticate to the ArcGIS Server (see problem #1 above). Here are a couple of options.

#### Normalize the Domains
First, the client could make it so Geocortex and ArcGIS Server are both on the same domain. Typically this might be do-able by reconfiguring the App Proxy to proxy URL paths to servers instead of proxying the entire server. Meaning, instead of saying _please proxy everything to geocortex.client.com to internal-gcx-server.client.com_, we could say _please proxy geocortex.client.com/Geocortex to internal-gcx-server.client.com/Geocortex_. This would allow us to set up the ArcGIS proxy as `geocortex.client.com/arcgis --> internal-ags-server.client.com/arcgis`. The Azure App Proxy lets you do this, thus having two internal servers on the same external domain.

The problem with that is that the client does not send internal users out through the App Proxy when using the maps. The internal DNS has arcgis.client.com pointing to the internal-ags-server IP and geocortex.client.com pointing to the internal-gcx-server IP. If the Geocortex site config were updated so all ArcGIS Server URLs used geocortex.client.com/arcgis, internal users would be out of luck.

An alternate approach would be to set up a proxy in IIS using ARR and URL rewriting on the Geocortex Server that would forward everything on the /arcgis path to the internal-ags-server. The client would then delete the App Proxy entry for arcgis.client.com because everything could go through geocortex.client.com both internally and externally.

#### Disable Preauthentication
A different approach could be to disable preauthentication on the App Proxy. The App Proxy doesn't _have_ to authenticate users to just use the proxy - if the internal resource is secured itself then the App Proxy could be configured to pass the request through every time. This would mean that the XHR requests to arcgis.client.com would not return a 3xx redirect, and the CORS requests would be successful.

One big problem with this is that since the requests are not preauthenticated by the App Proxy, the server is open to the Internet and if someone mistakenly publishes something onto that server that allows anonymous access, it'll mean anyone in the world will be able to get there.

Another problem relates to the issue of the NTLM negotiation. Even though ArcGIS Server is using NTLM right now, it would __ideally__ authenticate using Azure AD like the proxy does. This would avoid the scenario where the user has to log in twice when hitting the app from outside (once authenticating to the App Proxy for GVH and another time authenticating to ArcGIS Server via NTLM). Disabling preauthentication wouldn't really work in that scenario since the ArcGIS Server itself would respond with a 3xx redirect and we'd be back in the same boat.

#### Wildcard Proxy
A better option would address the issue of why the App Proxy is authenticating the user for arcgis.client.com when they've already authenticated to geocortex.client.com. We know that this technically is because the `AzureAppProxyAccess` cookie is only for `geocortex.client.com`, but wouldn't it be great if that `AzureAppProxyAccess` cookie could also be forwarded along to the arcgis proxy? Turns out there is a way in the App Proxy to configure [wildcard](https://docs.microsoft.com/en-us/azure/active-directory/manage-apps/application-proxy-wildcard) proxies. This would allow the client to set up something like geocortex.maps.client.com and arcgis.maps.client.com, and then proxy *.maps.client.com broadly. Since wildcard applications share the same authorization settings the `AzureAppProxyAccess` cookie will be set for *.maps.client.com and will be shared across both servers. This means only one login (when loading GVH) and no CORS issues. We still have the problem of the NTLM negotiation but once this is resolved this solution will continue to work.

### Afterword
At the time of writing the solutions above are only _speculative_; I have only thought this through in my head and have not performed any implementation tests. I will update this post if we find that some of the solutions are invalid!