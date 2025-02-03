# Adapters

## Problem Statement
Some AI models, such as LLMs, are very large and this poses various challenges for using them in the web environment as dynamically downloaded models.
This is exacerbated by the web design principle of Storage Partitioning (Reference 6),
which seeks to isolate the local client storage for each origin to avoid cross-site tracking privacy risks such as fingerprinting.
At the same time, use of AI models locally can provide privacy benefits by avoiding sending sensitive data to the cloud.

One way to mitigate the fingerprinting risk is to bind the locally-available LLMs to the browser version,
so that knowledge of the availability of a particular LLM model provides no additional information
over that already provided by the browser version.
In general, the more common a model is the less fingerprinting information it provides, so other
mechanisms that allow for sharing of large base models can also mitigate fingerprinting risks.

However, relying on a single or a small number of base model(s) is also inflexible,
and does not allow an origin's developer to adapt the model to their needs.

A possible solution is to use "adapters", which represent a fine-tuned LLM (or other AI model)
with using a base model modified by a "delta" which has a relatively small number of weights.
This is also called parameter-efficient fine tuning (PEFT), as when training for fine-tuning the base model is
frozen and only the parameters in the adapter are trained.
The delta is typically represented in a compressed form,
the most common of which is LoRA (low-rank approximation - see Reference 1), although other representations of the delta
are possible (LoHa, LoKr, OFT, BOFT, AdaLoRA - see Reference 2).
LoRA adapters are additive to the base tensors they are applied to.
However, some of the other adapters, such as OFT, modify the base tensor multiplicatively.
To be general, we will talk about an adapter "modifying" the base tensor it applies to.

Because of the compressed representation used,
adapters are generally one or two orders of magnitude smaller than base models.

The following is described as an extension to the WebNN API (Reference 7), which provides an
API to build opaque representations of models represented by `MLGraph` objects.  Our proposal
provides a means to modify existing `MLGraph` objects (however constructed) with an adapter,
resulting in a new (conceptually distinct) `MLGraph` object.

## Use Cases
The use cases of adapters are as follows:
1. The base model is loaded first and stored using existing web client storage mechanisms (Reference 3),
   then adapters are loaded to adapt the base model to different purposes (e.g. summarization,
   planning, etc.).
      - There may be more than one adaptation used in a given session.
      - This use case *could* be implemented in a library but this would require rebuilding
        the base model from data stored on disk upon every page load, and in fact upon each adaptation
        (since the WebNN API "consumes" input data), and would also limit implementation options e.g. for
        quantized models.
      - With the proposed API, allowing modification of an already compiled model,
        there is the opportunity to provide an additional API to serialized/deserialize or otherwise recall
        a pre-compiled graph (if this is done from existing storage mechanisms there is no additional fingerprinting risk),
        avoiding the compilation cost and providing more implementation flexibility.
2. The base model, represented as a precompiled `MLGraph` object,
   is retrieved from a (proposed, new, model-specific) cache (Reference 4) and then the adapter is applied to
   modify it for a specific purpose.
      - In this case the cache implementation, if it allows for
        cross-site sharing, may implement some fingerprinting mitigation mechanisms but the adapter could be
        stored per-origin, avoiding the need for these mitigations.
3. The base model is bound to the browser and is shared cross-site,
   with the privacy risk mitigated as noted above.
      - Adapters can then be stored per-origin, using normal local web caching mechanisms.
      - This could be an extension of the proposed Prompt API (Reference 5).
      - This extension could either allow the `MLGraph` object for a "bound" model to be fetched or
        allow application of an adapter using a mechanism consistent with what is proposed here.
      - However, note that adapters are model specific and the actual model used may vary by browser,
        so the base model and/or browser version would have to be checked before the adapter is selected and fetched.

We focus on adapter mechanisms that modify the parameter tensors of existing models,
as opposed to other kinds of adapters that modify the embedding used by a model or make other
changes to its evaluation.

## Design Considerations
What is missing in the existing APIs are the following capabilities:
1. A way to retrieve a base model.
2. A way to apply an adapter to a base model to obtain a derived model.

