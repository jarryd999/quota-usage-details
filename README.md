# Explainer: Storage Quota Usage Details

This document proposes an addition to the Storage API (specifically the Storage Manager interface) that would allow script to asynchronously request an estimation of quota usage broken down by storage backend.

## Problem
There have been frequent requests from users of `navigator.storage.estimate()` to provide a per storage type breakdown estimation.  Currently, a call to this function yields only an estimate of the quota usage for all storage systems combined, making it difficult to reason about what is using up quota.  This change primarily aims to help developers in debugging applications.  With a detailed per system usage breakdown, apps are provided more context and clues to detect and diagnose storage overuse problems.  For example, one can consider an email client that uses IndexedDB to store text and Cache Storage to store attachments.  With the proposed change, said app would be able to debug problematic storage scenarios: high Cache Storage usage but low IndexedDB usage would suggest the app forgot to delete attachments when evicting messages from the local cache.  If there was high usage for both storage backends, this would mean the app is caching too many messages, suggesting the eviction policy is not behaving correctly.

## Concepts
* `usage` reflects how many bytes a given origin is effectively using for same-origin data, which in turn can be impacted by internal compression techniques, fixed-size allocation blocks that might include unused space, and the presence of "tombstone" records that might be created temporarily following a deletion. To prevent the leakage of exact size information, cross-origin, opaque resources saved locally may contribute additional padding bytes to the overall usage value.  As data is stored, the "usage"  goes up and as data is deleted the usage goes down. 
* `quota` reflects the amount of space currently granted to an origin. The value depends on some constant factors like the overall storage size, but also a number of potentially volatile factors, including the amount of storage space that's currently unused. Operations in Indexed DB, Cache API, WebSQL, FileSystem, etc. will fail with a `QuotaExceededError` if usage would increase past quota.
* Within origin-scoped data, a subset is quota-managed data accounted for by the `Quota Manager`
 * e.g. in Chrome/Firefox indexed DB is quota-managed data, whereas local storage is not quota-managed data  (note: neither case is specified behavior).

## Proposed Solution
In addition to providing an overall quota and usage, a call to `storage.estimate()` will also provide an estimate of the usage by each quota-managed storage system. The proposed change adds another member, `usageDetails` to the dictionary returned by `navigator.storage.estimate()`.  This new dictonary will contain key-value pairs showing the usage of each storage system, where the keys are the names of the storage systems and the value is an estimate, in bytes, of how much disk space said system is using. 

<p align="center">
<img src="https://github.com/jarryd999/quota-usage-details/blob/master/UsageDetails.png?raw=true" />
<br/>
Figure 1: Example use the API that highlights how usageDetails mirrors devtools' usage report
</p>

## Caveats Worth Mentioning
The dictionary will omit any pair in which the usage is 0. This lets developers worry only about the storage systems they use and avoids breaking the web when storage systems are deprecated.

The cost to this decision is that developers might not know that there are storage systems missing from the usage details dictionary. It's not obvious that the total usage might differ from the sum of the usageDetails members. This is something that can be addressed via documentation. An alternative is to have an other property on usageDetails that's literally total - Sum(usageDetails).

localStorage is not counted in the quota. I think we should address this via documentation, and not change the API shape. An alternative is to add a usageDetails.localStorage that's always set to zero.

## Alternatives Considered

1. Expose subsystems with 0 usage.
   * Benefit: the API shape is intuitive.
   * Cost: we can't measure if we can remove storage systems without breaking the Web.

2. Expose a synchronous storage.getUsageDetails() function that takes storage system name as a paramter and returns usage.  This would return 0 for unknown categories.
   * Benefit: storage.getUsageDetails('fileSystem') will work correctly whether the browser supports a file system API or not.
   * Cost: the param values are not discoverable.
