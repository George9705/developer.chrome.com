---
layout: 'layouts/doc-post.njk'
title: 'Quick API Reference'
subhead: 'Tutorial focused on teaching extension service worker concepts.'
description: 'Quickly access Chrome API reference with Omnibox.'
date: 2023-04-02
# updated: 2022-06-13
---

## Overview {: #overview }

This tutorial builds an extension that allows users to open Chrome API reference pages using the omnibox. It also provides a daily Chrome extension tip.

{% Video src="video/BhuKGJaIeLNPW9ehns59NfwqKxF2/WmVEGpEZ9ts1J0pUOzEr.mp4", width="600", height="398", autoplay="true", muted="true"%}

This article will show how to do the following tasks in an extension service worker:

- Register a service worker and import modules.
- Find extension service worker logs.
- Manage state and handle events.
- Trigger periodic events.
- Communicate with content scripts.

## Before you start {: #prereq }

This guide assumes that you have basic web development experience. We recommend reviewing [Extensions 101][doc-ext-101] and [Development Basics][doc-dev-basics] for an introduction to extension development.

## Build the extension {: #build }

Start by creating a new directory called `quick-api-reference` to hold the extension files, or download the source code from our [GitHub samples][github-open-api] repo.

### Step 1: Register the service worker {: #step-1 }

Create the [manifest][doc-manifest] file in the root of the project and add the following code:

{% Label %}manifest.json:{% endLabel %}

```json/8-10
{
  "manifest_version": 3,
  "name": "Open extension API reference",
  "version": "1.0.0",
  "icons": {
    "16": "icon-16.png",
    "128": "icon-128.png"
  },
  "background": {
    "service_worker": "service-worker.js",
  },
}
```

Extensions register their service worker in the manifest, which only takes a single JavaScript file.
There's no need to use `navigator.serviceWorker.register()`, like you would in a web app. See
[Differences between extension and web service workers](tbd) to learn more.

You can download the icons located on the [Github repo][github-open-api].

### Step 2: Import multiple files {: #step-2 }

This extension has two features, so we will split the extension logic into two files. The following code declares the service worker as an [ES Module][mdn-es-module], which allows us to import multiple files:

{% Label %}manifest.json:{% endLabel %}

```json/3-3
{
 "background": {
    "service_worker": "service-worker.js",
    "type": "module"
  },
}
```

Create the `service-worker.js` file and import two modules:

```js
import './sw-omnibox.js';
import './sw-tips.js';
```

Create these files and add a console log to each one.

{% Columns %}

{% Column %}

{% Label %}sw-omnibox.js:{% endLabel %}

```js
console.log("sw-omnibox.js")
```
{% endColumn %}

{% Column %}
{% Label %}sw-tips.js:{% endLabel %}

```js
console.log("sw-tips.js")
```
{% endColumn %}

{% endColumns %}

See [Importing scripts](tbd) to learn about other ways to import multiple files in a service worker.


{% Aside %}

Remember to set `type.module` when using a modern module bundler framework.

{% endAside %}

### _Optional: Debugging the service worker_ {: #step-3 }

Let's quickly go over how to find the service worker logs and know when it has terminated. Follow the instructions to [Load an unpacked extension][doc-dev-basics-unpacked] and wait 30 seconds for the service worker to stop. Click on the "service worker" hyperlink to inspect it. 

{% Video src="video/BhuKGJaIeLNPW9ehns59NfwqKxF2/D1XRaA6q4xn9Ylwe1u1N.mp4", width="800", height="314", autoplay="true", muted="true", loop="true" %}

Did you notice that inspecting the service worker woke it up? That's right! Opening the service worker in the devtools will keep it active.

Now let's break the extension to learn where to locate errors. One way to do this is to delete the ".js" from the `'./sw-omnibox.js'` import in the `sw.js` file. Chrome will be unable to register the service worker.

Go back to chrome://extensions and refresh the extension. The following error will appear:

