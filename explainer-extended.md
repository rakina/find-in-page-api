# Other Find-in-page Improvements

## Introduction

As mentioned in the [main explainer](README.md), web authors have expressed interest to improve the find-in-page experience. Some ideas such as allowing regex queries, or finding in other pages in the website as well is better suited when the web authors to make their own find-in-page UI and utilize the API described in the main explainer. Some cases, however, might not need a completely custom UI. This explainer describes those cases and our proposed APIs to them.


## Example code

```js
// Listen to find events and make the UI wait for loading of data
window.addEventListener("findinginpage", e => {
  if (e.direction == "prev" && e.currentActiveMatchIndex <= LOAD_MATCHES_THRESHOLD) {
    // loadNewTweets adds new tweets to the DOM tree.
    e.waitUntil(loadNewTweets());
  } else if (e.direction == "next" && e.currentActiveMatchIndex >= e.totalMatches - LOAD_MATCHES_THRESHOLD) {
    // loadTweetsBefore adds new tweets to the DOM tree.
    e.waitUntil(loadTweetsBefore(bottomTweet));
  }
});

// Add additional results to browser's find-in-page results
window.addEventListener("findinginpage", e => {
  const offScreenResult = new FindInPageResult({
    async goToCallback() {
      // We have the virtualized infinite list in memory currently, and found that the item in index 3000 should be loaded
      // We can utilize e.searchString, e.caseSensitive, e.wholeWordsOnly to determine which items need to be loaded
      await myInfiniteList.scrollTo(3000);
      // Browser will automatically re-search when this returns,
      // and find the result in the now-updated DOM
    },
    estimatedScrollPosition: 90 // out of 100??
  });
  
  e.addResultsAfterLast([offScreenResult]);
});

// Notify browser to redo last find-in-page query when new data is loaded
<iframe id="myframe" src="..." onload="window.navigator.redoFind()"></iframe>
```

## Key scenarios


### Want to load more data into the DOM before browser searches the DOM

A web page detects that browser's find-in-page activity and has some more data that is not in the DOM tree. The web page wants to load more data to the DOM tree before the find action is done so that the browser’s built-in search can find the data when it detects a find event. The data can be in memory and just need to be added to the DOM tree, or the data can also be fetched from the server. 

For example, `loadNewTweets` in the first example code fetches new tweets from the server and adds them to the DOM tree. The find-in-page action is delayed until it's done loading the new tweets.

### Want to supplement results to the browser’s find results, but not load them to DOM until needed

A web page wants to give additional search results to the browser, but not load them to the DOM yet. The web page can know what things to load to the DOM when it's needed by doing its own calculation to the data it has in memory. The web page can provide the result with a callback function that loads the relevant data. The callback function is called only when needed (when the user wants to see the result, by pressing prev/next until it reaches the index of the added result). After the callback is done, the browser re-searches. This is useful when we really don't want to add too much into the DOM until it's really needed, such as when we have a virtualized infinite list.

For example, the `callback` in `offScreenResult`is called when the browser have hit the end of its original results and the infinite list scrolls to index 3000, adding the relevant data into the DOM. The browser then re-searches with the updated DOM.

### The DOM has been modified since the start of the last find query

New content has been added/some content has been modified on the DOM, and the web page wants the browser to redo the built-in search for the currently running/recently finished find query, so that the results reflect the current situation of the DOM.



###  A search engine wants to find a certain search query, but not all data is in the DOM (due to lazy rendering, etc)

For search engines that use a full headless browser for their crawling (in order to, e.g., execute JavaScript), the crawler can invoke the headless browser's find-in-page functionality. The web page will use these APIs in the fashion described above, loading the appropriate results. The crawler can then index the DOM. 


## APIs

### Notify web page of user’s action in browser’s Find UI, and allow them to delay action until new data is loaded

When the browser’s Find UI receives some user action such as find next or update to the find query, the browser will fire a DOM Event `findinginpage`. The event provides the method `waitUntil(promise)` that will make the action associated with the event to wait until `promise` is settled. 

`findinginpage` event have these attributes:
 * `searchString`: corresponds to the search string typed into the browser’s find UI
 * `findNext`: boolean value, true if this is a find next action
 * `findPrev`: boolean value, true if this is a find previous action

Additionally it may also have the attributes `caseSensitive` and `wholeWordsOnly` that correspond to the value of the options in the browser’s find UI. The event will also have a signal property, which is an AbortSignal that gets signaled if the user cancels the search (including by initiating a new search, which may happen by changing the search string, or changing any of the options)

### Allow web page to send find results to browser

Additional methods on `findinginpage` event: `addResultsBeforeFirst(findResults)` and `addResultsAfterLast(findResults)`, where `findResults` is an array of `FindInPageResult`s.

`FindInPageResult` have these attributes:
* `callback`: the function to be called when we want to show this result (when the user calls find prev/next until it reaches this find result)
* `estimatedScrollPosition`: the estimated scroll position of where the result is in the page

Calling `addResults{beforeFirst,afterLast}` with a list of `FindInPageResult`s will add them to the browser’s find-in-page results, before the first result and after the last result, respectively. When the find result needs to be shown (when active match is nearing the result), `callback` is called and the browser redoes the find-in-page.

### Allow web page to notify browser that new content has been loaded/modified

`window.navigator.redoFind()` can be called to notify browser that some content has been loaded/modified, and the browser should redo finding.

*Note: we might actually not need an API for this, and just improve browsers internally so that it will redo find in page if it’s idle and detected change in the DOM (though currently Chrome, Safari and Firefox does not redo find when DOM content is updated)*
