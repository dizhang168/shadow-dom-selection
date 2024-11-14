# Defining Composed for Selection API in Composed Tree

In this explainer, we list a set of definitions and algorithmic steps so we can spec getComposedRanges() for Selection API.

## In HTML Specifications

## Definition of composed tree

Github issues:

- [Composed tree infrastructure](https://github.com/whatwg/dom/issues/725)

Use cases:

- event.composed/composedPath
- getRootNodeOptions
- Selection API
- CSS: Selectors
- title, bidi, etc?

Spec Proposal:

Add new section after [4.2.1 Document tree](https://dom.spec.whatwg.org/#document-trees):

### 4.2.X Composed tree

A **composed tree** is a [tree](https://dom.spec.whatwg.org/#trees) of trees where the root is a [shadow-including root](https://dom.spec.whatwg.org/#concept-shadow-including-root).

A **composed document tree** is a composed tree and its [shadow-including root](https://dom.spec.whatwg.org/#concept-shadow-including-root) is a document.

A node is **in a composed document tree** if its shadow-including root is a document.

An object A is **composed preceding** an object B if A and B are in the same composed tree and A comes before B in tree order.

An object A is **composed following** an object B if A and B are in the same composed tree and A comes after B in tree order.

### composed position

Add defintion in [5.2 Boundary points](https://dom.spec.whatwg.org/#boundary-points):

The **composed position** of a boundary point (nodeA, offsetA) relative to a boundary point (nodeB, offsetB) is **composed before**, **composed equal**, or **composed after**, as returned by these steps:

1. Assert: nodeA and nodeB have the same **shadow-inclusive root**.

2. If nodeA is nodeB, then return _composed equal_ if offsetA is offsetB, _composed before_ if offsetA is less than offsetB, and _composed after_ if offsetA is greater than offsetB.
3. If nodeA is _composed following_ nodeB:

   1. If the position of (nodeB, offsetB) relative to (nodeA, offsetA) is _composed before_, return _composed after_.

   2. Return _composed before_.

4. If nodeA is an **shadow-including ancestor** of nodeB:

   1. Let child be nodeB.

   2. While child is not a child of nodeA, set child to its **composed parent**.

   3. If child’s index is less than offsetA, then return _composed after_.

5. Return _composed before_.

Question: Not sure if "composed equal" is necessary to define.

### StaticRange

Add definition after [StaticRange valid](https://dom.spec.whatwg.org/#staticrange).

A StaticRange is **composed valid** if all of the following are true:

- Its start and end are in the same **composed tree**.

- Its start offset is between 0 and its start node’s length, inclusive.

- Its end offset is between 0 and its end node’s length, inclusive.

- Its start is **composed before** or **composed equal** to its end.

### Other HTML spec updates

With these new defintions, we can go back to some existing definitions to refer to them.

For example, the _event.composed_ can be updated from

> Returns true or false depending on how event was initialized. True if event invokes listeners past a ShadowRoot node that is the root of its target; otherwise false.

to

> Returns true or false depending on how event was initialized. True if event invokes listeners across the composed tree; otherwise false.

## In Selection API

Github issues:

- [Clarify association between a selection and its range](https://github.com/w3c/selection-api/issues/2)
- [Need spec changes to Range and StaticRange to support nodes in different tree?](https://github.com/w3c/selection-api/issues/169)

Currently, every document with a browsing context has a unique **selection** associated with it. This selection must be shared by all content of the document. Each selection can be associated with a single **range**. Currently, this is a **live range that is scoped within a tree node**. This is a problem because the selection needs to be for the document as a whole, but this live range is not able to cross shadow boundaries.

### New Composed anchor and focus

We add new concepts, which should determine the Selection inside a composed tree by tracking the selection information across the document in private values. The implementation is left to the User Agent. These values are not exposed to the web (until we want to introduce future APIs to do so).

Spec Proposal:

- Every Selection has a **composed anchor**, which defaults to the [anchor](https://w3c.github.io/selection-api/#dfn-anchor), but can be set and accessed by APIs in the Selection interface.

- Every Selection has a **composed focus**, which defaults to the [focus](https://w3c.github.io/selection-api/#dfn-focus), but can be set and accessed by APIs in the Selection interface.

We should also change the NOTE

> anchor and focus of selection need not to be in the document tree. It could be in a shadow tree of the same document.

to

> composed anchor and composed focus of selection need not to be in the document tree. It could be in a shadow tree of the same document.

Note: We don't need to add a composed direction because direction should already be returning the direction by considering endpoints across the composed tree.

When Selection API functions modify the Live Range, it should look at potentially modifying the composed focus/anchor.
If we want to access this information, it will have to be through the getComposedRanges() API, by passing a list of shadow roots we can traverse in.

#### setBaseAndExtent

We should modify the steps of [setBaseAndExtent()](https://w3c.github.io/selection-api/#dom-selection-setbaseandextent) by changing existing step:

7. If focus is before anchor, set this's direction to backwards. Otherwise, set it to forwards.

to

7. If focus is **composed before** anchor, set this's direction to backwards. Otherwise, set it to forwards.
8. Set this's **composed focus** to focus and this's **composed anchor** to anchor.

### getComposedRanges

We update the existing steps of [getComposedRanges](https://w3c.github.io/selection-api/#dom-selection-getcomposedranges):

2. Otherwise, let startNode be start node of the range associated with this, and let startOffset be start offset of the range.
3. ...
4. Let endNode be end node of the range associated with this, and let endOffset be end offset of the range.

to be

2. If direction is "forward", let startNode and startOffset be the node and offset of the **composed anchor**. Else, let startNode and startOffset be the node and offset of the **composed focus**.
3. ...
4. If direction is "forward", let startNode and startOffset be the node and offset of the **composed focus**. Else, let startNode and startOffset be the node and offset of the **composed anchor**.

### Other Selection API spec updates

We should update the definiton of **direction** and others that use before/after/equal to use composed before/composed after/composed equal.
