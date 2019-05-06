# Explainer: Storage Quota Usage Details

This document proposes an addition to the Storage API (specifically the Storage Manager interface) that would allow script to asynchronously request an estimation of quota usage broken down by storage backend.

## Problem
There have been frequent requests from users of [`navigator.storage.estimate()`](https://developer.mozilla.org/en-US/docs/Web/API/StorageManager/estimate) to provide a per-storage system breakdown estimate.  

Currently, a call to this function yields only an estimate of the quota usage for all storage systems combined, making it difficult for developers to reason about what is using up quota.  

For example, consider an email client that uses IndexedDB to store text and Cache Storage to store attachments.  If this client had a pattern of high Cache Storage usage, but low IndexedDB usage, a developer might be able to determine that the app forgot to delete attachments when evicting messages from the local cache.  If there were high usage for both storage backends, this might mean the app is caching too many messages, suggesting the eviction policy is not behaving correctly.

By providing a more detailed breakdown of how storage is being used, the change proposed below will allow developers to make these assessments.

## Concepts

Currently, `StorageManager.estimate()` asynchronously provides a `StorageEstimate` object with two fields, `usage` and `quota`.

* `usage` reflects how many bytes a given origin is effectively using for same-origin data, which in turn can be impacted by internal compression techniques, fixed-size allocation blocks that might include unused space, and the presence of "tombstone" records that might be created temporarily following a deletion. 
   * To prevent the leakage of exact size information, cross-origin, opaque resources saved locally may contribute additional padding bytes to the overall usage value.  As data is stored, the usage goes up and as data is deleted the usage goes down. 
* `quota` reflects the amount of space currently granted to an origin. 
   * The value depends on some constant factors like the overall storage size, but also a number of potentially volatile factors, including the amount of storage space that's currently unused. Operations in IndexedDB, Cache API, WebSQL, FileSystem, etc. will fail with a `QuotaExceededError` if usage would increase past quota.

Within origin-scoped data, a subset is quota-managed data accounted for by the `Quota Manager`
  * e.g. in Chrome/Firefox IndexedDB is quota-managed data, whereas localStorage is not quota-managed data  (note: neither case is specified behavior).

## Proposed Solution

The proposed change adds another member, `usageDetails`, to the `StorageEstimate` dictionary returned by `navigator.storage.estimate()`.  This new dictonary will contain key-value pairs showing the usage of each storage system, where the keys are the names of the storage systems and the value is an estimate, in bytes, of how much disk space said system is using. 

<p align="center">
<img src="https://github.com/jarryd999/quota-usage-details/blob/master/UsageDetails.png?raw=true" />
<br/>
Figure 1: Example use of the API from the Chrome devtools console, demonstrating that usageDetails mirrors devtools' usage report.
</p>

## Caveats Worth Mentioning
1. The dictionary will omit any pair in which the usage is 0. This lets developers worry only about the storage systems they use and avoids backwards incompatibility when storage systems are deprecated. 
   * The cost to this decision is that developers might not know that there are storage systems missing from the usage details dictionary. Developers would also run into issues if they wrote things like `usageDetails.caches + usageDetails.indexedDB` which, if the usage for either of those was 0, would return `NaN`.  Instead, developers would have to be defensive and write something like `(usageDetails.caches || 0) + (usageDetails.indexedDB || 0)`. 

2. It's not obvious that the total usage might differ from the sum of the `usageDetails` members. 
   * Today, this could be due to storage systems whose usage isn't reported. In the future, there may be overhead, or we may not be able to attribute usage with 100% accuracy. This is something that can be addressed via documentation. An alternative is to have an `other` property on `usageDetails` that's literally `total - Sum(usageDetails)`.

3. `DOMStorage` (`localStorage`) is not counted in the quota. This can be addressed via documentation rather than a change to the API shape.

## Alternatives Considered

1. Expose storage systems with 0 usage.
   * Benefit: the API shape is intuitive.
   * Cost: we can't measure if we can remove storage systems without breaking the Web.

2. Expose a synchronous storage.getUsageDetails() function that takes the storage system name as a parameter and returns usage.  This would return 0 for unknown categories.
   * Benefit: storage.getUsageDetails('fileSystem') will work correctly whether the browser supports a file system API or not.
   * Cost: the param values are not discoverable.
