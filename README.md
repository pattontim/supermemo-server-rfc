# supermemo-server-rfc

## SuperMemo Community Server API - RFC

This document is WIP, but comments on Issues or Discord are welcome.

### At a glance

This is a dual part primer and proposal for a system that can be implemented in SuperMemo, in fact already is in implemented -in a way- by the community. It relies on stable technologies which have remained the same for decades and demonstrates a future proof way to expose supermemo data outside of the application - without too much time spent by the SuperMemo team.

### Primer

At its core this proposal depends on communication between SM and user scripts running inside the browser embedded in supermemo. As it happens, this already exists in the form of the incremental video scripts. SuperMemo (SM), currently allows users to [load their own incremental video scripts](https://help.supermemo.org/wiki/Incremental_video#Your_own_incremental_video_script) when opening YT elements. When SM navigates to an element containing a YouTube HTML component, it loads the data using a file url in an embedded MSHTML page.

> **file://<SUPERMEMO_PATH>/bin/YouTube.htm**?videoid=D3FDDPFrcQ0&resume=0:00&start=1:13&stop=1:20&ts=20241170165953

We will focus on file in the bolded text. If we look at the contents of this file we can see primarily two main parts...

```html
<input type="hidden" value="zzzYT-IDzzz" id="videoid" />
<input type="hidden" value="1" id="imposeboundries" />
<fieldset hidden>
    <input id="resumevideoat" value="..."/>
    <input id="startvideoat" value="..."/>
    <input id="stopvideoat" value="..."/>
    <input type="checkbox" checked="true" id="repeat"/>
    <select id="extracts"></select>
    <input value="0" id="customPromptVisible" />
    <select id="extracts"></select>
</fieldset>
```

```javascript
document.location.href = "https://youtube.supermemo.org/?" + sUrlString;
```

The first part contains DOM < input> elements that are rendered onto the page, SM needs these to work properly.  

The second part tells the SM component to load HTML and scripts from the supermemo server.

---

> ?**videoid**=D3FDDPFrcQ0&**resume**=0:00&**start**=1:13&**stop**=1:20&**ts**=20241170165953

Remember the rest of that URL? Well this contains all of the data needed by the youtube.supermemo.org/ script. It tells the page to populate all of the < input> elements with this data.

When the user navigates away from the youtube.supermemo.org/ page, SM reads the values in the < input> fields and updates the collection with that data.

```html
<input type="text" id="startvideoat" value={...start}/>
```

In this example, the element rendered in the DOM takes on the value 1:13. When the user changes the clip start time to say 2:34 then when they hit "Next Repetition", SM will read the value and update the clip start time. This presents an opportunity for communication as represented by the following diagram.

<!-- TODO diagram -->
![alt](/supermemo-data-flow.png)

## The proposal

In effect, the process of updating the element DOM values represent a way to communicate with SM using user scripts and persist data to the collection.

> ?**videoid**=D3FDDPFrcQ0&**resume**=0:00&**start**=1:13&**stop**=1:20&ts=20241170165953

So what if we place more data in the sUrlString?

> ...?&**currentInterval**=67&**numReps**=6&**nextReviewDate**=Feb-17-2024&**elementPriority**=1.19 ...

... And add more input fields?

```html
<fieldset hidden>
    <input id="resumevideoat" value="..."/>
    <input id="startvideoat" value="..."/>
    ...
    <input id="elementPriority" value="1.19">
    <input id="dismissed" value="0">

```

User scripts can then call

```javascript
document.getElementById("elementPriority").value = "2.20"
document.getElementById("dismissed").value = "1"
```

And then SuperMemo can now read a change in the DOM and update the element when the user navigates elements! Elements can have the "hidden" attribute so the user cannot see them, allowing us to cache lets of data.

### Incremental implementation [SM team]

These changes can be trialled and developed incrementally. First off, you can trial the changes only for the existing YouTube.htm script. Secondly, you could provide data in the sUrlString and provide readonly DOM elements to indicate that SuperMemo will not read these values back if the user changes them.

```html
<input id="elementPriority" value="1.19" readonly hidden> // we will ignore changes!
```

This way, the SuperMemo team can populate as many readonly fields as are easy to implement. This will allow the community to start building tools that can interact with SuperMemo.

Proposed fields:

- Element's priority percentage
- Element's prioirity in the queue
- Number of repetitions
- Number of lapses
- Date of last rep
- Date of next rep
- Dismissed state
- Element number
- Parent element number
- Previous sibling element number
- Element type (Topic, Item, etc)
- First review date

How user scripts can potentially be extended to generic elements and not just the YouTube component is discussed in the limitations section.

### Community User Script and Server

![alt](/supermemo-respons.png)

Many projects have community implementations of their software. In effect, we propose a Community Server that can be used to store and retrieve data from running SuperMemo instances. This would be a pair of user scripts and server maintained by the community. This server waits for user scripts running in supermemo to send information about the status of the applicatioo. That data can then be retrieved by clients connected to the server, thru API endpoints.

Example endpoints:

> POST /api/v1/currentElement

```json
{
    "currentInterval": 67,
    "numReps": 6,
    "nextReviewDate": "Feb-17-2024",
    "elementPriority": 1.19,
    "dismissed": 0
}
```

> GET /api/v1/currentElement


```json
{
    "currentInterval": 67,
    "numReps": 6,
    "nextReviewDate": "Feb-17-2024",
    "elementPriority": 1.19,
    "dismissed": 0
}
```

Example: updating dismissed state from user script

```javascript
document.getElementById("dismissed").value = "1"
fetch('http://localhost:3000/api/v1/currentElement', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ dismissed: 1 }),
})
```

Now, other clients can read the dismissed state from the server or listen for changes to the current element. This can be used outside of SuperMemo to build tools that can interact with software!

```javascript
const source = new EventSource('http://localhost:3000/api/v1/sub/currentElement');
source.onmessage = function(event) {
  console.log('element changed!', event.data);
};
```

This transfers the responsibility of maintaining the server to the community and allows the SuperMemo team to focus on the core application. The team needs to only provide data in the sUrlString and some DOM elements!

### Non elrment data

Aside from data from the current element, there is the issue of retrieving collection data. The SM team may need to find a way to cleverly embed collection data into the DOM of the current element.

## Limitations

### Element component User Scripts

Currently, user scripts can only be loaded on youtube elements. YouTube elements are special since they can upgrade the HTML page to IE11 by including this in their HEAD.

```html
<meta http-equiv="X-UA-Compatible" content="IE=11">
```

The current suggestion is to make it so that user scripts can be loaded on generic Element components too. SuperMemo would use a similar system, loading a template from

> **<SUPERMEMO_PATH>\bin\Elem.htm

This would allow the community to implement user scripts for this proposal for every component in SuperMemo. However, most element components currently target IE5. If it is not possible to upgrade the component to IE11, then it will be significantly harder for the community to implement this proposal. An iframe can be used to isolate the user script from editor components.

### Persistence

Since data is only updated when the user navigates to a new element, the data is not always in sync with what the user does or the state of the application. This can be considered a limitation of this approach.

In order to mitigate some side effects, I suggest the server requires a ping from the user script every 5 seconds to indicate that an element is currently visible. Data will be invalidated after this time or when a user navigates to a new element. Furthermore, if data changes are possible thru OOM communication then a request will only succeed if the user script verifies that it has set the data.

Another issue is that when SM updates the current element, lets say after a repetition, the user script will not be able to know what changes have occured.

### Future proofing

I don't expect this proposal to change much as tech changes. If Edge is embedded into supermemo then then approach will still work.

## Other findings

### Cross window communication

By opening a window in a user script, you can communicate with the window using the window.opener property. This can be used to communicate with the SuperMemo window from a user script running in a different window. For example, in the past I have used this to pop out the youtube player into a seperate window.

user script

```javascript
window.open('http://localhost:3000/static/hello-world.html');
window.addEventListener('message', function(event) {
  console.log('received message', event.data);
});
```

hello-world.html

```javascript
window.opener.postMessage('hello world', '*');
```
