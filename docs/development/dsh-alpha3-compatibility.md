# DSH 0.2.1-alpha.3 compatibility

Exact upstream commit: `2fbf4359b82fb94c03dcce0b831660d87f12ea5c`.

Compared with alpha.2, the LLM runtime adds `imageRequestTarget` and invokes the registered adapter method. DVR supplies the upstream identity default only to its own adapters when the Host exposes that capability. The existing public Host minimum is unchanged.

Validation is in progress. No release is authorized.
