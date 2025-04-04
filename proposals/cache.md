# Cache
An API to allow WebNN models to be saved to and retrieved from a cache on
persistent storage.   

NOTE: This proposal supports cross-origin caching, however the group consensus
is that scope is too broad and we should first focus on the same-origin case,
in alignment with the Storage Partitioning principle used by other APIs.
For an updated proposal that narrows the scope to the same-origin case, see
[localcache.md](localcache.md).

## Problem Statement
Models can be large and in many cases we would like to reuse previously downloaded models.
While a loader library can store and retrieve models prior to building model graphs, 
this potentially results in duplicated storage.  Also, storing and retrieving models
prior to building forces the recompilation to happen every session.  It would be useful
to have a local storage and retrieval mechanism that permits compilation to be bypassed as well.

There are many possible approaches to caching data for reuse between sessions that can be 
used for models, as discussed in (Reference 2).  This proposal also simplifies and abstracts
them, giving the implementation the freedom to store data using an opaque, implementation-defined
mechanism.

However, it should be noted that *cross-origin* sharing of models carries various privacy risks.
Currently, the best known practice to preserve privacy is to use
Client-Side Storage Partitioning (Reference 3), where every origin has its own storage.   

This initial proposal is suitable for the partitioned storage model and is presented that way.
However, later in the document some extensions to support cross-origin sharing are proposed that attempt
to mitigate various privacy risks.  

## Use Cases

### Use Case 1: Same-Origin Reuse
A developer may want to avoid redundant download *and* compilation of a model that 
may have previously been downloaded and built in a previous session by the same origin.

Avoiding the download process improves the user experience and also reduces the load
on the network (and the potential cost to the user if data charges are applicable).
Avoiding the build process improves the user experience by minimizing startup time.

### Use Case 2: Cross-Origin Reuse (TENTATIVE)
Many models may be used by more than one developer, in the same way that software libraries
may be shared.  Like software libraries, we expect models to be reused by different origins.
However, models are much larger so the cost of duplicated storage is larger.

This is not as high priority as the above use case and the proposal is designed to
allow its implementation to be deferred, or not implemented at all in some platforms.

### Use Case 3: Adapters (EXPLORATORY)
Separately an API is being proposed to support adapters [5], based on PEFT representation such as LoRA.
As a prerequiste to implementing adapters, it must be possible to gain access to a previously downloaded
model.

## Design Considerations
- Ideally, it should not be possible to use the cache for storage of data other than models.
  In particular we should avoid situations where the cache can be used for large cookies or trackers,
  as this may require user consent, etc.
- Loading a model from the cache should be as efficient as possible.
- The API should be simple and easy to use.
- An application needs to be able to check the cache *before* downloading a model, so an
  implicit, automatic approach is not possible.
  
## Proposed API
The proposed API is based on a set of methods on `MLContext` objects,
described in the IDL below:
```js
partial interface MLContext {
  Promise<MLGraph> loadGraph(DOMString key);
  Promise<DOMString> saveGraph(MLGraph graph, optional DOMString key);
  Promise<DOMString> hashGraph(MLGraph graph);
  undefined deleteGraph(DOMString key);
  undefined deleteSavedGraphs();  // of this origin
};
```
In the following each method is discussed.

### saveGraph
This method intentionally does not specify a storage location, file system handle, or the like.
Models are stored in an opaque serialization and in an unspecified location that does
not count against the overall storage quota of the origin.
Depending on various factors, the system may also choose not to store particular models;
see "Implementation Notes".

The user-provided key is optional.  In all cases, the system would compute and return a
deterministic data-dependent hash key that could also be used for retreival (with some caveats discussed later).
If no user-provided key is given, then the hash *must* be used for retreival.

If the same graph is "saved" more than once with a different user-provided key than the
latest user-provided key is used and older keys are lost, e.g. the function will act like a "rename".
The hash key will not change in this case, and the system may also skip recompilation.

If a graph is saved with the same user-provided name as a graph already in the cache the new
graph will be new one accessed by the user-provided name.  The old graph may still be accessible 
using its hash (e.g. it will not necessarily be automatically deleted).

An implementation may choose not to store any particular model (for example, a model may be too
large to fit in the available storage).  In this case the method still succeeds but the empty string
may be returned as the "hash" (to avoid the system having to compute the hash for models that
are not stored).  

