# Requirements (DRAFT)
See also "pain points" from [Hybrid AI Requirements presentation on 2024-04-04](https://github.com/webmachinelearning/hybrid-ai/blob/main/presentations/2024-04-04-Hybrid_AI_for_the_Web-Requirements.pdf).  Marked as draft since we may still want to revise these.

## General Goals: 
* Saving space
* Minimizing download latency

## Detailed Goals:
1. Sharing/reusing large models
   - Across sites
2. Same resources at different URLs
3. Same resources in different formats
4. Want to expose "built-in" models
5. Generalizing models
   - Categories (e.g. "en-jp text translation")
   - Versions (e.g. ">= 1.3.*")
6. Need solution that can handle adapters
   - Applications sharing a base model
   - Foundational models are large
   - Adapters are small
  
## Additional Candidate Goals
1. Supporting model updates

## Future Work
Note: Most of the above currently addresses the problems of "model management", not
distributing models between client and server (e.g. "hybrid AI").

