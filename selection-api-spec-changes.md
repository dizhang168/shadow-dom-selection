# Defining Composed for Selection API in Composed Tree

Author: dizhang168

Last updated: Dec 12, 2024

In this explainer, we list a set of definitions and algorithmic steps so we can spec getComposedRanges() for Selection API.

## Motivation

The current specifications for implementing Selection API's getComposedRanges() says to use the live range associated with the Selection. However, a live range is defined as an object that implements Range, with its start and end nodes contained within a tree scope. If so, the range cannot have endpoints that crosses the shadow boundaries. When getComposedRanges() access this range's endpoints, we are also getting endpoints that crosses the shadow boundaries, hence not actually composed.

## Proposal

First, we need to change the definition of a **Live Range** to:

- Is a [range](https://dom.spec.whatwg.org/#concept-range) (object implements AbstractRange)
- Is affected by mutations to the [node tree](https://dom.spec.whatwg.org/#concept-node-tree).

Second, with this new definition, we can define the new concept of **Composed Live Range**:

- Is a **live range** that has an associated single [Range](https://dom.spec.whatwg.org/#range) object _cached range_ (object implements Range).

Then, we update Selection API to be associated with that composed range.

## Spec Changes

### To define in the DOM spec

A **live range** is a [range](https://dom.spec.whatwg.org/#concept-range) that is affected by mutations to the [node tree](https://dom.spec.whatwg.org/#concept-node-tree).

Objects implementing the [Range](https://dom.spec.whatwg.org/#range) interface are <span style="text-decoration:underline;">live ranges</span>.

A **composed live range** is a <span style="text-decoration:underline;">live range</span> that has an associated single [Range](https://dom.spec.whatwg.org/#range) object _cached range_.

Note: The cached range is used to maintain backward compatibility for getRangeAt().

**Mutation: To [remove a node](https://dom.spec.whatwg.org/#concept-node-remove)**

8. For each **composed live range** whose **start** node is a **shadow-including inclusive descendant** of node, set its **start** to (parent, index).

9. For each **composed live range** whose **end** node is a **shadow-including inclusive descendant** of node, set its **end** to (parent, index).

**Change “To set the boundary point (node, offset) of a [Range](https://dom.spec.whatwg.org/#range) range, run these steps:”**

[https://dom.spec.whatwg.org/#concept-range-bp-set](https://dom.spec.whatwg.org/#concept-range-bp-set)

Add new step 5:

If range is the cached range of a _composed live range_,

1. Set _composed live range_'s **end** to range's end.
2. Set _composed live range_'s **start** to range's start.

**StaticRange**

Add definition after [StaticRange valid](https://dom.spec.whatwg.org/#staticrange).

A StaticRange is **composed valid** if all of the followings are true:

- Its start and end have the same **shadow-inclusive root**.
- Its start offset is between 0 and its start node's length, inclusive.
- Its end offset is between 0 and its end node's length, inclusive.
- Its start is **composed before** or **composed equa**l to its end.

**composed position**

Add definition in [5.2 Boundary points](https://dom.spec.whatwg.org/#boundary-points):

The **composed position** of a [boundary point](https://dom.spec.whatwg.org/#concept-range-bp) (nodeA, offsetA) relative to a [boundary point](https://dom.spec.whatwg.org/#concept-range-bp) (nodeB, offsetB) is **composed before, composed equal, or composed after**, as returned by these steps:

1. Assert: nodeA and nodeB have the same shadow-inclusive root.
2. If nodeA is nodeB, then return _composed equal_ if offsetA is offsetB, _composed before_ if offsetA is less than offsetB, and _composed after_ if offsetA is greater than offsetB.
3. If nodeA comes after nodeB in tree order, return the composed position of (nodeB, offsetB) relative to boundary point (nodeA, offsetA).
4. If nodeA is a shadow-including ancestor of nodeB:
   1. Let child be nodeB.
   2. While child is not a child of nodeA, set child to its parent or shadow host.
   3. If child's index is less than offsetA, then return _composed after_.
5. Return _composed before_.

### To define in the Selection API spec

**Change**

Each selection can be associated with a single [range](AbstractRange).

To

Each selection can be associated with a single **composed** **live** **range**.

**Change anchor/focus definition:**

Each [selection](https://w3c.github.io/selection-api/#dfn-selection)s also have an anchor and a focus. If the [selection](https://w3c.github.io/selection-api/#dfn-selection)'s [range](https://dom.spec.whatwg.org/#concept-range) is null, its [anchor](https://w3c.github.io/selection-api/#dfn-anchor) and [focus](https://w3c.github.io/selection-api/#dfn-focus) are both null. If the [selection](https://w3c.github.io/selection-api/#dfn-selection)'s [range](https://dom.spec.whatwg.org/#concept-range) is not null and its [direction](https://w3c.github.io/selection-api/#dfn-direction) is [forwards](https://w3c.github.io/selection-api/#dfn-forwards), its [anchor](https://w3c.github.io/selection-api/#dfn-anchor) is the [range](https://dom.spec.whatwg.org/#concept-range)'s [start](https://dom.spec.whatwg.org/#concept-range-start), and its [focus](https://w3c.github.io/selection-api/#dfn-focus) is the [end](https://dom.spec.whatwg.org/#concept-range-end). Otherwise, its [focus](https://w3c.github.io/selection-api/#dfn-focus) is the [start](https://dom.spec.whatwg.org/#concept-range-start) and its [anchor](https://w3c.github.io/selection-api/#dfn-anchor) is the [end](https://dom.spec.whatwg.org/#concept-range-end).

To

Each [selection](https://w3c.github.io/selection-api/#dfn-selection) also has an anchor and a focus. If the [selection](https://w3c.github.io/selection-api/#dfn-selection)'s **composed live** **range** is null, its [anchor](https://w3c.github.io/selection-api/#dfn-anchor) and [focus](https://w3c.github.io/selection-api/#dfn-focus) are both null. If the [selection](https://w3c.github.io/selection-api/#dfn-selection)'s **composed live** **range** is not null and its [direction](https://w3c.github.io/selection-api/#dfn-direction) is [forwards](https://w3c.github.io/selection-api/#dfn-forwards), its [anchor](https://w3c.github.io/selection-api/#dfn-anchor) is the **composed** [range](https://dom.spec.whatwg.org/#concept-range)'s [start](https://dom.spec.whatwg.org/#concept-range-start), and its [focus](https://w3c.github.io/selection-api/#dfn-focus) is the [end](https://dom.spec.whatwg.org/#concept-range-end). Otherwise, its [focus](https://w3c.github.io/selection-api/#dfn-focus) is the [start](https://dom.spec.whatwg.org/#concept-range-start) and its [anchor](https://w3c.github.io/selection-api/#dfn-anchor) is the [end](https://dom.spec.whatwg.org/#concept-range-end).

**Change getRangeAt(0):**

The method must throw an <code>[IndexSizeError](https://webidl.spec.whatwg.org/#indexsizeerror)</code> exception if is not <code>0</code>, or if [this](https://webidl.spec.whatwg.org/#this) is [empty](https://w3c.github.io/selection-api/#dfn-empty) or either [focus](https://w3c.github.io/selection-api/#dfn-focus) or [anchor](https://w3c.github.io/selection-api/#dfn-anchor) is not in the [document tree](https://dom.spec.whatwg.org/#concept-document-tree). Otherwise, it must return a reference to (not a copy of) <strong>[this](https://webidl.spec.whatwg.org/#this)'s composed live [range](https://dom.spec.whatwg.org/#concept-range)'s cached range</strong>.

**Change setBaseAndExtent()**

5. If is <span style="text-decoration:underline;">composed [before](https://dom.spec.whatwg.org/#concept-range-bp-before)</span> ,

   1. [set the start](https://dom.spec.whatwg.org/#concept-range-bp-set) of to and [set the end](https://dom.spec.whatwg.org/#concept-range-bp-set) of newRange to .
   2. set this composed live range's [start](https://dom.spec.whatwg.org/#concept-range-start) to and this composed live range's [end](https://dom.spec.whatwg.org/#concept-range-end) to .

6. Otherwise,

   3. [set the start](https://dom.spec.whatwg.org/#concept-range-bp-set) of to focus and [set the end](https://dom.spec.whatwg.org/#concept-range-bp-set) of newRange to anchor.
   4. set this composed live range's [start](https://dom.spec.whatwg.org/#concept-range-start) to focus and this composed live range's [end](https://dom.spec.whatwg.org/#concept-range-end) to anchor.

7. Set this's composed live range's cached range to newRange.

**Update collapse()**

7. Set the [start](https://dom.spec.whatwg.org/#concept-range-start) and the [end](https://dom.spec.whatwg.org/#concept-range-end) of this's composed live range to (node, offset).

8. Set this's composed live range's live range to newRange.

**Update selectAllChildren()**

6. Set the <span style="text-decoration:underline;">start</span> of this's composed live range to (node, 0).

7. Set the <span style="text-decoration:underline;">end</span> of this's composed live range to (node, childCount).

8. Set this's composed live range's live range to newRange.

**Update extend()**

6. [...] Set the <span style="text-decoration:underline;">start</span> of this's composed live range to oldAnchor and set the <span style="text-decoration:underline;">end</span> of this's composed live range to newFocus.

7. [...] Set the <span style="text-decoration:underline;">start</span> of this's composed live range to newFocus and set the <span style="text-decoration:underline;">end</span> of this's composed live range to oldAnchor.

8. Set this's composed live range's live range to newRange.

**Update getComposedRanges()**

2. Otherwise, let _startNode_ be [start node](https://dom.spec.whatwg.org/#concept-range-start-node) and let _startOffset_ be <span style="text-decoration:underline;">start offset</span> of the **composed live range** associated with [this](https://webidl.spec.whatwg.org/#this).

3. Let _endNode_ be [end node](https://dom.spec.whatwg.org/#concept-range-end-node) and let _endOffset_ be [end offset](https://dom.spec.whatwg.org/#concept-range-end-offset) of the **composed live range** associated with [this](https://webidl.spec.whatwg.org/#this).

## Issues

- https://github.com/w3c/selection-api/issues/2

- https://github.com/w3c/selection-api/issues/168

- https://github.com/whatwg/dom/issues/772

Related issues:

- https://github.com/w3c/selection-api/issues/79

- https://github.com/w3c/selection-api/issues/175

- https://github.com/whatwg/dom/issues/725
