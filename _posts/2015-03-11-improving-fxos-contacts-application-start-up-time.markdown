---
layout: post
title:  "Improving the Firefox OS Contacts application start-up time"
date:   2015-03-11
categories: ["firefoxos", "performance"]
author: "Fernando Jiménez"
---

One of the biggest challenges that we have in Firefox OS is the [performance](https://developer.mozilla.org/en-US/Apps/Build/Performance/Performance_fundamentals). We have been fighting it since day one and by applying some [different techniques](https://developer.mozilla.org/en-US/Apps/Build/Performance/Optimizing_startup_performance) we managed to get to a point where we have some very decent [application start-up time numbers](https://datazilla.mozilla.org/b2g).

The last application to get a considerable performance boost has been the Contacts application.

During the last few weeks, the Contacts team [has been working](https://bugzilla.mozilla.org/show_bug.cgi?id=1112551) on a [patch](https://github.com/mozilla-b2g/gaia/commit/f1d0684817e5802961c02a04dcf667cfaf09d6ee) that finally landed on master yesterday. The result is an improvement of around 720 milliseconds of [perceived start-up time](https://developer.mozilla.org/en-US/Apps/Build/Performance/Firefox_OS_app_responsiveness_guidelines#Stages), which means that we saved almost 50% of the previous start-up time.

[Datazilla](https://datazilla.mozilla.org/b2g/?branch=master&device=flame-319MB&range=7&test=startup_%3E_moz-app-visually-complete&app_list=communications/contacts&app=communications/contacts&gaia_rev=9645d45d5777880e&gecko_rev=f6259882882b&plot=median) already shows the change.

![Datazilla changes](/content/images/2015/03/contactsperfimprovement-1.png)

[Comparing](https://github.com/stasm/test-perf-summary) the results of running the [Gaia performance tests](https://developer.mozilla.org/en-US/Firefox_OS/Platform/Automated_testing/Gaia_performance_tests) with a heavy workload before and after the patch we get the following numbers:

| communications/contacts (means in ms) | Base  | Patch | Δ    |
|:-------------------------------------:|-------|-------|------|
| moz-chrome-dom-loaded                 | 1147  | 585   | -562 |
| moz-chrome-interactive                | 1267  | 1393  | 126  |
| moz-app-visually-complete             | 1601  | 874   | -727 |
| moz-content-interactive               | 2131  | 1393  | -738 |
| moz-app-loaded                        | 10942 | 10409 | -533 |

As you can see we are sending the `moz-app-visually-complete` event ~727 milliseconds earlier than before. This is the event that we use to indicate that the application appears visually ready for user interaction and the one that we really want to send as soon as possible. We also get similar improvements for the `moz-chrome-dom-loaded`, `moz-content-interactive` and `moz-app-loaded` events. You can also notice that we had to make some trade offs and we lost some ground with the `moz-chrome-interactive` event. If you look closer, you will see that chrome and content are marked as interactive at the same time. This was not happening before. We were not able to interact with the application content until almost one second after being able to interact with the application chrome. Now we have everything ready at once ~700 milliseconds before and given that the most important part of the Contacts application is the contacts data itself, we consider that the small lost in the `moz-chrome-interactive` event is worth the result. You can check the [MDN responsiveness guidelines](https://developer.mozilla.org/en-US/Apps/Build/Performance/Firefox_OS_app_responsiveness_guidelines#Stages) page for more details about these events.

I recorded a quick video comparing the previous situation (left) with the current one (right). (Apologies for the low quality of the recording).

<iframe title="vimeo-player" src="https://player.vimeo.com/video/121901924" width="710" height="360" frameborder="0" allowfullscreen></iframe>

How did we get there
--------------------

The target was to have some usable UI ready as soon as possible before the browser painted anything on the screen. For us, this usable UI is the application chrome with the `Add contact` and `Settings` options and the first chunk of contacts, including favorite and [ICE](http://en.wikipedia.org/wiki/In_case_of_emergency) contacts.

So far we were not doing bad showing the application chrome, but we were taking extra time to load the first group of visible contacts. To show this first content we needed to do a request to the [MozContacts API](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/mozContacts) to obtain the list of stored contacts and start appending to the DOM one new node per each contact information retrieved. The thing is that the result of this request rarely changed from one execution to the other. So why not caching it?

We followed the same approach that the Email team already applied on the [Email application](https://groups.google.com/forum/#!topic/mozilla.dev.gaia/v_jVuwOJMKI) for caching the email list. We used [localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage) to save the result of getting the contacts list from the MozContacts API and rendering the first chunk of contacts. To avoid having to do object serialization and parsing before and after accessing the localStorage item, we initially tried storing the whole [outerHTML](https://developer.mozilla.org/en-US/docs/Web/API/Element/outerHTML) string of the [contacts groups container](https://github.com/mozilla-b2g/gaia/blob/master/apps/communications/contacts/index.html#L218) holding the first chunk of contacts and applying it via [innerHTML](https://developer.mozilla.org/en-US/docs/Web/API/Element/innerHTML), but that did not give good enough performance and it made the logic for managing the contacts cache harder. Also, in the end we figured out that we needed to store other information like the language direction or the cache date along with the HTML to decide wether the cache was valid or not, so object serialization and parsing was required in any case. Instead of that, we ended up storing an object with this information to ease the cache eviction decision and enough information to rebuild the DOM containing the first chunk of contacts. We applied this data to the DOM via [documentFragment](https://developer.mozilla.org/en-US/docs/Web/API/DocumentFragment). You can checkout the code for [building](https://github.com/mozilla-b2g/gaia/blob/master/apps/communications/contacts/js/views/list.js#L2274) and [applying](https://github.com/mozilla-b2g/gaia/blob/master/apps/communications/contacts/js/bootstrap.js#L173) the cache.

The trickiest part of maintaining this cache is the eviction policy. We need to evict and rebuild the cache every time a contact is changed (added, removed or edited) and because this can happen from inside and from outside of the Contacts app (even when the app is closed), we need to be specially careful and verify the cache after applying it to the DOM without affecting the performance or causing visual reflows. You can follow [this code](https://github.com/mozilla-b2g/gaia/blob/master/apps/communications/contacts/js/views/list.js#L2375) to see how we managed to do that. Other scenarios where we need to evict the cache are language direction changes, [ICE](http://en.wikipedia.org/wiki/In_case_of_emergency) contacts changes, favorite contacts modifications and when the user changes the way the contacts are displayed (by first or last name).

Apart from building the cache mechanism we also changed the application bootstrap process in a way that we only load the minimum set of scripts required to get the cached information from localStorage and to apply it in the DOM. Once we have this process completed, we load the rest of the application Javascript that is required to continue the rest of the boot process. You can see this logic in the new [bootstrap](https://github.com/mozilla-b2g/gaia/blob/master/apps/communications/contacts/js/bootstrap.js#L410) script.

Next steps
----------

We want to keep improving the performance of the Contacts application. The next thing that we want to target is improving the loading of the contacts thumbnails. In fact, [Francisco Jordano](https://twitter.com/mepartoconmigo) has already started working on [it](https://bugzilla.mozilla.org/show_bug.cgi?id=1089538) and there are already some visible improvements.

<iframe width="710" height="381" src="https://www.youtube-nocookie.com/embed/lOx-Ym2qUlM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

We also want to experiment with caching most part or even the whole contacts list in different chunks to allow the user to use the alpha scrolling and to get a fully loaded application even sooner.

Finally, the Gaia team is starting to play around with [Service Workers](http://www.html5rocks.com/en/tutorials/service-worker/introduction/) and with the idea of using this new feature to cache already rendered views in a similar way that we did for Contacts. I cannot wait to see more progress in this area :)

Credits
-------

[Francisco Jordano](https://twitter.com/mepartoconmigo), [Johan Lorenzo](https://github.com/JohanLorenzo), [Sergi Mansilla](http://sergimansilla.com/).