{% Video src="video/BhuKGJaIeLNPW9ehns59NfwqKxF2/AbMNDSbURLKjH1Jm1C9Q.mp4", width="400", height="477", autoplay="true", muted="true", loop="true" %}

See [Debugging extensions](tbd) for more ways debug the extension service worker.

{% Aside 'caution' %}
Don't forget to fix the file name before moving on!
{% endAside %}

### Step 4: Initialize the state {: #step-4 }

Extensions can save initial values to storage on installation. To use the [`chrome.storage`][api-storage] API, we need to request permission in the manifest:

{% Label %}manifest.json:{% endLabel %}

```json
{
  ...
  "permissions": ["storage"],
}
```

The following code sets the default [omnibox][api-omnibox] suggestions when the extension is first installed:

{% Label %}sw-omnibox.js:{% endLabel %}

```js
...
// Save default API suggestions
chrome.runtime.onInstalled.addListener(({ reason }) => {
  if (reason === 'install') {
    chrome.storage.local.set({
      apiSuggestions: ['tabs', 'storage', 'scripting']
    });
  }
});
```

Service workers do not have direct access to the DOM or the [window object][mdn-window], therefore cannot use
[window.localStorage()][mdn-local-storage] to store values. Also, service workers are short-lived execution environments;
they get terminated repeatedly throughout a user's browser session, which makes it incompatible with
global variables.

See [Saving states](TBD) to learn about storage options for extension service workers.

### Step 5: Register your events {: #step-5 }

All event listeners need to be registered in the global scope of the service worker. In other words, event listeners should not be nested in functions. This way Chrome can immediately invoke all event handlers, even if the extension's async startup logic hasn't finished. 

To use the [`chrome.omnibox`][api-omnibox] API first add the omnibox keyword to the manifest:

{% Label %}manifest.json:{% endLabel %}

```json
{
  ...
  "minimum_chrome_version": "102",
  "omnibox": {
    "keyword": "api"
  },
}
```

{% Aside 'important' %}
The [`"minimum_chrome_version"`][manifest-min-version] explains how this key behaves when a user tries to install your extension but isn't using a compatible version of Chrome.

{% endAside %}

The following code registers the omnibox event listeners at the top level of the script and updates [chrome.storage][api-storage] with the most recent api search.

{% Label %}sw-omnibox.js:{% endLabel %}

```js
...
const URL_CHROME_EXTENSIONS_DOC =
  'https://developer.chrome.com/docs/extensions/reference/';
const NUMBER_OF_PREVIOUS_SEARCHES = 4;

// Display the suggestions after user starts typing
chrome.omnibox.onInputChanged.addListener(async (input, suggest) => {
  await chrome.omnibox.setDefaultSuggestion({
    description: 'Enter a Chrome API or choose from past searches'
  });
  const { apiSuggestions } = await chrome.storage.local.get('apiSuggestions');
  const suggestions = apiSuggestions.map((api) => {
    return { content: api, description: `Open chrome.${api} API` };
  });
  suggest(suggestions);
});

// Open the reference page of the chosen API
chrome.omnibox.onInputEntered.addListener((input) => {
  chrome.tabs.create({ url: URL_CHROME_EXTENSIONS_DOC + input });
  // Save the latest keyword
  updateHistory(input);
});

async function updateHistory(input) {
  const { apiSuggestions } = await chrome.storage.local.get('apiSuggestions');
  apiSuggestions.unshift(input);
  apiSuggestions.splice(NUMBER_OF_PREVIOUS_SEARCHES);
  await chrome.storage.local.set({ apiSuggestions });
}
```

{% Aside 'important' %}

Extension service workers have access to both web APIs and Chrome APIs, with a few exceptions.
For a deep dive, see [Service Workers...](tbd) 

{% endAside %}

### Step 6: Set up a recurring event {: #step-6 }

The `setTimeout()` or `setInterval()` methods are commonly used to perform delayed or periodic
tasks. However, these APIs can fail because the scheduler will cancel the timers when the service
worker is terminated. Instead, extensions can use the [`chrome.alarms`][api-alarms] API. 

