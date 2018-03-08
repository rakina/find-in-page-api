# Find-in-page Improvements

Last updated 2018-03-08

## Introduction

There have been a lot of interest in improving the find-in-page experience from web developers, including:
* Giving the user a completely custom UI, suppressing the browser’s find UI, consistently
* Allowing data not in the DOM tree to be findable, even if there might be an infinite amount of data
* Reflecting real-time data loading/modification on find results
* Treating multiple webpages as one searchable instance, e.g. books, multi-page articles, or the multipage HTML spec 

To address these interests, we are presenting a collection of Web APIs to interact with or even completely override the browser’s find-in-page UI.
### Goals

* Allow web authors to give their custom find UI instead of the browser’s find UI consistently **(focus of this explainer)**
* Allow the browser’s find-in-page to find everything in the webpage that the web author deems as findable **(in [another document](explainer-extended.md))**
### Non-goals

* Extending the browser's built-in UI to find items that will never be in the current page's DOM, such as other chapters in a book, or other threads in a forum

## Providing a custom Find-in-page UI


### Example code

```js
// Intercepting find-in-page and completely replacing it with a custom UI
window.on('openfind', e => {
  e.preventDefault();
  const searchUI = generateSearchUI(); // details elided
  document.body.appendChild(searchUI);
  searchUI.querySelector('input[type="search"]').focus();
});
```

### `openfind` Event

When the user initiates action to open browser’s Find UI (by keypress or menu selection), the browser will first fire a DOM Event `openfind` (using the Event constructor). If the webpage cancels the event by calling `preventDefault()`, then the browser’s Find UI will not be shown. Otherwise, the browser's Find UI will be shown as usual.

### Use case

A web page has a completely custom way of loading data, and they want to provide a completely custom search UI. They can do this by adding an event listener for `openfind` Event, and calling `preventDefault()` on it, and then show their search UI instead.
```js
window.addEventListener("openfind", e => {
  e.preventDefault();
  showCustomUI();
});
```
*Note: currently there are a lot of websites that try to do this by detecting Ctrl+F, but they failed to detect when a user opens the Find UI from the browser’s menu (which is pretty much the only option in mobile).*


## Other Find-in-page APIs
Other than suppressing the browser's Find UI, there are some cases that we want to propose solutions for, such as making the browser wait for loading of data before proceeding a find action or adding to the browser's list of find results. As these cases are more complicated, we are making a separate document for those cases [here](explainer-extended.md).


