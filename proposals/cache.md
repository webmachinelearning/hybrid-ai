# Cache
An API to allow WebNN models to be saved to and retrieved from a cache on
persistent storage. 

## Problem Statement
Models can be large and in many cases we would like to reuse previously downloaded models.
While a loader library can store and retrieve models prior to building model graphs, 
this potentially results in duplicated storage.  Also storing and retrieving models
prior to building forces the recompilation to happen upon every use.  It would be useful
to have a local storage and retrieval mechanism that permits compilation to be bypassed as well.

There are many possible approaches to caching data for reuse between sessions that can be 
used for models, as discussed in (Reference 2).  This proposal also simplifies and abstracts
them, giving the implementation the freedom to store data using an opaque, implementation-defined
mechanism.

However, it should be noted that *cross-origin* sharing of models carries various privacy risks.
Currently, the best known practice to preserve privacy is to use
Client-Side Storage Partitioning (Reference 3), where every origin has its own storage.   

This initial proposal is suitable for the partitioned storage model and is presented that way.
Later we may consider extensions to support cross-origin sharing, if appropriate privacy mitigations can
be defined.

## Use Cases

### Use Case 1: Same-Origin Reuse
A developer may want to avoid redundant download (and possibly building) of a model that 
may have previously been downloaded and built in a previous session by the same origin.
Avoiding the download/build process improves the user experience and also reduces the load
on the network (and the potential cost to the user if data charges are applicable).

### Use Case 2: Adapters
Separately an API is being proposed to support adapters [5], based on PEFT representation such as LoRA.
As a prerequiste to implementing adapters, it must be possible to gain access to a previously downloaded
model.

## Out of Scope
The following use case is currently out of scope but may be considered in future work:

### Use Case X1: Cross-Origin Reuse
Many models may be used by more than one developer, in the same way that software libraries
may be shared. 

## Design Considerations
- It should not be possible to use the cache for storage of data other than models.  In particular
  we should avoid situations where the cache can be used for large cookies or trackers.
- Loading a model from the cache should be as efficient as possible.
- The API should be simple and easy to use.
- While the proposed API does not support Use Case X1 (Cross-Origin Reuse), it should be possible
  to extend the API in the future to do so in a backward-compatible way.
  
## Proposed API
The proposed API is based on two new methods, `store` on `MLGraph` objects
and `fetch` on `MLGraphBuilder` objects.

### Store
The following additional method on `MLGraph`
would allow storing a model in the cache, returning
a hash of the model that can be used as a key:
```js
const hash = graph.store()
```
This API intentionally does not specify a storage location, file system handle, or the like.
Models are stored in an opaque serialization and in an unspecified location that does
not count against the overall storage quota of the origin.
Depending on various factors, the system may also choose not to store particular models;
see "Implementation Notes".

NOTE: One design alternative omits this method, and instead automatically attempts to
cache ALL models.  See later discussion.

### Fetch
Later, the model can be retrieved from the cache using the hash.  This would
behave similarily to `build`:
```js
graph = await builder.fetch(hash);
```

Hashes are deterministic so it is also possible to treat the hash as the name of 
a specific model that may or may not already be in the cache.  In the following
we assume hashes are computed using SHA-256, which should be sufficient:
```js
graph = await builder.fetch("2ff2366c-3acb-4a58-93ac-53971b0e1b18");
```
Use of a data-dependent hash as a name avoids issues
with cached models being used as large cookies, as well
as ensuring that the model is validated and is the model expected (and not
a modified, potentially malicious model given the same name).
For potential extension to the cross-origin use case, use of hashes also prevents the
use of the cache for data exfiltration.

The `builder` in the above examples is an object with the WebNN `MLGraphBuilder` interface.
The context of this builder should be compatible with the context under which the
original graph was stored, in particular it should support the same target devices.
The returned graph object will appear as if it had been built by the provided `MLGraphBuilder` interface.

If for any reason a `fetch` cannot be completed, including device incompatibility (e.g. the 
previously stored model was compiled for a different device) the `fetch` function will return a 
`null` value.

NOTE: Consider throwning an exception instead of returning `null`.  Should be consistent with
the `build` method of `MLGraphBuilder`, which rejects the promise and throws various exceptions
for different failure modes.  The most likely failure mode, given by definition the model should
have previously be successfully compiled, is in Step 16 of Section 7.7.4 "build method"
of (Reference 4), which returns an `"OperationError" DOMException`.

It is also possible that a model previously stored in a the cache may not be available
for fetching later, either because it was flushed or was not stored for implementation-defined
reasons.

