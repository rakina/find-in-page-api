# Find-in-page Improvements
Last updated 2018-03-08
## What’s all this then?

There have been a lot of interest in improving the find-in-page experience from web developers, including:
* Giving the user a completely custom UI, suppressing the browser’s find UI, consistently
* Allowing data not in the DOM tree to be findable, even if there might be an infinite amount of data
* Reflecting real-time data loading/modification on find results
* Treating multiple webpages as one searchable instance, e.g. books, multi-page articles, or the multipage HTML spec 

To address these interests, we are presenting a collection of Web APIs to interact with or even completely override the browser’s find-in-page UI.
### Goals

* Allow the browser’s find-in-page to find everything in the webpage that the web author deems as findable
* Allow web authors to give their custom find UI instead of the browser’s find UI consistently
### Non-goals

* Extending the browser's built-in UI to find items that will never be in the current page's DOM, such as other chapters in a book, or other threads in a forum

## Example code

```js
// Intercepting find-in-page and completely replacing it with a custom UI
window.on('openfind', e => {
  e.preventDefault();
  const searchUI = generateSearchUI(); // details elided
  document.body.appendChild(searchUI);
  searchUI.querySelector('input[type="search"]').focus();
});


// Listen to find events and make the UI wait for loading of data
window.addEventListener(‘findinginpage’, e => {
  if (e.findNext) {
    e.waitUntil(loadTweetsBefore(bottomTweet));
  } else if (e.findPrev) {
    e.waitUntil(loadNewTweets());
  }
});

// TODO: add example for setting deferred results?

```
## Key scenarios
### Give web page’s custom find UI instead of browser’s find UI

A web page has a completely custom way of loading data, and they want to provide a completely custom search UI. They can do this by adding an event listener for `openfind` Event, and calling `preventDefault()` on it, and then show their search UI instead.
```js
window.addEventListener("openfind", e => {
  e.preventDefault();
  showCustomUI();
});
```
*Note: currently there are a lot of websites that try to do this by detecting Ctrl+F, but they failed to detect when a user opens the Find UI from the browser’s menu (which is pretty much the only option in mobile).*

### Want to allow browser to search data not currently in the DOM tree by loading the data

A web page doesn’t have all the data currently in the DOM tree detects a find event, and want to load more data so that the browser’s built-in search can find the data.

```js
window.addEventListener("findinginpage", e => {
  e.waitUntil(loadMoreDataIntoDOM());
});
```

### The DOM has been modified since the start of the last find query

New content has been added/some content has been modified on the DOM, and the web page wants the browser to redo the built-in search for the currently running/recently finished find query, so that the results reflect the current situation of the DOM.
```js
window.navigator.redoFind()
```
###  The web page wants to supplement results to the browser’s find results

A web page wants to give additional search results to the browser, possibly due to not wanting to load more data into the DOM (maybe when there is an infinite amount of data)
```js
window.addEventListener("findinginpage", e => {
  const offScreenResult = new FindInPageResult({
    async goToCallback() {
      await myInfiniteList.scrollTo(3000);
      // Browser will automatically re-search when this returns,
      // and find the result in the now-updated DOM
    },
    estimatedScrollPosition: 90 // out of 100??
  });
  
  e.setDeferredResults([offScreenResult]);
});
```

###  A search engine wants to find a certain search query, but not all data is in the DOM (due to lazy rendering, etc)

For search engines that use a full headless browser for their crawling (in order to, e.g., execute JavaScript), the crawler can invoke the headless browser's find-in-page functionality. The web page will use these APIs in the fashion described above, loading the appropriate results. The crawler can then index the DOM. The crawler can then repeat this process, using the browser's find-next-result functionality and reindexing until all results are found.

## APIs/Solutions
### Notify web page that user wants to open the Find UI, and allow suppressing

When the user initiates action to open browser’s Find UI (by keypress or menu selection), the browser will first fire a DOM Event “openfind” (using the Event constructor). If the webpage cancels the event, by calling preventDefault(), then the browser’s Find UI will not be shown.

### Notify web page of user’s action in browser’s Find UI, and allow them to delay action until new data is loaded

When the browser’s Find UI receives some user action such as find next or update to the find query, the browser will fire a DOM Event `findinginpage`. The event provides the method `waitUntil(promise)` that will make the action associated with the event to wait until `promise` is settled. 

`findinginpage` event have these attributes:
 * `searchString`: corresponds to the search string typed into the browser’s find UI
 * `findNext`: boolean value, true if this is a find next action
 * `findPrev`: boolean value, true if this is a find previous action

Additionally it may also have the attributes `caseSensitive` and `wholeWordsOnly` that correspond to the value of the options in the browser’s find UI. The event will also have a signal property, which is an AbortSignal that gets signaled if the user cancels the search (including by initiating a new search, which may happen by changing the search string, or changing any of the options)

### Allow web page to send find results to browser

Additional methods on `findinginpage` event: `setDeferredResultsBeforeFirst(findResults)` and `setDeferredResultsAfterLast(findResults)`, where `findResults` is an array of `FindInPageResult`s.

`FindInPageResult` have these attributes:
* `callback`: the function to be called when we want to show this result (when the user calls find prev/next until it reaches this find result)
* `estimatedScrollPosition`: the estimated scroll position of where the result is in the page


Calling `setDeferredResults{BeforeFirst,AfterLast}` with a list of `FindInPageResult`s will add them to the browser’s find-in-page results, before the first result and after the last result, respectively. When the find result needs to be shown, `callback` is called.

### Allow web page to notify browser that new content has been loaded/modified

`Window.navigator.redoFind()` can be called to notify browser that some content has been loaded/modified, and the browser should redo finding.

*Note: we might actually not need an API for this, and just improve browsers internally so that it will redo find in page if it’s idle and detected change in the DOM (though currently Chrome, Safari and Firefox does not redo find when DOM content is updated)*