Adapters need to be applied to corresponding weight tensors in the base model,
so it is not possible to specify an adapter without knowing at least some of the structure of the base model.
However, to support proprietary models, we want to limit this information as much as possible.

There are a few other considerations:
- Subset: Not all weight tensors in a model may need to be modified. An adapter may only apply to a subset.
- Locking: A model provider may want to protect certain parts of a model from modification.
- Encoding: LoRA is a very common adapter representation but is not the only possibility.
- Data Type: An adapter may use a different data type or quantization scheme than the base model.
  
## Retrieving the Base Model
In the following sections we will describe, for each use case, how to accomplish step 1,
retrieving the base model.
In later sections, we assume that the `MLGraph` object of the base
model has been retrieved by any one of these mechanisms.  

### Use Case 1: Rebuilding Base Models
For Use Case 1, we *could* rebuild a base model from data stored 
locally e.g. in an IndexedDB API or via the File System Access API, and then
apply an adapter to it, but this incurs extra costs to recompile the base
model every time the page is loaded (or at least to recompute a hash to check
if the model is already in an internal cache).   It may be worth considering
another API extension to serialize/deserialize a compiled model to and from
external storage (this would be useful to improve load times even if adapters are 
not used).

### Use Case 2: Cached Models
For Use Case 2 are working on a proposal to support a model-specific cache that can store
compiled models (Reference 4). 
In summary, this proposal allows storing a model in the cache, which then returns
a hash of the model that can be used as a key:
```js
const hash = aimm.cache.store(graph)
```
Later, the model can be retrieved from the cache using the hash:
```js
graph = await aimm.cache.fetch(hash);
```
Hashes are deterministic so it is also possible to treat the hash as the name of 
a specific model that may or may not already be in the cache:
```js
graph = await aimm.cache.fetch("2ff2366c-3acb-4a58-93ac-53971b0e1b18");
```
Use of a data-dependent hash as a name avoids issues
with cross-site caches being used to exfiltrate data between sites, as well
as validating the model.

### Use Case 3: Browser-Bound Models
To support browser-bound models,
the proposed APIs could be extended to retrieve such models given an identifier for that model.
In the following we assume an API is provided to retrieve a model, represented as an `MLGraph` object.

For example, for Use Case 3 the API to fetch the graph for a bound model could be extended to 
support the following:
```js
graph = await ai.fetch("gemini-nano");
```
Here `graph` is an `MLGraph` object and `context` is an `MLContext`.
If this is not acceptable, later we will describe an alternative that can apply the adapter to the 
bound model directly.

## Example Model
To be concrete we will use the following explicitly-built example model (which falls under Use Case 1)
which is based on the simple example in the WebNN specification.  This extract of the 
code has only been modified to add labels to each of the nodes, which we need in
our proposed API to apply an adapter.  Note that this is NOT an LLM - adapters
are a general mechanism that can be applied to any model!
```js
// Set up Graph Builder
const TENSOR_SHAPE = [1, 2, 2, 2];
const TENSOR_SIZE = 8;
const builder = new MLGraphBuilder(context);
const desc = {
  dataType: 'float32',
  shape: TENSOR_SHAPE
};

// Specify all the nodes in the graph; note labels.
const constantBuffer1 = new Float32Array(TENSOR_SIZE).fill(0.5);
const constant1 = builder.constant(desc, constantBuffer1, {label: "const_1"});
const input1 = builder.input('input1', desc);
const constantBuffer2 = new Float32Array(TENSOR_SIZE).fill(0.5);
const constant2 = builder.constant(desc, constantBuffer2, {label: "const_2"});
const input2 = builder.input('input2', desc);
const intermediateOutput1 = builder.add(constant1, input1, {label: "add_1"});
const intermediateOutput2 = builder.add(constant2, input2, {label: "add_2"});
const output = builder.mul(intermediateOutput1, intermediateOutput2, {label: "mul_1"});

// Compile the constructed graph.
const graph = await builder.build({'output': output});
```

We also assume the internal (compiled) represention of models allows for weight updates.
Conceptually, however, the base model is not changed and a new model (with a new `MLGraph` object
representing it) is made available after the adapter is applied.