NOTE: If exceptions are used to signal errors, additional exception classes might be
needed to signal additional sources of error, such as the model not being in the cache.
However, this could also be used for "probing" attacks (fingerprinting a system by what models seem 
to be in the cache) so it might be best from a privacy
point of view to consolidate error types.

## Privacy Mitigations
- The WebNN API is already disallowed in third-party contexts, which mitigates
  several scenarios in which the API can be abused for tracking.
- The presence of an item in the cache in theory could be used for tracking repeat
  visits by a particular user agent to the same site, by generating a random model
  and then computing and storing the hash on the server side.   However, note that
  the API only checks for the existence of a given hash in the cache, it does not
  provide a list of all items in the cache.  An attacker would have to "guess" at
  which hash is in the cache, and possibly test them one by one.  The API can limit
  the number of attempts to retrieve models from the cache to avoid a system trying
  to exhaustively probe the cache.  This failure can be "soft" e.g. the cache can
  just stop trying to retrieve data after a certain number of attempts and let all
  fetches "fail".

## Implementation Notes
- This API can be implemented in three ways.  The proposed API is intentially designed to allow
  all three:
   1. A shim around the entire `MLGraph` API that allows the data to build models to be captured
      in a data structure so they can be stored and retrieved from a file.  In this implementation
      "fetching" a model from the cache requires it to be rebuilt by traversing the data structure
      and invoking the native WebNN API.  This approach is slow and inefficient but is portable and
      allows the model to be retargetted to other devices.
   3. Internal to the WebNN API, the internal representation of the model, in particular the weights and the
      operator graph, can be stored prior to compilation.  In this case fetching the model still requires
      recompilation but does not have the overhead of managing the data in Javascript.  It also allows
      a model to be retargeted to other devices.
   4. Internal to the WebNN API, the internal representation of the compiled model, including the weights
      and the compiled kernels, is stored to a file so that it can be quickly retrieved and run on a
      target device.  This is faster since it avoids compilation overhead, but requires the ability
      to serialize/deserialize a compiled representation which may not be available for all backends.
      The representation is also device-specific.
- The hash is based on the model, not for the device it is compiled for.  This leads to a corner
  case where the model is available, but has been compiled for a different device.  In this case the
  fetch API would fail and the developer might rebuild the model for the new device, and might store
  the new compiled model in the cache.  A simple implementation might only store one copy of every model
  (discarding and replacing the older compiled model in this case) but a "better" implementation might
  store multiple compiled versions and return the one that is the best fit to the provided context.
- An implementation may choose to not store some models at all, and may also delete stored models at any
  time.  The simplest implementation is to store no models so fetches will always fail.  This is
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
  In addition, we would also need to add another method, perhaps `hash`, to retrieve the hash value
  from the built graph.
- A separate "AI Model Management" class/interface could be used.  However, making the API methods
  on `MLGraph` and `MLGraphBuilder` is more natural for an object-oriented system.
- The use of a hash means that the *exact* model specified is fetched.  If there are multiple versions
  of the same model that are functionally equivalent, a system would have to try the variants one
  by one (maybe from newest to oldest) to find if one is in the cache.  As an extension to this API,
  we could consider allowing `fetch` to take a list of hashes, and allow it to retrieve the model
  associated with any of them.
- We *could* use model names, e.g. `llama3.1:70b` rather than hashes.  However, then the name would
  have to be provided at storage time (so automatic storage would not be possible...) and a developer
  could abuse the API as a large name-value store.   This also leads to additional risks e.g. for
  data exfiltration if we ever extend the API to cross-origin caching.  To avoid the inconvenience
  of using hashes, a library could be provided to access a database of hashes for common models.

## References
1. Thomas Steiner, Google,
   [Cache AI Models in the Browser](https://developer.chrome.com/docs/ai/cache-models). 
2. Michael McCool, Geoff Gustafson, Sudeep Divakaran, and Muthaiah Venkatachalam.
   [AI Model Management](https://github.com/webmachinelearning/hybrid-ai/blob/main/presentations/WebML%20Discussion%20-%20Hybrid%20AI%20for%20the%20Web%20-%20AI%20Model%20Management%20TPAC%202024%20Breakout.pdf).
   September 25, 2024, W3C TPAC 2024 Breakout.
3. W3C Privacy CG. [Client-Side Storage Partitioning](https://github.com/privacycg/storage-partitioning).
4. Ningxin Hu and Dwayne Robinson (Editors), W3C Web Machine Learning WG. [Web Neural Network API](https://www.w3.org/TR/webnn/).
5. Michael McCool, [Adapters proposal](adapters.md).
