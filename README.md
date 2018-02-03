Motivation:

It's non-trivial to construct a nice DnD experience when your app uses iframes.

This should serve as a reference or a reminder to future me on how to do this nicely.

Method 1: Drag and drop with an iframe on a different origin. i.e:

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

iframe -> main:

1. When user starts dragging, send a message the main window to create an invisible (div) element
   and place it over the whole page.
2. When it's dropped,it's actually on our invisible div (instead of the iframe or the page).
3. But, you have to handle the dragend event in the iframe, and send the coords/data to the parent frame.
4. Inside the parent/main, remove the overlay, and use document.elementFromPoint(message.mouse.x, message.mouse.y)
   to work out where it's been dropped.
5. Now you have two choices. Either synthesise a drop event on the element you got in step 4. Or,
   depending on your application, just programatically trigger what would have happened if you'd done the drop.

Basically:

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

TODO: dragging from parent to iframe. (very similar, just in reverse);
