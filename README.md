# Find-in-page Improvements

## Introduction

There have been a lot of interest in improving the find-in-page experience from web developers, including:
* Giving the user a completely custom UI, suppressing the browser’s find UI, consistently
* Allowing data not in the DOM tree to be findable, even if there might be an infinite amount of data
* Reflecting real-time data loading/modification on find results
* Treating multiple webpages as one searchable instance, e.g. books, multi-page articles, or the multipage HTML spec 

Sources: [1](https://github.com/whatwg/html/issues/2858), [2](https://lists.w3.org/Archives/Public/public-whatwg-archive/2014Feb/0140.html), [3](https://lists.w3.org/Archives/Public/public-webapps/2013OctDec/0640.html), [4](https://discourse.wicg.io/t/allowing-browser-find-in-page-to-find-non-dom-text/1205)

To address these interests, we are presenting a collection of Web APIs to interact with or even completely override the browser’s find-in-page UI.
### Goals

* Allow web authors to give their custom find UI instead of the browser’s find UI consistently **(focus of this explainer)**
* Allow the browser’s find-in-page to find everything in the webpage that the web author deems as findable **(in [another document](explainer-extended.md))**
### Non-goals

* Extending the browser's built-in UI to find items that will never be in the current page's DOM, such as other chapters in a book, or other threads in a forum

## Providing a custom find-in-page UI


### Example code

```js
// Intercepting find-in-page and completely replacing it with a custom UI
window.addEventListener('openfind', e => {
  e.preventDefault();
  const searchUI = generateSearchUI(); // details elided
  document.body.appendChild(searchUI);
  searchUI.querySelector('input[type="search"]').focus();
});
```

### `openfind` Event

When the user initiates action to open browser’s Find UI (by keypress or menu selection), the browser will first fire a DOM Event `openfind` (using the Event constructor). If the webpage cancels the event by calling `preventDefault()`, then the browser’s Find UI will not be shown. Otherwise, the browser's Find UI will be shown as usual.

The browser should also provide a "power user" mechanism for accessing the default Find UI that is not interceptable; for example, something like Ctrl+Shift+F, holding Shift while clicking the menu item, or a page-level preference similar to the popup blocker toggle or other permissions. In most cases sites will be using `openfind` to fix an otherwise-broken find-in-page experience, where the important content is not in the DOM due to lazy- or canvas-based rendering. But sometimes power users have a clear concept of the DOM vs. other in-memory data structures, and will want a way to search the DOM specifically, instead of using the web page-supplied UI to search over arbitrary data.

### Use case

A web page has a completely custom way of loading data, and they want to provide a completely custom search UI. They can do this by adding an event listener for `openfind` Event, and calling `preventDefault()` on it, and then show their search UI instead.

### Real world example
Currently there are a lot of websites that try to do this by detecting Ctrl+F, but they failed to detect when a user opens the Find UI from the browser’s menu (which is pretty much the only option in mobile). This creates a different UX when the custom UI failed to show, and might confuse users.

Some examples:

* [Github's text editor](https://gist.github.com/)
  * Their custom UI provides the ability to search with regex, and on long files, searching the whole file with their own UI
  * When find-in-page is done through browser UI (mobile, text editor is not focused when user does Ctrl+F, etc), some text on the bottom part of a really long file may not be found, and user can't use regex as usual.
  * They will benefit from this API so that they can consistently show their find-in-page UI in every case, and they need their own UI because they want to support regex queries.
  
* Office Online, Google Docs, Sheets, Slides
  * Their custom UI provides the ability to search the whole file
  * When find-in-page is done through browser UI on a sufficiently large file such as a Doc/Slide with many pages or a Sheet with many row, not everything in the file is found (only those within a few rows/pages away are found)
  * They will benefit from this API so that they can consistently show their find-in-page UI in every case
  * In the long term, they might also benefit from the API in the [extended explainer](explainer-extended.md) that allow webpages to add results to browser's results


* [Discourse](https://discourse.wicg.io/)
  *  Their custom UI provides the ability to search in other topics, or search for users/categories/posts etc
  *  When find-in-page is done through browser UI, user can only search within the opened topic/page, and text in posts on the bottom might not be found if the topic has a lot of posts
  * They will benefit from this API so that they can consistently show their find-in-page UI in every case, and they need their own UI because they want to support finding in other pages other than the currently opened one.


## Other find-in-page APIs
Other than suppressing the browser's Find UI, there are some cases that we want to propose solutions for, such as making the browser wait for loading of data before proceeding a find action or adding to the browser's list of find results. As these cases are more complicated, we are making a separate document for those cases [here](explainer-extended.md).


