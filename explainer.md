# Hybrid AI Exploration
## Authors
* Michael McCool 
* Geoff Gustafson
* Sudeep Divakaran

## Introduction
ML on the client supports many use cases better than server-based approaches, and with lower cost for the application provider. However, clients can vary significantly in capabilities. A hybrid approach that can flexibly shift work between server and client can support elasticity and avoid the problem of developers targeting only the weakest clients’ capabilities.

The **overall goal** of hybrid AI is to maximize the user experience in machine learning applications by providing the web developer the tools to manage the distribution of data and compute resources between servers and the client. 

For example, ML models are large. This creates network cost, transfer time, and storage problems. As mentioned, client capabilities can vary. This creates adaptation, partitioning, and versioning problems. We would like to discuss **potential solutions** to these problems, such as shared caches, progressive model updates, and capability/requirements negotiation.

## Requirements and Goals
For the end user, most of the existing WebNN use cases share **common user requirements**:
* **Enhance User Experience**
    * Reduce load times
    * Meet latency targets for human interactions
* **Portability and Elasticity**
    * Minimize compute, storage, and network transfer costs
    * Support clients of different capability levels, including older/newer clients
    * Adaptive to varying resource availability
* **Data Privacy**
    * User choice for location of data storage and computation
    * Video and audio streams both high bandwidth and generally private
    * Personal data (personally-identifiable information)
    * Confidential business information

Even though it is not a primary requirement, **developer ease of use** is a factor for adoption. An approach that easily allows a developer to shift load between the server and the client using simple, consistent abstractions will allow for more Hybrid AI applications to be developed faster than one with completely different programming models.

## Open Issues
Current implementations of hybrid AI applications (see User Research and References) have the following problems when targeting many of the WebNN use cases:
* **If the model runs on the server**, then large amounts of (possibly private) data may need to be streamed to the server. This incurs a per-use latency.
* **If the model runs on the client**, large models need to be downloaded, possibly multiple times in different contexts. This incurs a startup latency.
* **Users need control over private data**, so the choice of whether or not a model needs to run on the client may have to be overridden by the user's preferences in some cases.
* **Clients vary in capabilities**, so the developer does not know in advance how to split up the computational work.
* **Models are large and can consume significant storage** on the client, which needs to be managed.
* **Applications may use multiple models** that need to communicate with each other, but may each run on either the client or server.
* **Multiple applications may be present and need to share resources** such as storage, memory, and compute.  This may cause the actual capabilities of the client to vary over time.
* It may be necessary to **hide the exact capabilities of the client** from the developer to avoid fingerprinting. However, the platform must be able to **match models with client capabilities**. An application may provide a choice of multiple models to support elasticity. See [Performance Adaptation](https://www.w3.org/TR/webnn/#usecase-perf-adapt).

## Non-goals
* **Protecting proprietary models** downloaded to the client from interception is a non-goal (but may be addressed in implementations or later work).
* **Automatic factoring of models** is a non-goal. The developer needs to break models into pieces that are managed atomically, each of which runs on either the server or client.
* **Automatic optimization of models** is a non-goal. The developer needs to consider how to minimize the size of models; however they may be able to use generic features expected on clients such as small data types.
* **Model training** is a non-goal. The system will be focused on inference. However, fine tuning may be used in limited circumstances.
* **Complete modelling of the client’s capabilities** is a non-goal.
* **Extreme scalability** is a non-goal. While there may be multiple applications on the client there should be only a handful that a single user can use at once.
* **Extreme performance** is a non-goal. While important, other goals such as portability and security, which are also important to the user experience, create trade-offs.
* **Managing models outside the web client** is a non-goal. For example, internal platform models may be present and used to support system features but the proposed system would not manage them. However, *access* to those separately managed platform models may be useful.

## User Research and References
* [Existing WebNN Use Cases](https://www.w3.org/TR/webnn/#usecases) - Set of agreed-upon use cases for WebML.  Most of these have latency or privacy requirements and several require large models (e.g. language translation).
* [Storage APIs for caching and sharing large models across origins discussion in the WebML WG](https://www.w3.org/2023/09/21-webmachinelearning-minutes.html#t03) - Some previous discussion within the WebML on the problem of model caching. Includes discussion of experience with prototype and highlights cross-site sharing as a key issue.
* [Google on Hybrid AI](https://www.linkedin.com/comm/pulse/web-ml-monthly-16-1-billion-downloads-jasons-crystal-ball-jason-mayes-tcaac) - A general discussion of Hybrid AI based on using smaller models on client and falling back to server only when necessary.  Mentions need to cache models, and potentially breaking up model between client and server.  Mentions potential privacy benefits of partitioned models, surveys several applications already using hybrid approach.
* [Qualcomm - Getting Personal with On-Device AI](https://www.qualcomm.com/news/onq/2023/10/getting-personal-with-on-device-ai) - Describes several interesting personalized AI client use cases.
* [Moor Insights on AI PC](https://www.intel.com/content/www/us/en/business/enterprise-computers/resources/moor-insights-ai-pc-paper.html) - Describes the AI PC opportunity and lists several client AI applications, with a focus on Windows and Microsoft.
* [Priority of Constituencies](https://www.w3.org/TR/design-principles/#priority-of-constituencies) - The general W3C framework for prioritizing requirements.
* [WNIG Cloud-Edge Coordination Use Cases](https://w3c.github.io/edge-computing-web-exploration/) - Two explicit ML use cases but others may have ML aspects, e.g. video editing.
  While these emphasize edge computing (offload from the client) several can also be interpreted as use cases simply needing additional performance on the client.
