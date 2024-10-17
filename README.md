# Selection across Shadow DOM

Explainer for supporting Selection across Shadow DOM

Author: Di Zhang
Last updated: October 17, 2024

## Overview

This is an updated version of the [Explainer](https://github.com/mfreed7/shadow-dom-selection) by mfreed7.

## Current resolved API

As of October 2024, all browser implementers agree on adding the new `Selection.getComposedRanges()` and `Selection.direction` to expose information about the existing selection on a page. The output of `getComposedRanges` is a list of one `StaticRange`, where the endpoints might be crossing two different trees within the same owner document.

## Open questions

### Spec changes for getComposedRanges

We need to update Selection API specification to have Composed definitions. See [selection-api-spec-changes](./selection-api-spec-changes.md) for the full proposal.

### Flat Tree vs Composed Tree

Previously, with the assumption that all selections are within one tree, a DOM traversal is sufficient However, with the idea of a composed tree comes the necessity of re-considering how we traverse and compare the selection points.

Should we keep using a composed tree or is this API an opportunity to fix the existing Selection API to compare selection endpoints in a visual manner (using a flat tree)?

See main issue here: https://github.com/w3c/selection-api/issues/336

See [flat-tree-setBaseAndExtent](./flat-tree-setBaseAndExtent.md) for a proposal on how to extend the API to use flat tree traversal to set selection endpoints. Similar API should be designed for the other API functions.

### Effect of DOM mutations on the results getComposedRanges()

See main issue: https://github.com/w3c/selection-api/issues/168
