# supermemo-server-rfc

## SuperMemo Community Server API - RFC

### At a glance

This is a dual part primer and proposal for a system that can be implemented in SuperMemo, in fact already is in implemented -in a way- by the community. It relies on stable technologies which have remained the same for decades and demonstrates a future proof way to expose supermemo data outside of the application - without too much time and effort spent by the SuperMemo team. It could be touted as a major feature of supermemo and extensions built on top of it could generate a lot of buzz about supermemo in the learning community.

### Primer

At its core this proposal depends on communication between SM and user scripts running inside the browser embedded in supermemo. As it happens, this already exists in the form of the incremental video scripts. SuperMemo currently allows users to [load their own incremental video scripts](https://help.supermemo.org/wiki/Incremental_video#Your_own_incremental_video_script) when opening YT elements. When SM navigates to an element containing a YouTube HTML component, it loads the data using a file url in an embedded MSHTML page.

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

The first part contains DOM < input> elements that are rendered onto the page, SM needs these to work properly. These encode data used by the youtube player: the video id, the start time, the stop time, etc.

The second part tells the SM component to load HTML from a server and therefore whichever scripts attached to it from the supermemo server.

```html
<script type="text/javascript">
  // All of the code to play supermemo videos here!
</script>
```

A script loaded by youtube.supermemo.org/?...

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

These changes can be trialled and developed incrementally and potentially very easily. First off, you can trial the changes only for the existing YouTube.htm script. Secondly, you could provide data in the sUrlString and provide readonly DOM elements to indicate that SuperMemo will not read these values back if the user changes them.

```html
<input id="elementPriority" value="1.19" readonly hidden> // we will ignore changes!
```

This way, the SuperMemo team can populate as many readonly fields as are easy to implement. This will allow the community to start building tools that can interact with SuperMemo. The SuperMemo team can then add extras as is feasible.

Proposed fields:

- Collection name
- Element number
- Element's priority percentage
- Element's prioirity in the queue
- Number of repetitions
- Number of lapses
- Date of last rep
- Date of next rep
- Dismissed state
- Parent element number
- Previous sibling element number
- Element type (Topic, Item, etc)
- First review date

How user scripts can potentially be extended to generic elements and not just the YouTube component is discussed in the limitations section.

### Community User Script and Server

![alt](/supermemo-respons.png)

Many projects have community implementations of their software. In effect, we propose a Community Server that can be used to store and retrieve data from running SuperMemo instances. This would be a pair of user scripts and server maintained by the community. This server waits for user scripts running in supermemo to send information about the status of the application. That data can then be retrieved by clients connected to the server, thru API endpoints.

Example endpoints:

> POST /api/v1/collection/japanese/currentElement

```json
{
    "elementId": 123,
    "currentInterval": 67,
    "numReps": 6,
    "nextReviewDate": "Feb-17-2024",
    "elementPriority": 1.19,
    "dismissed": 0
}
```

> GET /api/v1/collection/japanese/currentElement

```json
{
    "elementId": 123,
    "currentInterval": 67,
    "numReps": 6,
    "nextReviewDate": "Feb-17-2024",
    "elementPriority": 1.19,
    "dismissed": 0
}
```

Example: updating dismissed state from user script

```javascript
const collection = "japanese"
document.getElementById("dismissed").value = "1"
fetch(`http://localhost:3000/api/v1/collection/${collection}/currentElement`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ dismissed: 1, ... }),
})
```

Now, other clients can read the dismissed state from the server or listen for changes to the current element. This can be used outside of SuperMemo to build tools that can interact with the software!

```javascript
const source = new EventSource('http://localhost:3000/api/v1/sub/collections/japanese/currentElement');
source.onmessage = function(event) {
  console.log('element changed! Currently visible element:', event.data.elementId);
};
```

### Non element data

Aside from data from the current element, there is the issue of retrieving collection data. The SM team may need to find a way to cleverly embed collection data into the DOM of the current element. Such as always reporting the collection name and the current element number with every request.

### Element component User Scripts

Currently, user scripts can only be loaded on youtube elements. YouTube elements are special since they can upgrade the HTML page to IE11 by including this in their HEAD.

```html
<meta http-equiv="X-UA-Compatible" content="IE=11">
```

The current suggestion is to make it so that user scripts can be loaded on generic Element components too. SuperMemo would use a similar system, loading a template from

> **<SUPERMEMO_PATH>\bin\ElemFront.htm

> **<SUPERMEMO_PATH>\bin\ElemBack.htm

> ... other element component types

### Reading Element content

It would be very helpful if user scripts can read element data. This should be possible by accessing document.body.innerHTML on scripts running on elements. This would allow user scripts to read the content of the element and send it to the server. The script can wait for updates from the server and then update the element content. Then users will be able to programatically send data to supermemo from outside the application:

user script [element component#0]

```javascript
const collection = ... // from the DOM
const elementId = ... // from the DOM
const elementContent = document.body.innerHTML
fetch(`http://localhost:3000/api/v1/collection/${collection}/currentElement`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ elementId, elementContent, ... }),
})
// wait for updates from the server
const source = new EventSource(`http://localhost:3000/api/v1/sub/collections/${collection}/element/${elementId}`);
source.onmessage = function(event) {
  console.log('client said hello');
  // update the element content!!
  document.body.innerHTML = event.data.elementContent;
};
```

client:

```javascript
const source = new EventSource('http://localhost:3000/api/v1/sub/collections/japanese/element/0');
source.onmessage = function(event) {
  console.log('Currently visible element content', event.data.elementContent);
  const elemContent = event.data.elementContent;
  fetch(`http://localhost:3000/api/v1/collection/japanese/element/0`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ elementContent: elemContent + ' hello', ... }),
  })
};
```

Thru this process apps outside of supermemo can read and write element content!

### Conclusion

This all transfers the responsibility of maintaining the server and user scripts to the community and allows the SuperMemo team to focus on the core application. The team needs to only provide data in the sUrlString, some DOM elements and (optionally) allow users to load Element components with template HTML! Notice that all the user scripts above can be written by the community.

## Limitations

### IE5 target for Element Components

Internally most element components currently target IE5. If it is not possible to upgrade the component to IE11, then it may be harder for the community to implement this proposal. A hidden iframe can be used to isolate the user scripts and data from editor components.

### Persistence

Since data is only updated when the user navigates to a new element, the data is not always in sync with what the user does or the state of the application. This can be considered a limitation of this approach.

In order to mitigate some side effects, I suggest the server requires a ping from the user script every 5 seconds to indicate that an element is currently visible. Data will be invalidated after this time or when a user navigates to a new element. Furthermore, if data changes are possible thru OOM communication then a request will only succeed if the user script verifies that it has set the data.

Another issue is that when SM updates the current element, lets say after a repetition, the user script will not be able to know what changes have occured.

### Reading the DOM in real time

It may be difficult for the SuperMemo engine to read or update the state of the DOM in real time. If this is difficult to implement, then it is not a huge problem since scripts can operate under an assumption that the data is only updated when the user navigates to a new element and can be invalid at any time.

### Future proofing

I don't expect this proposal to change much as tech changes. If Edge is embedded into supermemo then then approach will still work.

## Request for Open Source documentation

I would like to request that the SuperMemo team consider providing documentation or source code related to incremental video components and MSHTML, to help the community write scripts. When writing incremental video scripts sometimes I have run into issues where changes to the HTML have caused supermemo to act in unexpected ways that aren't yet publicly documented.

### Learn button disabled

For example, I found that SM needs the customPromptVisible string to be present in the DOM **on page load** or the Learn button cannot be pressed! It took some time to figure out that SM searches for it as a raw string and needs to be there before my react app renders. From this I was able to ascertain that supermemo does a string search on the entire document to find the exact line of text.

```html
    <div id="root">
      <!-- DO NOT DELETE
        <input type="hidden" value="0" id="customPromptVisible" />   <-- super important!

        SM NEEDS this customPromptVisible string or disables the Learn button
        SM does searches for this as a raw string so it can be commented out
      -->
    </div>
```

### Missing extract blue/yellow background

A blue/yellow background appeared in the past in SM18 and was used to indicate whether a video is an extract or topic. Somewhere along the way this stopped appearing and so I was left wondering if I had broken something or it was related to the stuff that broke about a year and a half ago with YouTube.

```html
#Link: http://www.youtube.com/watch?v=sAoYNO1eaW8
#Comment: References will be downloaded in a separate thread
```

Having access to some source code related to how the yellow/blue background is set could let the community find workarounds or fixes.

## Other findings

### Cross window communication

By opening a window in a user script, you can communicate with the window using the window.opener property. This can be used to communicate with the SuperMemo window from a user script running in a different window. For example, in the past I have used this to pop out the youtube player into a seperate window.

user script running in supermemo

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
