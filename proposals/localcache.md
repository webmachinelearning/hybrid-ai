# Local (Same-Origin) Model Cache
An API to allow WebNN models to be saved to and retrieved from a cache on
persistent storage.   

NOTE: This proposal is limited to same-origin caching, and follows the 
Storage Partitioning principle used by other local storage APIs.  
For the cross-origin case, see the proposal in [cache.md](cache.md),
however the group consensus is that this is out of scope for now.

Related Issue: https://github.com/webmachinelearning/webnn/issues/807

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
This proposal is limited to this case and designed to be similar to other APIs that provide local storage. 

## Use Cases

### Use Case 1: Same-Origin Reuse
A developer may want to avoid redundant download *and* compilation of a model that 
may have previously been downloaded and built in a previous session by the same origin.

Avoiding the download process improves the user experience and also reduces the load
on the network (and the potential cost to the user if data charges are applicable).
Avoiding the build process improves the user experience by minimizing startup time.

## Out of Scope

### Cross-Origin Reuse 
Sharing data cross-origin raises many privacy concerns that are difficult to mitigate,
and so cross-origin sharing of models using is considered out of scope of this proposal.

## Design Considerations
- Loading a model from the cache should be as efficient as possible.
- The API should be simple and easy to use.
- An application needs to be able to check the cache *before* downloading a model.
- The representation of the model should be opaque.
- The API should not provide general file-system access.
- The cache should work with models compiled internally for different devices.
  It is not necessary to be able to reuse a model compiled for one device for a different device.
  
## Proposed API
The proposed API is based on a set of methods on `MLContext` objects,
described in the IDL below:
```js
partial interface MLContext {
  Promise<sequence<DOMString>> listGraphs();
  Promise<MLGraph> loadGraph(DOMString key);
  Promise<DOMString> saveGraph(MLGraph graph, DOMString key);
  undefined deleteGraph(DOMString key);
  undefined deleteSavedGraphs();  // of this origin
};
```
In the following each method is discussed.
### listGraphs
Returns a sequence of all the keys for all models currently in the cache that can
be loaded.  The order of the keys is arbitrary.  Each key appears only once.

### saveGraph
This method intentionally does not specify a storage location, file system handle, or the like.
Models are stored in an opaque serialization and in an unspecified location.
Depending on various factors, the system may also choose not to store particular models;
see "Implementation Notes".

- DISCUSS how quota is handled.  Is there a different quota for models vs. other items?
- DISCUSS whether this API is a cache or a file-system.  **Currently the proposal is written
  as for a cache, so the system will delete models as necessary to manage space.**  We may want
  to consider an extension that makes models more persistent and requires explicit deletion
  (although there will still probably have to be cases where the system needs to delete models
  itself anyway).

NOTE: In the original proposal the key was optional.  It is now required.

The user-provided key is an abitrary, non-empty string that identifies the model.
Names are not shared across origins, e.g. the model namespaces as well as the storage is
partitioned.

If the save succeeds then the key of the model is returned asynchronously.

If a graph is saved with the same user-provided key as a graph already in the cache the old graph
will be automatically deleted.

- DISCUSS: What do we do about models that are currently "open", e.g. in use with a live
  handle?  We should look at other APIs to see how they handle this case but it might be a good
  idea to refuse to delete models "in use" and throw an exception.

An implementation may choose not to store any particular model (for example, a model may be too
large to fit in the available storage).  In this case the method still succeeds but the empty string
may be returned.

- DISCUSS: Might be better to throw an exception in this case.  Look at other storage APIs.

The system should prioritize new models.  In particular, if there is not enough space to store
the current model but there are older models not retrieved by the API, the system may
automatically delete one or more of these inactive models to save space.

An empty string cannot be used as a user-provided key. 

### loadGraph
Allows a model to be retrieved from the cache using the user-defined key.

If for any reason a `loadGraph` cannot be completed an exception will be thrown to reject the promise.
These should be consistent with those used for the `build` method for different failure modes.
The most likely failure mode, given by definition the model should
have previously be successfully compiled, is in Step 16 of Section 7.7.4 "build method"
of (Reference 4), which returns an `"OperationError" DOMException`.

It is also possible that a model previously stored in the cache may not be available
for fetching later, either because it was flushed (automatically deleted) 
or was not stored for implementation-defined reasons.
It may or may not be possible for the same model saved under an MLContext for one
device to be retrievable and usable by a different device.

- DISCUSS: How exactly do we want to deal with the cross-device case?  If loads can fail
  "for any reason" then it leaves it up to the implementation whether models can be re-used
  across devices or now.  However, suppose for some reason an application switches between two
  different devices frequently.  Then, storing a model under the same name in a different device
  erases all versions of the compiled model this may lead to model thrashing, so maybe we should
  specify separate caches for each device.

To allow for a "null" no-cache implementation, an implementation may choose
to not implement a cache at all and always throw exceptions for this method, even for
`MLGraph` objects that were just saved and are still live in the current session.

The `loadGraph` method will always throw an exception for an attempt to use the empty string
as a key.

## deleteGraph
Given the user-provided key used to save the model.  The associated model is removed from the cache
if it is present.
This method always succeeds, even if the model was not originally in the cache or is no longer in the cache.
Idempotent.

Subsequent loads for this key will always fail unless a model is saved again for the same key.

NOTE: an implementation may choose to delete saved models automatically for any reason.

## deleteSavedGraphs
Delete all graphs of this origin.  Always succeeds, even if the cache is empty.
Idempotent.

All subsequent loads will fail until new models are saved.

### Privacy Mitigations
- The WebNN API is already disallowed in third-party contexts, which mitigates
  several scenarios in which the API can be abused for tracking.
- Like other storage APIs, this API uses Storage Partitioning and should
  have similar restrictions as other storage APIs.  The use of arbitrary
  user-defined keys means this API is equivalent to a key-value store that
  (potentially) persists across sessions.

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
      recompilation but does not have the overhead of managing the data in Javascript.  This implementation also allows
      a cached model to be retargeted to other devices.
   3. Internal to the WebNN API, the internal representation of the compiled model, including the weights
      and the compiled kernels, is stored to a file so that it can be quickly retrieved and run on a
      target device.  This is faster since it avoids compilation overhead, but requires the ability
      to serialize/deserialize a compiled representation which may not be available for all backends.
      The representation is also device-specific.
   4. No cache. Loads always throw exceptions, and returned keys from stores are always the empty string.
- An implementation may choose to not store some models at all, and may also delete stored models at any
  time.  The simplest implementation (4 above) is to store no models at all.  This is
  acceptable and would be compatible with the proposal, although obviously would not address the use
  cases.  For systems with finite storage capacity, an implementation can automatically delete older
  models, in which case they would have to be reloaded by the developer.  An implementation may also
  choose not to store certain models for implementation-defined reasons, e.g. models that are too large.

## Design Alternatives
- This is a cache, so the storeGraph method is only a "hint" and an implementation may choose not to store
  anything.
- An alternative is an API that acts more like a file system, where stores succeed if there is space and fail
  if there is not.  In this case the developer would be responsible for deleting old models to make space as
  necessary - however, note the API does not provide any metadata on the stored models, e.g. age, size, etc.
  Also, in this case a "null" implementation is not possible - and this is useful as some backends may not
  support saving models at all.

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