An empty string cannot be used as a user-provided key.  If such as string is provided
as the user-defined key it will be ignored and the behavior will be the same as if the 
optional argument were omitted.

### hashGraph
Access the hash key for an existing graph.  This is a convenience function in case the
graph is built inside a library and it is not possible to access and modify the call to
`saveGraph`.   For the same `MLGraph` object this always returns the same string.

### loadGraph
Allows a model to be retrieved from the cache using either the hash key or the user-defined
key.

Hashes are deterministic so it is also possible to treat the hash as the name of
a specific model that may or may not already be in the cache.  In the following
we assume hashes are computed using SHA-256, which should be sufficient.  The method to
compute the hash will be defined by the standard so it is consistent on all platforms -
this is necessary so an application can look up the hash on a model server independent
of the platform.  Hash names will be checked first, and then user-provided names, so it 
is possible (if a developer does it intentionally) for a hash to shadow a user-provided name.

If for any reason a `loadGraph` cannot be completed an exception will be thrown to reject the promise.
These should be consistent with those used for the `build` method for different failure modes.
The most likely failure mode, given by definition the model should
have previously be successfully compiled, is in Step 16 of Section 7.7.4 "build method"
of (Reference 4), which returns an `"OperationError" DOMException`.

It is also possible that a model previously stored in the cache may not be available
for fetching later, either because it was flushed or was not stored for implementation-defined
reasons.  

To allow for a "null" no-cache implementation, an implementation may choose
to not implement a cache at all and always throw exceptions for this method, even for
`MLGraph` objects that were just saved and are still live in the current session.

The `loadGraph` method will always throw an exception for the empty string "hash" that may
be returned by a refused `storeGraph` operation.

## deleteGraph
Given either the user-provided key or the hash key, a model is removed from the cache.
This method always succeeds, even if the model was not originally in the cache (this is to avoid
someone using the delete operation as a cache probe) or is no longer in the cache.
Idempotent.

Subsequent loads for these keys will fail unless the graph is saved again.

Note: an implementation may choose to delete saved models automatically for any reason.

## deleteSavedGraphs
Delete all graphs of this origin.  Always succeeds, even if the cache is empty.
Idempotent.

All subsequent loads will fail until new models are saved.

## Design Considerations
Use of a data-dependent hash as a name avoids issues
with cached models being used as large cookies, as well
as ensuring that the model is validated and is the model expected (and not
a modified, potentially malicious model given the same name).  It also strengthens
protections against fingerprinting and tracking.

If arbitrary user-defined keys are used, then the cache essentially becomes a key-value store,
and may be subject to the same privacy controls as cookies and localStorage, e.g. requiring
user consent to use.  User-defined keys are provided as a convenience feature for the developer but could
potentially cause other inconveniences for the user.



### Privacy Mitigations
- The WebNN API is already disallowed in third-party contexts, which mitigates
  several scenarios in which the API can be abused for tracking.
- The presence of an item in the cache in theory could be used for tracking repeat
  visits by a particular user agent to the same site, by generating a random model
  and then computing and storing the hash on the server side.   However, note that
  the API only checks for the existence of a given hash in the cache, it (intentionally) does not
  provide a list of all items in the cache.  An attacker would have to "guess" at
  which hash is in the cache, and possibly test them one by one.  The API can limit
  the number of attempts to retrieve models from the cache to resist a system trying
  to exhaustively probe the cache.  This failure can be "soft" e.g. the cache can
  just stop trying to retrieve data after a certain number of attempts and let all
  `loadGraph` calls "fail".  For the same reason `deleteGraph` calls always succeed
  to avoid having them being used as probes.

## Extensions for XSS Case
Although not a primary goal, it is worth thinking about how the API might be extended to the 
cross-origin case while mitigating privacy risks.

The primary change would be *requiring* data-dependent hashes as keys to access models
possibly loaded from another origin.
Within a session, user-provided keys could be used, but such names would only be able to 
load models saved from the same origin.  This is to
prevent the unauthorized exfiltration of bulk data between origins.

In addition, a privacy-sensitive browser might require the use of hashes to access 
models even for a single origin, in order to more strongly mitigate other risks, such
as fingerprinting.  A browser may also not strictly disallow the use of "friendly" names to
retrieve items from the cache, but may instead choose to require user consent.

### Privacy Mitigations
- A shared key-value store with arbitrary data-independent keys can be used for
  exfiltrating large amounts of data from one origin to another.  The restriction
  to only allow use of hashes as keys to non-same-origin models blocks data exfiltration
  since to compute the hash key the data has to be known.  Therefore the key-value store
  cannot be used to transmit *unknown* data.