## Loading the Adapter
We propose that an adapter is represented to the API as an dictionary of (label, delta) pairs.
First we need to load the tensors used for a LoRA approximation.  In this case the only
tensors in the (overly-simple) example graph are the two input constants.  Although we labelled
all the nodes we only need to modify those specific nodes:
```js
const adapter_const_1_a = await fetchConstantByNpy(baseUrl + 'const_1_a.npy');
const adapter_const_1_b = await fetchConstantByNpy(baseUrl + 'const_1_b.npy');
const adapter_const_2_a = await fetchConstantByNpy(baseUrl + 'const_2_a.npy');
const adapter_const_2_b = await fetchConstantByNpy(baseUrl + 'const_2_b.npy');
```
In the existing library [9], the function `buildConstantByNpy` returns a WebNN node directly
using the graph builder object provided as an argument.
In the implementation of that function a buffer is first populated and only in the last line is it
converted to a WebNN node.  Here `fetchConstantByNpy` is a proposed function that returns
raw data to build a node, not the node itself, and so does not require a builder argument.

In the LoRA encoding, the representation of each delta requires two tensors.  The outer product
of these tensors is then used to reconstruct the delta, which is added to the base tensor.
## Constructing the Adapter
Then we can construct a dictionary representing the adapter:
```js
const adapter = {
   "const_1": {
       "encoding": "lora",
       "a": adapter_const_1_a,
       "b": adapter_const_1_b
   },
   "const_2": {
       "encoding": "lora",
       "a": adapter_const_2_a,
       "b": adapter_const_2_b
   }
};
```

## Applying the Adapter
Now, assuming we have a `MLGraph` in hand representing the base model, we can apply the adapter.
```js
const adapted_graph = await graph.adapt(adapter);
```

If we want to apply an adapter to a browser-bound model and we do not want to have to return
an `MLGraph` object as proposed above, we could alternatively extend the browser-bound AI model
API to allow the adapter to be applied directly:
```js
const session = await ai.createTextSession({
  "model": "phi-3-mini",
  "adapter":  my_phi_3_mini_adapter
});
```
Here the `my_phi_3_mini_adapter` would have to be aligned with the structure of the base model.
This assumes the base model is known or can be specified in the Prompt API.

## Adapter API Notes
- This API assumes we are just copying and updating the weights but not changing the compiled
  code, so we want to use the same device etc. specified in the context the original graph was 
  compiled under.
  If we *did* want to repurpose a compiled graph and move it to another device it would be more
  appropriate to propose a separate API for that.
- A "label" needs to be assigned to every modifiable node in the base model.
  The same labels already included in the API for debugging purposes can be used (Note 2).
- Not providing a developer-specified label for a node in a base model would "lock" that node and prevent it from being modified (Note 3).
- The current specification uses "may" terminology for labels.
  If storing label with compiled models is optional then adapters would also have to be optional and not supported if labels are not.
- The API allows an adapter "encoding" to be chosen (Note 4),
  which specifies how each delta is represented.
  In the short term we would only support one approach, "lora", but including an identifier will allow for future generalization.
  The proposed representation also allows mixing adapter encodings, allowing different approaches to be used for different nodes.
- To simplify matters, we might want to require adapters to match the data type of the base model.
- For the "lora" representation, the delta would be represented as a pair of tensors,
  the outer product of which should have the same size as the weight tensor in the base model identified by the "label".
  In general, the data provided for the adapter delta would depend on the adapter encoding chosen.
- The "label" approach allows a subset of the nodes in the base model to be adapted.
  It also hides some of the details of the base model, in particular the operation and the actual weights,
  as well as the connection topology.
  It is necessary however to reveal at least the size of the weight tensors.
- When the adapter is applied, there is an implied asynchronous step which would compile the new model.
  An adapter application would, as with building a model from scratch,
  "consume" (transfer ownership) of the memory used to represent the adapter representation.
  