To use the Alarms API, request the `"alarms"` permission in the manifest. The extension also needs to request [host permission][doc-host-perm] to retrieve the extension tips from the glitch site:

{% Label %}manifest.json:{% endLabel %}

```json/2/3
{
  ...
  "permissions": ["storage", "alarms"],
  "permissions": ["storage"],
  "host_permissions": ["https://extension-tips.glitch.me/*"],
}
```

The following code sets up an alarm once a day to fetch the daily tip and save it to
[`chrome.storage.local()`][api-storage]:

{% Label %}sw-tips.js:{% endLabel %}

```js
// Fetch tip & save in storage
const updateTip = async () => {
  const response = await fetch('https://extension-tips.glitch.me/tips.json');
  const tips = await response.json();
  const randomIndex = Math.floor(Math.random() * tips.length);
  await chrome.storage.local.set({ tip: tips[randomIndex] });
};

// Create a daily alarm and retrieves the first tip when extension is installed.
chrome.runtime.onInstalled.addListener(({ reason }) => {
  if (reason === 'install') {
    chrome.alarms.create({ delayInMinutes: 1, periodInMinutes: 1440 });
    updateTip();
  }
});

// Update tip once a the day
chrome.alarms.onAlarm.addListener(updateTip);

```

{% Aside %}

All [Chrome API][doc-apis] event listeners and methods restart the service worker 30 second termination timer. To learn more about the extension service worker lifecycle, see [TBD](tbd)

{% endAside %}

### Step 7: Communicate with other contexts {: #step-7 }

[Content scripts][doc-content] communicate with the rest of the extension through [message passing][doc-messages]. In this example, the content script will request the tip of the day from the service worker. 

First, declare the content script in the manifest and add the match pattern corresponding to the [Chrome API][doc-apis] reference documentation.

{% Label %}manifest.json:{% endLabel %}

```json
{
  ...
  "content_scripts": [
    {
      "matches": ["https://developer.chrome.com/docs/extensions/reference/*"],
      "js": ["content.js"]
    }
  ]
}

```

Create a new content file. The following code generates a button that will open the tip popover. It also sends a message to the service worker requesting the extension tip.

{% Label %}content.js:{% endLabel %}

```js
(async () => {
  const nav = document.querySelector('.navigation-rail__links');

  const { tip } = await chrome.runtime.sendMessage({ greeting: 'tip' });

  const tipWidget = createDomElement(`
    <button class="navigation-rail__link" popovertarget="tip-popover" popovertargetaction="show" style="padding: 0; border: none; background: none;>
      <div class="navigation-rail__icon">
        <svg class="icon" xmlns="http://www.w3.org/2000/svg" aria-hidden="true" width="24" height="24" viewBox="0 0 24 24" fill="none"> 
        <path d='M15 16H9M14.5 9C14.5 7.61929 13.3807 6.5 12 6.5M6 9C6 11.2208 7.2066 13.1599 9 14.1973V18.5C9 19.8807 10.1193 21 11.5 21H12.5C13.8807 21 15 19.8807 15 18.5V14.1973C16.7934 13.1599 18 11.2208 18 9C18 5.68629 15.3137 3 12 3C8.68629 3 6 5.68629 6 9Z'"></path>
        </svg>
      </div>
      <span>Tip</span> 
    </button>
  `);

  const popover = createDomElement(
    `<div id='tip-popover' popover>${tip}</div>`
  );

  document.body.append(popover);
  nav.append(tipWidget);
})();

function createDomElement(html) {
  const dom = new DOMParser().parseFromString(html, 'text/html');
  return dom.body.firstElementChild;
}
```


{% Details %}
{% DetailsSummary %}
💡 **Interesting JavaScript used in this code**
{% endDetailsSummary %}

TBD

- DomParser
- Popover API
- SVG element

{% endDetails %}

The following code sends the daily tip from the service worker to the content script. 

{% Label %}sw-api.js:{% endLabel %}