- For similar reasons, a tracking ID for a particular user is data which needs to be
  independent of the key.  Forcing different data items to use different hash keys
  and ALSO limiting cache probes and preventing enumeration blocks easy recovery
  of a tracking ID left behind in another origin.

## Implementation Notes
- This API can be implemented in (at least) four ways.  The proposed API is intentially designed to allow
  the following:
   1. A shim around the entire `MLGraph` API that allows the data to build models to be captured
      in a data structure so they can be stored and retrieved from a file.  In this implementation
      "fetching" a model from the cache requires it to be rebuilt by traversing the data structure
      and invoking the native WebNN API.  This approach is slow and inefficient but is portable and
      allows the model to be retargetted to other devices.
   2. Internal to the WebNN API, the internal representation of the model, in particular the weights and the
      operator graph, can be stored prior to compilation.  In this case fetching the model still requires
      recompilation but does not have the overhead of managing the data in Javascript.  It also allows
      a model to be retargeted to other devices.
   3. Internal to the WebNN API, the internal representation of the compiled model, including the weights
      and the compiled kernels, is stored to a file so that it can be quickly retrieved and run on a
      target device.  This is faster since it avoids compilation overhead, but requires the ability
      to serialize/deserialize a compiled representation which may not be available for all backends.
      The representation is also device-specific.
   4. No cache. Loads always throw exceptions, and returned "hashes" from Stores are always the empty string.
- The hash is based on the model, and should not depend on the device it is compiled for.  This leads to the 
  case where the model is available, but has been compiled for a different device, which might duplicate some
  storage.  We should allow implementations to report that such models ARE in the cache (although it
  means models would have to be stored in a portable format, compilation would still be needed, and so on).
- An implementation may choose to not store some models at all, and may also delete stored models at any
  time.  The simplest implementation (4 above) is to store no models at all.  This is
  acceptable and would be compatible with the proposal, although obviously would not address the use
  cases.  For systems with finite storage capacity, an implementation can automatically delete older
  models, in which case they would have to be reloaded by the developer.  An implementation may also
  choose not to store certain models for implementation-defined reasons, e.g. models that are too large.

## Design Alternatives
- We could omit the `store` operation and always attempt to automatically store models in the cache as
  part of the `build` process.
  There may, however, be reasons while a developer does not want to allow a model to get saved to disk,
  e.g. proprietary models.  However, as this is a rare case an alternative would be to have a flag
  to mark such models as `uncacheable`, but by default allow caching.
- The use of a hash means that the *exact* model specified is fetched.  If there are multiple versions
  of the same model that are functionally equivalent, a system would have to try the variants one
  by one (maybe from newest to oldest) to find if one is in the cache.  As an extension to this API,
  we could consider allowing `loadGraph` to take a list of hashes, and allow it to retrieve the model
  associated with any of them. To avoid the inconvenience
  of using hashes, a library could be provided to access a database of hashes for common models.
- There is some overlap of design considerations and motivations with the Cross-Origin Storage (COS)
  API proposal, see Reference 6.  However this proposal is more focused on models and is a cache,
  not a general file storage mechanism.

## References
1. Thomas Steiner, Google,
   [Cache AI Models in the Browser](https://developer.chrome.com/docs/ai/cache-models). 
2. Michael McCool, Geoff Gustafson, Sudeep Divakaran, and Muthaiah Venkatachalam.
   [AI Model Management](https://github.com/webmachinelearning/hybrid-ai/blob/main/presentations/WebML%20Discussion%20-%20Hybrid%20AI%20for%20the%20Web%20-%20AI%20Model%20Management%20TPAC%202024%20Breakout.pdf).
   September 25, 2024, W3C TPAC 2024 Breakout.
3. W3C Privacy CG. [Client-Side Storage Partitioning](https://github.com/privacycg/storage-partitioning).
4. Ningxin Hu and Dwayne Robinson (Editors), W3C Web Machine Learning WG. [Web Neural Network API](https://www.w3.org/TR/webnn/).
5. Michael McCool, [Adapters proposal](adapters.md).
6. Thomas Steiner, Christian Liebel, and François Beaufort, [Cross-Origin Storage (COS) API (proposal)](https://github.com/explainers-by-googlers/cross-origin-storage)
