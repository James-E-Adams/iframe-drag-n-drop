# Drag/Drop between iframes and parent(s).

It's non-trivial to construct a nice DnD experience when your app uses iframes.

This should serve as a reference or a reminder to future me on how to do this nicely.

## Use case: Drag and drop with an iframe on a different origin

        |------------------------------------------------------|
        |main app on good.com                                  |
        |                                                      |
        |                                                      |
        |                                                      |
        |           |------------------------|                 |
        |           |     iframe on evil.com |                 |
        |           |                        |                 |
        |           |                        |                 |
        |           |                        |                 |
        |           |                        |                 |
        |           |                        |                 |
        |           |------------------------|                 |
        |                                                      |
        |                                                      |
        |                                                      |
        |                                                      |
        |                                                      |
        |------------------------------------------------------|

(apologies for the sub optimal ascii art )

So it goes into it a little here, https://bugs.chromium.org/p/chromium/issues/detail?id=251718

But because of certain cross-origin security issues, you can't drag and drop between the iframe
and the root document without some extra work.

We needed this exact functionality on a project I've been working on, but it took a fair bit of research
before I found a reasonably clean way to do this.


## iframe -> parent

1. When user starts dragging, send a message the main window to create an invisible (div) element
   and place it over the whole page.
2. When it's dropped, it's actually on our invisible div (instead of the iframe or the page).
3. But, you have to handle the dragend event in the iframe, and send the coords/data to the parent frame.
4. Inside the parent/main, remove the overlay, and use document.elementFromPoint(message.mouse.x, message.mouse.y)
   to work out where it's been dropped.
5. Now you have two choices. Either synthesise a drop event on the element you got in step 4. Or,
   depending on your application, just programatically trigger what would have happened if you'd done the drop.

### Sample code

```js
/** Iframe code **/
//say you have this div inside your iframe that looks like this:
let elementInsideIframe = "<div draggable='true'>";
elementInsideIframe.addEventListener("dragstart", event => {
  //send message to parent somehow.
  //usually this can be done with https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage
  window.parent.postMessage("dragging!", "*", false);
  //The last arg can be a Transferable Object, but I haven't used this before.
  //https://developer.mozilla.org/en-US/docs/Web/API/Transferable
});
//Also handle the 'dragend' event:
elementInsideIframe.addEventListener("dragend", event => {
  //send message to parent again.
  const data = {
      mouse:{
        x: event.pageX,
        y: event.pageY,
      }),
     textPayload: 'hey man!',
  }
  window.parent.postMessage(JSON.stringify(data), "*", false);
});
```

Now inside the main window:

```js
function receiveMessage(event)
{
    //maybe check the origin here before you look at the messages
    //and confirm you trust the sender.

    //Handle string messages
    switch(event.data) {
        case 'dragging!':
            //set up a div overlay
            const overlay = document.createElement("div");
            overlay.className = "overlay";
            overlay.innerHTML =
                '<div class="text"> Drag and drop somewhere! </div>';
            const domInsertionPoint = document.getElementsByClassName(
                "wherever-you-want-to-insert-the-overlay"
            )[0];
            domInsertionPoint.appendChild(overlay);
        //maybe handle some other string messages here:
        default:
            return;
    }
    //handle the data type messages
    try {
        const data = JSON.parse(event.data);
        if (typeof data = 'object' && data.hasOwnProperty('mouse')) {
            document.getElementsByClassName("overlay")[0].remove();
            const syntheticEvent = new DragEvent("drop");
            // Override dataTransfer first so we can define our own.
            Object.defineProperty(syntheticEvent, "dataTransfer", {
                value: {}
            });
            // Once dataTransfer is overridden, you can define getData.
            syntheticEvent.dataTransfer.getData = dragFormatRequested => {
                // only handling text atm.
                if (["text", "text/html"].indexOf(dragFormatRequested) > -1) {
                    return data.textPayload;
                }
            };
            const target = document.elementFromPoint(data.mouse.x, data.mouse.y)
            Object.defineProperty(syntheticEvent, "target", {
                value: target
            });
            target.dispatchEvent(syntheticEvent);
        }
    } catch(error) {
        //error was probably SyntaxError for trying to parse something
        //that isn't JSON.
    }
}
//listen for the message from the child.
window.addEventListener("message", receiveMessage, false);
```

Example styling for the overlay
```css
.overlay {
    position: fixed;
    display: block;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: pink;
    z-index: 50;
    opacity: 0.3;

    .text {
        position: absolute;
        top: 50%;
        left: 50%;
        font-size: 50px;
        color: green;
        transform: translate(-50%, -50%);
    }
}
```

## Parents -> iframe
Very similar concept. A method I found pretty reasonable was to put a `div` as a sibling for each iframe. This div acts in a similar manner as the previous overlay method, but you have an overlay for each iframe rather than over the whole viewpoint. 

So keeping the iframe cover div in the dom from the time the iframe exists, style it like: 

```css
.iframe-cover {
    position: absolute;
    top: -10px;
    left: -10px;
    right: -10px;
    bottom: -10px;
    z-index: 3; // This may need to be larger depending on the other z-indices in your application.
    display: none;
    background-color: rgba(255, 255, 255, 0.01);
}
```

And then for the element you're dragging:

```javascript
element.addEventListener("dragstart", event => {
        // display the overlay. Using JQ but can be done with web apis.
        $(".iframe-cover").css({'display':'block'})
        // other logic for this event
       });
       
element.addEventListener("dragend", event => {
        // remove the overlay after drag ends.
        $(".iframe-cover").css({'display':'none'})
        // other logic for this event
       });       
```

And wherever you injected the overlay into the dom, let's assume you have a reference to it as `iframeCover`:
```javascript
   iframeCover.addEventListener("drop", event => {
        // grab whatever you need from the event dataTransfer
        // send it to the iframe via postMessage similar to above (but parent -> child)
   })
```

You can also use this method for supporting drag/drop between two different iframes (on different domains). 
You just have to make sure you trigger the overlays to display when the parent receives a drag-start message from an iframe.