```js
...
// Send tip to content script via messaging
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.greeting === 'tip') {
    chrome.storage.local.get('tip').then(sendResponse);
    return true;
  }
});
```

## Test that it works {: #try-out }

Verify that the file structure of your project looks like the following: 
<!-- TODO: Add actual file structure -->
{% Img src="image/BhuKGJaIeLNPW9ehns59NfwqKxF2/S86ooJMjFm5uvf906u9a.png", 
alt="The contents of the extension folder: manifest.json, service-worker.js, sw-omnibox.js, sw-tips.js,
content.js, and icons.", width="700", height="468" %}

### Load your extension locally {: #locally }

To load an unpacked extension in developer mode, follow the steps in [Development
Basics][doc-dev-basics-unpacked].

### Test the extension on a documentation page {: #open-sites }

TBD

## 🎯 Potential enhancements {: #challenge }

Based on what you’ve learned today, try to accomplish any of the following:

- Add a CSS stylesheet to the content script.
- TBD
- TBD

## Keep building! {: #continue }

Congratulations on finishing this tutorial 🎉. Continue leveling up your skills by completing other
tutorials on this series:

| Extension                        | What you will learn                                            |
|----------------------------------|----------------------------------------------------------------|
| [Reading time][tut-reading-time] | To insert an element on a specific set of pages automatically. |
| [Tabs Manager][tut-tabs-manager] | To create a popup that manages browser tabs.                   |

## Continue exploring

To continue your extension service worker learning path, we recommend exploring the following articles:

- TBD
- TBD
- TBD



[api-scripting]: /docs/extensions/reference/scripting/
[api-storage]: /docs/extensions/reference/storage
[api-alarms]: /docs/extensions/reference/alarms
[api-omnibox]: /docs/extensions/reference/omnibox
[doc-apis]: /docs/extensions/reference
[doc-dev-basics-unpacked]: /docs/extensions/mv3/getstarted/development-basics#load-unpacked
[doc-dev-basics]: /docs/extensions/mv3/getstarted/development-basics
[doc-devguide]: /docs/extensions/mv3/devguide/
[doc-ext-101]: /docs/extensions/mv3/getstarted/extensions-101/
[doc-manifest]: /docs/extensions/mv3/manifest/
[doc-perms-warning]: /docs/extensions/mv3/permission_warnings/#required_permissions
[doc-sw]: /docs/extensions/mv3/service_workers/
[doc-content]: /docs/extensions/mv3/content_scripts/
[doc-messages]: /docs/extensions/mv3/messaging
[doc-welcome]: /docs/extensions/mv3/
[github-focus-mode-icons]: https://github.com/GoogleChrome/chrome-extensions-samples/tree/main/functional-samples/tutorial.focus-mode/images
[github-open-api]: https://github.com/GoogleChrome/chrome-extensions-samples/tree/main/functional-samples/
[mdn-es-module]: https://web.dev/es-modules-in-sw/
[mdn-indexeddb]: https://developer.mozilla.org/docs/Web/API/IndexedDB_API
[runtime-oninstalled]: /docs/extensions/reference/runtime#event-onInstalled
[tut-focus-mode-step6]: /docs/extensions/mv3/getstarted/tut-focus-mode#step-6
[tut-reading-time-step1]: /docs/extensions/mv3/getstarted/tut-reading-time#step-1
[tut-reading-time-step2]: /docs/extensions/mv3/getstarted/tut-reading-time#step-2
[tut-reading-time]: /docs/extensions/mv3/getstarted/tut-reading-time
[tut-tabs-manager]: /docs/extensions/mv3/getstarted/tut-tabs-manager
[mdn-local-storage]: https://developer.mozilla.org/docs/Web/API/Window/localStorage
[doc-host-perm]: /docs/extensions/mv3/match_patterns/
[mdn-window]: https://developer.mozilla.org/docs/Web/API/Window
[manifest-min-version]: https://developer.chrome.com/docs/extensions/mv3/manifest/minimum_chrome_version/#enforcement