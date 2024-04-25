# Requirements
Note: Currently just a copy of "pain points" from presentation in https://github.com/webmachinelearning/hybrid-ai/pull/4 - needs elaboration.

Generally: saving space/download latency
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