## Implementation Note
There are two ways to apply an adapter to a model: 
- Copy: The adapter can be added to the weights, giving a new set of weights.
  To avoid modifying the original base model a copy of it needs to be made first.
  The new model takes the same amount of time to evaluate as the original (Note 1).
  The adapted model can be managed in a per-origin cache (however, since only the
  "compiled" model is available, one of the extended mechanisms mentioned would
  need to be used, e.g. to serialize/deserialize a compiled model, store it is in
  model cache, etc).
- Dynamic: The adapter is evaluated dynamically as the model is evaluated.
  This avoids copying the base model, and so would require less storage,
  but requires more computation,
  and the compilation of the base model would have to either be updated or
  a version would have to be compiled taking into account a potential adapter.
  However, this approach can also work with quantized base models without
  having to "unpack" them.
The proposed API permits either implementation approach.

## Design Alternatives
There are a few possible alternatives to discuss:
- Derived models.
  Should it be possible to apply an adapter to a model that already has an adapter applied to it?
  For the "copy" implementation this is straightforward, but again uses more storage.
  For the "dynamic" implementation such models would incur additional computation for each new adapter.
- Quantization.
  To save space, an adapter may be downloaded in a quantized or otherwise compressed form.
  If we do not support different data types in the adapter API a library could still be used to
  convert a compressed adapter to a form compatible with a base model.

## Notes:
1. Unless the adapter changes the sparsity structure of the base model.
   As WebNN does not currently support sparse representations this is not (yet) a factor.
2. Joshua Bell, [Add optional operator labels for more diagnosable error messages](https://github.com/webmachinelearning/webnn/pull/742)
   based on Yajing Tang, [Consider adding node labels for more diagnosable error messages for async errors.](https://github.com/webmachinelearning/webnn/issues/585).
4. We may, however, want to separately lock nodes using some other mechanism if we still want to expose locked nodes for debugging.
5. This can be global, although there might be some benefit from allowing each node's modification to use a different representation.
   The simplest approach is a global choice. To discuss.

## References
1. Edward J. Hu, Yelong Shen, Phillip Wallis, Zeyuan Allen-Zhu, Yuanzhi Li, Shean Wang, Lu Wang, and Weizhu Chen. 
   [LoRA: Low-Rank Adaptation of Large Language Models](https://arxiv.org/abs/2106.09685), 
   2021. 
2. Hugging Face,
   [PEFT Documentation: Adapters](https://huggingface.co/docs/peft/en/conceptual_guides/adapter).  
3. Thomas Steiner, Google,
   [Cache AI Models in the Browser](https://developer.chrome.com/docs/ai/cache-models). 
4. Michael McCool, Geoff Gustafson, Sudeep Divakaran, and Muthaiah Venkatachalam.
   [AI Model Management](https://github.com/webmachinelearning/hybrid-ai/blob/main/presentations/WebML%20Discussion%20-%20Hybrid%20AI%20for%20the%20Web%20-%20AI%20Model%20Management%20TPAC%202024%20Breakout.pdf).
   September 25, 2024, W3C TPAC 2024 Breakout.
   NOTE: Replace/extend with link to more complete explainer/proposal when available.
5. Domenic Denicola, Thomas Steiner, et al, Google, [Explainer for the Prompt API](https://github.com/explainers-by-googlers/prompt-api).  
6. W3C Privacy CG. [Client-Side Storage Partitioning](https://github.com/privacycg/storage-partitioning).
7. Ningxin Hu and Dwayne Robinson (Editors), W3C Web Machine Learning WG. [Web Neural Network API](https://www.w3.org/TR/webnn/).
8. WebNN-Samples, [nsnet2.js](https://github.com/webmachinelearning/webnn-samples/blob/master/nsnet2/nsnet2.js)
9. WebNN-Samples Utilities, [utils.js](https://github.com/webmachinelearning/webnn-samples/blob/master/common/utils.js).
   The modification of the code for `buildConstantByNpy` to `fetchConstantByNpy` would change the last line to
   `return [{dataType: type, dimensions: shape, shape}, typedArray];` and would remove `builder` from the parameter list.
   The function `buildConstantByNpy` can then be implemented easily in terms of it.
