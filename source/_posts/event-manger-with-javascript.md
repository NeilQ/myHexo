title: Use Event-Driven with javascript
date: 2014-12-14 18:58:56
tags:
- JavaScript
categories:
- JavaScript
---

Let me recommend using publish-subscribe pattern to refactor JavaScvript codes in Web development, base on Javascript Event Driven.

## Goal

> * To manage specific JavaScript method\events for web application as custom events, make it to high level.
> * To reduce the dependency between custom events and UI controls, decouple JavaScript codes.
> * To make UI codes more readable, maintainable.


## How to user Event Manager

**The eventManager codes is defined below:**
```javascript
var eventManager = {
    subscribe: function (eventType, handler) {
        $(window).on(eventType, handler);
    },
    notify: function (eventType, e) {
        if ((e == null) || (e.length === "undefined") || typeof e === "string") {
            $(window).trigger(eventType, e);
        } else {
            $(window).trigger(eventType, [e]);
        }
    }
}
```

**Define a JSON object “MyEvents” to store the events names:**
```javascript
var MyEvents = {
    NodeSelected: 'myEvent_NodeSelected',
    NodeCopied: 'myEvent_NodeCopied',
    NodeMoved: 'myEvent_NodeMoved'
}
```
Once we add a custom event, we should add a member to this object.

**Subscribe events at right place, usually in initialize method of a module:**
```javascript
MyFeature.Init = function () {  
    /* register for click event */
    eventManager.subscribe(MyEvents.NodeSelected, function (sender, args) {
        // do something here…
		console.log(args.name);
    });
 }
```

**Notify the event when needed:**
```javascript
function processNode(e) { 
        var eventDataItem = $("#treeView").data('kendoTreeView').dataItem(e);
        if ((eventDataItem) &&
            (eventManager)) {
            eventManager.notify(MyEvents.NodeSelected, eventDataItem);
        }
    }
```
We use extra parameter to communicate with the event handler, and we should avoid using an html element as the parameter to reduce the dependency with UI control.

## Notes
1.It is JavaScript publish-subscribe pattern rather than a feature, and it is good to follow the idea by using eventManager in further UI development.
2.There must be lots of custom events managed in the future, and we may be confused with them, so it’s a good idea to write some annotation to the specific event. Like below:
```javascript
var MyEvents = {
    /*
     * Happened after the tree node be selected in TreeView control.
     * Keep it consistency with the treeview of page1, page2 and page3
     */
    NodeSelected: 'myEvent_NodeSelected',
 
    // Happend after the node been copied
    NodeCopied: 'myEvent_NodeCopied',
```

3.The codes in handler method must be defensive.
