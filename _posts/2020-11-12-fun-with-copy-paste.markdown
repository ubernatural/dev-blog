---
layout: post
title:  "Fun (and Danger) of Copy Paste"
date:   2020-11-12 08:00:00 -0700
categories: development
comments_id: 7
---
Here is something that you've likely run into if you have been programming for a long time. It falls into the category of 'exceptionally difficult to troubleshoot' and so you should always be aware (and reminded) of it. It bit me today, I think the last time I encountered it was about 10 years ago --long enough that I became complacent.

If you don't read any further, take this advice: do not copy any sort of formatted text into your code. This includes text that seems like it's not formatted but is presented in a document viewer (e.g. Word, PDF), web page, or in this case a collaboration tool like Microsoft Teams.

I was in a MS Teams meeting when an innocuous bit of configuration was posted in the chat for someone to copy into their Geocortex Viewer configuration. It looked like this:  
`{siteToken}`

This was pasted into the viewer configuration using Notepad++. I'll use a screenshot for this one in particular...the raw token itself looked like this:  

![]({{ '/assets/images/HiddenDanger.png' | relative_url }})

But the viewer didn't work. Enter a bunch of troubleshooting with breakpoints and immediate commands and we discovered that we were looking for the pattern `{siteToken}` but that pattern did not exist in the configuration file. Hmmmm. But clearly it _was_ in the configuration. At this point I recalled the pain from 10 years earlier and had the bright idea to show all characters in Notepad++. 'That'll fix it!' I said. Here is the result.  

![]({{ '/assets/images/VeryWellHiddenDanger.png' | relative_url }})

Boo. That was anticlimactic. Pressing on though because it _still_ made no sense, we meticulously re-typed all of the tokens that we had copy/pasted from the MS Teams chat and tried again. It worked. To give you an idea of how challenging this was, here is the __bad__ token beside the __good__ token in Notepad++, showing all characters:  

![]({{ '/assets/images/SpidermanPointingCode.png' | relative_url }})

Look as closely as you want, you will not see any differences. The times I've run into this before it was usually with either line break differences (a classic) or single/double quote differences copying out of MS Word. In both those cases there are _very subtle_ differences in fixed-width fonts that you can identify. Not so here.

What happened was that copying out of MS Teams brought along with it two __Zero Width Space__ characters - one after the opening brace and one before the closing. Yes, not just a hard-to-see whitespace character, but one with _Zero Width_. Talk about an imposter.  

![]({{ '/assets/images/SpidermanPointing.png' | relative_url }})

Now, showing all characters in your editor is a great option and is still the best first check. However, another trick that I didn't use (cause I didn't realize until after the fact) is to change the _encoding_ in your editor. Previously hidden things become visible! Like throwing a handful of flour on a ghost (or on Predator). Here is our same two tokens, but changed to ANSI encoding from UTF-8:  

![]({{ '/assets/images/Gotcha.png' | relative_url }})

__Gotcha!__  
...he says, about an hour too late...