# Defining Composed for Selection API in Composed Tree

Author: dizhang168

Last updated: Jan 22, 2025

## Motivation

The current specifications for implementing Selection API's getComposedRanges() says to use the range associated with the Selection, with the implication that this range behaves like a live range. However, a live range is defined as an object that implements Range, which only [contains](https://dom.spec.whatwg.org/#contained) nodes that have the same root as the range’s start node. If so, the live range cannot have endpoints that cross different trees (i.e. the shadow boundaries).

In DOM specification, we use the word “composed” to mean an object that crosses the shadow boundaries. For example, a click event’s composedPath() returns an array of EventTarget, propagated across shadow boundaries. When the Selection API’s getComposedRanges() accesses the associated range's start and end, it is not accessing start and end that cross boundaries. We propose fixing this by adding a new **Composed Live Range** concept.

## The Current Spec

In the Selection API, the range associated with a selection is defined as:

“Each [selection](https://w3c.github.io/selection-api/#dfn-selection) can be associated with a single [range](https://dom.spec.whatwg.org/#concept-range). When there is no [range](https://dom.spec.whatwg.org/#concept-range) associated with the [selection](https://w3c.github.io/selection-api/#dfn-selection), the selection is empty. The selection must be initially [empty](https://w3c.github.io/selection-api/#dfn-empty).” [1]

This range is an object implementing an AbstractRange. [2]

Further, this associated range (and hence the selection) is affected by mutations:

“... the user agent must update the [range](https://dom.spec.whatwg.org/#concept-range) associated with [selection](https://w3c.github.io/selection-api/#dfn-selection) of the [node document](https://dom.spec.whatwg.org/#concept-node-document) of the [node](https://dom.spec.whatwg.org/#concept-node) as if it's a [live range](https://dom.spec.whatwg.org/#concept-live-range).” [3]

However, a live range itself is defined as any object implementing a Range. [4]

Further, the steps to “set the start or end” of a range restricts that the start and end endpoints to have the same root: [5]

- “If range’s root is not equal to node’s root, or if bp is after the range’s end, set range’s end to bp.”
- “If range’s root is not equal to node’s root, or if bp is before the range’s start, set range’s start to bp.”

Hence, we conclude the specifications as it is currently does not support crossing shadow boundaries.

[1] [https://w3c.github.io/selection-api/#dfn-selection](https://w3c.github.io/selection-api/#dfn-selection)

[2] [https://dom.spec.whatwg.org/#concept-range](https://dom.spec.whatwg.org/#concept-range)

[3] [https://w3c.github.io/selection-api/#responding-to-dom-mutations](https://w3c.github.io/selection-api/#responding-to-dom-mutations)

[4] [https://dom.spec.whatwg.org/#concept-live-range](https://dom.spec.whatwg.org/#concept-live-range)

[5] [https://dom.spec.whatwg.org/#concept-range-bp-set](https://dom.spec.whatwg.org/#concept-range-bp-set)

### What UA implementers do

To go around this problem, UA implementations have defined an internal concept of frame selections to keep track of the endpoints. In Blink, the Selection is associated with a FrameSelection, which itself stores a cached range:

[https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/core/editing/frame_selection.cc;l=1471;drc=5c5b42912e0f2ba99834acacaaef6b6f07e4ce42](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/core/editing/frame_selection.cc;l=1471;drc=5c5b42912e0f2ba99834acacaaef6b6f07e4ce42)

Then, while the cached live range tracks start and end to have the same root, the frame selection can track endpoints with different roots (while staying within the document).

Webkit/Gecko do something similar by being associated to a frame selection’s live range:

[https://github.com/WebKit/WebKit/blob/55d789ff3bf87148459a5bf12aa2b2c5ab547d56/Source/WebCore/page/DOMSelection.cpp#L77](https://github.com/WebKit/WebKit/blob/55d789ff3bf87148459a5bf12aa2b2c5ab547d56/Source/WebCore/page/DOMSelection.cpp#L77)

[https://searchfox.org/wubkat/source/Source/WebCore/page/DOMSelection.cpp#414](https://searchfox.org/wubkat/source/Source/WebCore/page/DOMSelection.cpp#414)

## Proposal

Issue: [https://github.com/w3c/selection-api/issues/2](https://github.com/w3c/selection-api/issues/2)

In this document, we propose adding text to the DOM and Selection API specification so this internal concept is better defined across user agents.

First, we need to redefine what a live range is and newly define the internal concept of a Composed live range:

```txt
A live range is a range that is affected by mutations to the node tree.
Objects implementing the Range interface are live ranges.

A composed live range is a live range that has one associated Range object cached live range.

Note: The cached live range is used to maintain backward compatibility for the getRangeAt() API.
```

### How Mutation works

Issue: [https://github.com/w3c/selection-api/issues/168](https://github.com/w3c/selection-api/issues/168)

**To insert a node into a _parent_ before a child**

[https://dom.spec.whatwg.org/#concept-node-insert](https://dom.spec.whatwg.org/#concept-node-insert)

Insertion does not change the range endpoint nodes. It might change the offset value:

- If the inserted node is within the range and a sibling of end node, end node’s offset might increase.
- If the inserted node is before and a sibling of start node, start node’s offset might increase.

In either case, there is no tree traversal and crossing of shadow boundaries. Nodes within a shadow tree will only have parents within the shadow tree.

For example:

```js
container.innerHTML = 'a<div id="host"></div>b';
const host = container.querySelector("#host");
const shadowRoot = host.attachShadow({ mode: "open" });
a = document.createTextNode("A");
b = document.createTextNode("B");
c = document.createTextNode("C");
d = document.createTextNode("D");
shadowRoot.appendChild(a);
shadowRoot.appendChild(b);
shadowRoot.appendChild(c);

selection = getSelection();
selection.setBaseAndExtent(container, 0, shadowRoot, 3);
shadowRoot.insertBefore(d, c);
selection.getComposedRanges({ shadowRoots: [shadowRoot] })[0];
```

The endContainer is the shadow root and the endOffset is now 4.

**To remove a node**

[https://dom.spec.whatwg.org/#concept-node-remove](https://dom.spec.whatwg.org/#concept-node-remove)

This mutation includes ancestor traversal so we need to update it for Composed Live Range.

For example:

```html
<!DOCTYPE html>
<div id="container"></div>

<script>
  const sel = getSelection();
  container.innerHTML = 'a<div id="outerhost"></div>b';
  const outerHost = container.querySelector("#outerhost");
  const outerRoot = outerHost.attachShadow({ mode: "open" });
  outerRoot.innerHTML = 'c<div id="innerHost"></div>d';
  const innerHost = outerRoot.querySelector("#innerHost");
  const innerRoot = innerHost.attachShadow({ mode: "open" });
  innerRoot.innerHTML = "hello, world";

  sel.setBaseAndExtent(container.firstChild, 0, innerRoot.firstChild, 4);
  outerHost.remove();
  const composedRange = sel.getComposedRanges({ shadowRoots: [innerRoot, outerRoot] })[0];
  // end node is rescoped to {container, 1}

  const liveRange = sel.getRangeAt(0);
  // end node is still {innerRoot.firstChild, 4}
</script>
```

The Selection has a composed live range with start node {container.firstChild, 0} and end node {innerRoot.firstChild, 4} and a cached live range that is collapsed at {innerRoot.firstChild, 4}.

When we call outerHost.remove():

- Cache live range’s start and end nodes are not inclusive descendant of outerHost; no mutation changes.
- Composed live range’s endNode is not an inclusive descendant of innerRoot nor innerRoot’s parent, can ignore step 5 and 7.
- Composed live range’s endNode is a shadow-inclusive descendant of innerRoot. Its end should be set to parent of outerHost, container and its offset should be 1.

```txt

8. For each composed live range whose start node is a shadow-including inclusive descendant of node, set its start to (parent, index).

9. For each composed live range whose end node is a shadow-including inclusive descendant of node, set its end to (parent, index).

```

**To normalize**

[https://dom.spec.whatwg.org/#dom-node-normalize](https://dom.spec.whatwg.org/#dom-node-normalize)

No change necessary since currently, calling normalize() on a shadow host is not expected to cross shadow boundaries.

For example:

```js
container.innerHTML = 'a<div id="host"></div>b';
const host = container.querySelector("#host");
const shadowRoot = host.attachShadow({ mode: "open" });
one = document.createTextNode("Part 1 ");
two = document.createTextNode("Part 2 ");
shadowRoot.appendChild(one);
shadowRoot.appendChild(two);

getSelection().setBaseAndExtent(one, 2, two, 2);

container.normalize(); // no effect
console.log(container);

shadowRoot.normalize(); // combine one and two together, selection still stays the same
console.log(shadowRoot);
```

**To replace CharacterData**

[https://dom.spec.whatwg.org/#concept-cd-replace](https://dom.spec.whatwg.org/#concept-cd-replace)

No change necessary since we are comparing the live range’s endpoints with the node to replace only. There are no boundaries to cross in this check.

For example:

```js
one = document.createTextNode("Part 1 ");
two = document.createTextNode("Part 2 ");

const shadowRoot = host.attachShadow({ mode: "open" });
shadowRoot.appendChild(one);
shadowRoot.appendChild(two);

getSelection().setBaseAndExtent(one, 2, two, 2);

two.replaceData(1, 4, "replaced");
```

The above selection is the same as if we did:

```js
one = document.createTextNode("Part 1 ");
two = document.createTextNode("Part 2 ");

host.appendChild(one);
host.appendChild(two);

getSelection().setBaseAndExtent(one, 2, two, 2);

two.replaceData(1, 4, "replaced");
```

**To split a Text node**

[https://dom.spec.whatwg.org/#concept-text-split](https://dom.spec.whatwg.org/#concept-text-split)

No change necessary since a shadow host cannot be a text node. Further, if this text node is in a shadow root, its parent is also within the shadow tree or is the shadow root.

For example:

```js
const shadowRoot = host.attachShadow({ mode: "open" });
shadowRoot.innerHTML = "ABCDE";

getSelection().setBaseAndExtent(shadowRoot.firstChild, 0, shadowRoot.firstChild, 3);
newNode = shadowRoot.firstChild.splitText(2);

getSelection().getComposedRanges({ shadowRoots: [shadowRoot] })[0]; // endContainer is CDE and endOffset is 1
```

### Interaction between a Range and a Selection

Issue: [https://github.com/whatwg/dom/issues/772](https://github.com/whatwg/dom/issues/772)

A cached live range’s changes should be directly reflected on the composed range. This means that when a user uses the Range API to call setStart/setEnd, it should also update the Selection’s anchor and focus. This principle should be respected for Composed Live Range.

Here is an interesting example I shared on the [github issue](https://github.com/whatwg/dom/issues/772#issuecomment-2491887033):

```html
<html>
  <body>
    <div id="light">Start outside shadow DOM</div>
    <div id="outerHost">
      outerHost
      <template shadowrootmode="open">
        <slot></slot>
        <div id="innerHost">
          innerHost
          <template shadowrootmode="open">
            <slot></slot>
          </template>
        </div>
      </template>
    </div>

    <script>
      selection = getSelection();
      outerHost = document.getElementById("outerHost");
      outerRoot = outerHost.shadowRoot;
      innerHost = outerRoot.getElementById("innerHost");

      // Step 1
      selection.setBaseAndExtent(light.firstChild, 10, innerHost.firstChild, 5);

      // Step 2
      range = selection.getRangeAt(0);
      range.setEnd(innerHost.firstChild, 6);
    </script>
  </body>
</html>
```

After step 1,

- Composed range should be at start{light.firstChild, 10} and end{innerHost.firstChild, 5}
- Live range is collapsed because it is across tree scopes, at start{innerHost.firstChild, 5} and end{innerHost.firstChild, 5}

After step 2,

- Live range is changed to start{innerHost.firstChild, 5} and end{innerHost.firstChild, 6}

**Option 1**: If we want the Composed Live Range to be updated to the same as its cached Live Range, we should change the spec:

```txt
Change "To set the boundary point (node, offset) of a Range range, run these steps:"
https://dom.spec.whatwg.org/#concept-range-bp-set

Add new step 5:
If range is associated with a composed live range,
Set composed live range's start and end to range's start and end.
```

Then, composed live range’s start will be {innerHost.firstChild, 5} and end will be {innerHost.firstChild, 6} as well.

This option is better if we want to keep consistency between Range ⇔ Selection.

**Option 2**: If we want the Composed Live Range to only update one endpoint at a time, then we should change the specification to:

```txt
Change "To set the boundary point (node, offset) of a Range range, run these steps:"
https://dom.spec.whatwg.org/#concept-range-bp-set

Add in step 4 part If these steps were invoked as "set the start":
3. If range is associated with a composed live range, set composed live range's start to range's start.
Add in step 4 part If these steps were invoked as "set the end":
3. If range is associated with a composed live range, set composed live range's end to range's end.
```

Then, composed live range’s start will stay as {light.firstChild, 10}, but end node will become {innerHost.firstChild, 6}.

This option is better if we want to avoid the side effect of a range.setStart might cause the selection.focusNode to change unexpectedly.

### Composed valid StaticRange and Composed position

Issue: [https://github.com/w3c/selection-api/issues/169](https://github.com/w3c/selection-api/issues/169)

Currently, by definition, the Selection getComposedRanges does not return a list of valid ranges. We can fix this by adding a new “composed valid” definition:

```txt
A StaticRange is composed valid if all of the followings are true:
- Its start and end have the same shadow-inclusive root.
- Its start offset is between 0 and its start node's length, inclusive.
- Its end offset is between 0 and its end node's length, inclusive.
- Its start is composed before or composed equal to its end.
```

Further, to support the above new definition and so we can compare boundary points across shadow boundaries, we also need to add new cases in position.

Rewrite definition in [5.2 Boundary points](https://dom.spec.whatwg.org/#boundary-points):

```txt
The composed position of a boundary point (nodeA, offsetA) relative to a boundary point (nodeB, offsetB) is before, equal, or after, as returned by these steps:
1. Assert: nodeA and nodeB have the same shadow-inclusive root.
2. If nodeA is nodeB,
  1. If offsetA is offsetB, return equal.
  2. If offsetA is less than offsetB, return before.
  3. If offsetA is greater than offsetB, return after.
3. If nodeA comes after nodeB in tree order, return the composed position of (nodeB, offsetB) relative to boundary point (nodeA, offsetA).
4. If nodeA is a shadow-including ancestor of nodeB:
  1. Let child be nodeB.
  2. While child is not a child of nodeA, set child to its parent or shadow host.
  3. If child's index is less than offsetA, then return after.
5. Return before.
```

## To define in the Selection API spec

Now that we have defined **Composed Live Range**, we need to update the language in the Selection API specification for everywhere we are referencing the associated range.

**Change**

Each selection can be associated with a single [range](AbstractRange).

To

Each selection is associated with a single **composed** **live** **range**.

**Change anchor/focus definition:**

Each [selection](https://w3c.github.io/selection-api/#dfn-selection)s also have an anchor and a focus. If the [selection](https://w3c.github.io/selection-api/#dfn-selection)'s [range](https://dom.spec.whatwg.org/#concept-range) is null, its [anchor](https://w3c.github.io/selection-api/#dfn-anchor) and [focus](https://w3c.github.io/selection-api/#dfn-focus) are both null. If the [selection](https://w3c.github.io/selection-api/#dfn-selection)'s [range](https://dom.spec.whatwg.org/#concept-range) is not null and its [direction](https://w3c.github.io/selection-api/#dfn-direction) is [forwards](https://w3c.github.io/selection-api/#dfn-forwards), its [anchor](https://w3c.github.io/selection-api/#dfn-anchor) is the [range](https://dom.spec.whatwg.org/#concept-range)'s [start](https://dom.spec.whatwg.org/#concept-range-start), and its [focus](https://w3c.github.io/selection-api/#dfn-focus) is the [end](https://dom.spec.whatwg.org/#concept-range-end). Otherwise, its [focus](https://w3c.github.io/selection-api/#dfn-focus) is the [start](https://dom.spec.whatwg.org/#concept-range-start) and its [anchor](https://w3c.github.io/selection-api/#dfn-anchor) is the [end](https://dom.spec.whatwg.org/#concept-range-end).

To

Each [selection](https://w3c.github.io/selection-api/#dfn-selection) also has an anchor and a focus. If the [selection](https://w3c.github.io/selection-api/#dfn-selection)'s **composed live** **range** is null, its [anchor](https://w3c.github.io/selection-api/#dfn-anchor) and [focus](https://w3c.github.io/selection-api/#dfn-focus) are both null. If the [selection](https://w3c.github.io/selection-api/#dfn-selection)'s **composed live** **range** is not null and its [direction](https://w3c.github.io/selection-api/#dfn-direction) is [forwards](https://w3c.github.io/selection-api/#dfn-forwards), its [anchor](https://w3c.github.io/selection-api/#dfn-anchor) is the **composed** [range](https://dom.spec.whatwg.org/#concept-range)'s [start](https://dom.spec.whatwg.org/#concept-range-start), and its [focus](https://w3c.github.io/selection-api/#dfn-focus) is the [end](https://dom.spec.whatwg.org/#concept-range-end). Otherwise, its [focus](https://w3c.github.io/selection-api/#dfn-focus) is the [start](https://dom.spec.whatwg.org/#concept-range-start) and its [anchor](https://w3c.github.io/selection-api/#dfn-anchor) is the [end](https://dom.spec.whatwg.org/#concept-range-end).

**Change getRangeAt(0):**

The method must throw an <code>[IndexSizeError](https://webidl.spec.whatwg.org/#indexsizeerror)</code> exception if is not <code>0</code>, or if [this](https://webidl.spec.whatwg.org/#this) is [empty](https://w3c.github.io/selection-api/#dfn-empty) or either [focus](https://w3c.github.io/selection-api/#dfn-focus) or [anchor](https://w3c.github.io/selection-api/#dfn-anchor) is not in the [document tree](https://dom.spec.whatwg.org/#concept-document-tree). Otherwise, it must return a reference to (not a copy of) <strong>[this](https://webidl.spec.whatwg.org/#this)'s composed live [range](https://dom.spec.whatwg.org/#concept-range)’s cached live range</strong>.

**Change setBaseAndExtent()**

5. If is <span style="text-decoration:underline;">composed [before](https://dom.spec.whatwg.org/#concept-range-bp-before)</span> ,

   1. [set the start](https://dom.spec.whatwg.org/#concept-range-bp-set) of to and [set the end](https://dom.spec.whatwg.org/#concept-range-bp-set) of newRange to .
   2. set this composed live range's [start](https://dom.spec.whatwg.org/#concept-range-start) to and this composed live range's [end](https://dom.spec.whatwg.org/#concept-range-end) to .

6. Otherwise,

   3. [set the start](https://dom.spec.whatwg.org/#concept-range-bp-set) of to focus and [set the end](https://dom.spec.whatwg.org/#concept-range-bp-set) of newRange to anchor.
   4. set this composed live range's [start](https://dom.spec.whatwg.org/#concept-range-start) to focus and this composed live range's [end](https://dom.spec.whatwg.org/#concept-range-end) to anchor.

7. Set this’s composed live range’s cached live range to newRange.

**Update collapse()**

7. Set\*\* **the **[start](https://dom.spec.whatwg.org/#concept-range-start) and the [end](https://dom.spec.whatwg.org/#concept-range-end) of this’s composed live range to (\***\*, \*\***).\*\*

8. Set this’s composed live range’s live range to newRange.

**Update selectAllChildren()**

6. Set the <span style="text-decoration:underline;">start</span> of this’s composed live range to (node, 0).

7. Set the <span style="text-decoration:underline;">end</span> of this’s composed live range to (node, childCount).

8. Set this’s composed live range’s live range to newRange.

**Update extend()**

6. [...] Set the <span style="text-decoration:underline;">start</span> of this’s composed live range to oldAnchor and set the <span style="text-decoration:underline;">end</span> of this’s composed live range to newFocus.

7. [...] Set the <span style="text-decoration:underline;">start</span> of this’s composed live range to newFocus and set the <span style="text-decoration:underline;">end</span> of this’s composed live range to oldAnchor.

8. Set this’s composed live range’s live range to newRange.

**Update getComposedRanges()**

2. Otherwise, let be [start node](https://dom.spec.whatwg.org/#concept-range-start-node) and let startOffset be <span style="text-decoration:underline;">start offset</span> of the **composed live rang**e associated with [this](https://webidl.spec.whatwg.org/#this).

3. Let be [end node](https://dom.spec.whatwg.org/#concept-range-end-node) and let be [end offset](https://dom.spec.whatwg.org/#concept-range-end-offset) of the **composed live range** associated with [this](https://webidl.spec.whatwg.org/#this).

4. Return an array consisting of new StaticRange whose start node is startNode, start offset is startOffset, end node is endNode, and end offset is endOffset.

## Issues

- https://github.com/w3c/selection-api/issues/2

- https://github.com/w3c/selection-api/issues/168

- https://github.com/w3c/selection-api/issues/169

- https://github.com/whatwg/dom/issues/772

Related issues:

- https://github.com/w3c/selection-api/issues/79

- https://github.com/w3c/selection-api/issues/175

- https://github.com/whatwg/dom/issues/725
