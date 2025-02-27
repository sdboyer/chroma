## Motivation

We want to foster an ecosystem of "knowledge packages," such that users can stitch together RAG programs by combining these packages with their own data. We will call such packages, and their format, "matrix." Drop in a matrix package - now your RAG app knows kung fu.

While many steps are necessary to reach this overarching goal, a clear initial step and the scope of this CIP is to define a knowledge package format and make the information held in Chroma portable into that format. The completeness criteria for this proposal is to allow a full round trip - chroma->matrix->chroma.

(Note that this CIP explicitly omits any actual dependency management behavior, treating that as a future problem to tackle.)

Ideally, the matrix package format could eventually be shared by the many tools in the emergent LLM/AI stack. This proposal seeks to balance that Chroma-agnostic goal with the reality that working, useful software is usually prerequisite to bootstrapping an ecosystem.
## Impact

This is a novel user path for Chroma, and as such does not affect any existing code or user behaviors. It may eventually affect workflows for developing and deploying Chroma-based apps, possibly even those that have no external dependencies.

The main impact is on maintenance burden and (possible) compatibility guarantees. By creating what can be thought of as a portable serialization format for most of what is ordinarily stored in Chroma, it is likely that some properties which previously could have been considered implementation details, now will be assumed by users to be part of Chroma's public API surface.

TODO examples

## Proposed Change

This is a proposal in two parts. First, we describe a package format. Then, we outline new chroma client capabilities and corresponding CLI subcommands that can round trip to and from this package format.

### Matrix Package Format
A matrix package is a filesystem format consisting of two files, co-occurring within the same directory. Each file serves a particular, well-defined role:

- `matrix.package.yaml`: defines the package name, version, and other common metadata, and certain metadata about the dataset in the package. Future iterations will add dependency declarations - the same kind of information that appears in `package.json` or `requirements.txt`. **Required**.
- `matrix.data.parquet`: contains the package's dataset. Each row closely mirrors Chroma db content, containing original document source, computed embeddings, and metadata. **Required**, but will likely become optional in the future (more below)
- TODO - how do we serialize a computed index? it doesn't have the same cardinality as the data, does it?

Most critical to note is that a matrix package contains *exactly one* dataset (see further discussion under [alternatives](#Alternatives-Considered)). For now, we can assume a 1:1 relationship between a matrix package and a Chroma collection. This simple correspondence may change once we allow dependencies between matrix packages.

Some of the design choices here are oriented towards the expected eventual case of files in a matrix package smoothly coordinating to form a sparse, distributed cache. More detail on the each file follows.
#### `matrix.package.yaml`

The following is an example `matrix.package.yaml` file, which we'll use to explain each property in the file:

```yaml
name: "matrixhub.com/somegroup/examplepackage"
version: "2023.09.08-1"
license: "Apache-2.0"

data:
  embeddings:
  - model: "hf://sentence-transformers/all-MiniLM-L6-v2"
    types:
      dimensions: 384
      quantize:
        bits: 8
        method: "k-means"
  digests:
    data: "sha256:decafbad"
    # index: "sha256:deadbeef" TODO do we need this?
```

##### `name`
```yaml
name: "matrixhub.com/somegroup/examplepackage"
```
The unique identifier of the package. With exactly one dataset per package, this can also be considered the name of the dataset.

The general intent of this naming pattern is to provide a familiar naming pattern (c.f. GitHub, HuggingFace), but ensure we're set up for a world where users/companies can easily run their own private registry to store their own private data.

A registry pattern like this would be complex to do from scratch, but is reasonably straightforward by relying on [OCI registries](https://github.com/opencontainers/distribution-spec/) as a base standard and protocol with battle-tested helper for building a server in Go, and client libraries in at least Python and Go. OCI registries will allow us to address the critical problem of storing and hashing large binary files - like parquet - as an internal part of the package.

[Every major hosting provider supports](https://oras.land/docs/compatible_oci_registries/) the OCI spec, so matrix users relying on these providers will have to do little to no additional work required in order to obtain private hosting for their artifacts.

TODO exact character restrictions

##### `version`
```yaml
version: "2023.09.08-1"
```
The version field defines the package's version.

Versioning schemes are a frustratingly subtle and complex topic. The simplest thing we can say about them is that [totally ordered](https://en.wikipedia.org/wiki/Total_order) version schemes are strongly preferable to those without. The format given above is a [CalVer](https://calver.org/) - that is, date-based versioning: `YYYY.MM.DD-X`, where `X` is an incrementing counter.

This format is more of a placeholder to facilitate discussion than a deeply considered proposal, and is at least as as notable for what it isn't as what it is:
- Dates provide a total order
- Day-level granularity with an incrementing counter that resets each day is broadly reasonable **IFF** the typical use case for matrix packages will be humans curating periodic releases through centralized processes (incrementing counter entails centralized process). But it may be a poor choice if we expect matrix packages to, for example, be AirFlow outputs.
- It's not [SemVer](https://semver.org/), because there's no obvious general rule we could use to differentiate between breaking and non-breaking changes to matrix packages.
- There's no space to wedge in arbitrary build identifiers or hash digests.

TODO flesh it out

##### `license`
```yaml
license: "Apache-2.0"
```
License contains an [SPDX license identifier](https://spdx.org/licenses/), specifying the license for the package and its contents.

Given all of the open questions ([for example](https://www.infoworld.com/article/3706091/rethinking-open-source-for-ai.html)) about AI training data and ownership, this could end up being a complex area that looks quite unlike the way software licensing has historically worked. However, there does not yet appear to be any alternative clear enough to justify deviating from SPDX standards.
##### `data`
`data` contains metadata about the package's dataset - the contents of `matrix.data.parquet`.

This is the meat of the matrix package, and before we go further, we need a brief detour into the underpinnings of package management.

The general principle within the `data` block is to describe correctness properties of the data, without necessarily specifying an exact procedure for arriving at those properties. The (handwavy) goal here is to achieve a distinction analogous to URIs vs. URLs, or types vs. values: enough information to be able to make bounded correctness statements, without being too prescriptive about implementation. It's **where** we're going, rather than **how** to get there.

Exactly how to apply this principle needs consideration for each section.

TODO how much of this can or should be encoded directly in the parquet file? is duplication OK? what would be the reason for having it here instead of or in addition to parquet?

##### `data.embedding`
```yaml
  embedding:
    model: "hf://sentence-transformers/all-MiniLM-L6-v2"
    types:
      dimensions: 384
      quantize:
        bits: 8
        method: "k-means"
```

The purpose of this stanza is to describe type-like properties of the vector space for the embeddings in the package. It is an attempt at "where, not how."

`embedding.model` tells us the model that produced the embeddings in the data file. A simple string identifier for the model that produced the embeddings is sufficient here to establish what amounts to a [nominal type system](https://medium.com/@thejameskyle/type-systems-structural-vs-nominal-typing-explained-56511dd969f4). The name of the model here can then be thought of as the name of the universe of weights defined by the trained model, and by extension the computed embeddings.

There is plenty of debate on the virtues of structural vs. nominal typing in the traditional software world. It often amounts to, "is it better DX to assume that 'if it quacks like a duck, it's a duck', or to have interface names implicitly carry some special meaning beyond their formally expressed properties?" For this case, nominal typing is a natural choice, because the opacity nominal typing matches the opacity of a trained model's weights.

This is closely related to, but more constrained than Chroma's embedding function - it names only the model, rather some known invokable function whose (opaque) behavior is to call that model.

TODO is this over-abstracting the notion of an embedding func, to the point where it's not useful at all?

`embedding.types` describes other knowable properties about the vector space in which the embeddings live - what's the dimensionality of the space? is it quantized? For Chroma's purposes, these descriptive properties should be treated as additional constraints on the [query pipeline](https://github.com/chroma-core/chroma/blob/6384b66bc19e6a6217c2c5aa55f3b539e30c2708/docs/CIP_6_Pipelines_Registry.md#query-pipelines) that must be applied when processing an input for vector search.

TODO i am totally out of my depth with quantization and need to run this through with someone. Key questions - does it make sense to try to reduce embedding and query pipeline fns to a model + some set of constraint arguments? Is quantization distinct from embedding - i.e. do we have one column for each in the parquet file?
TODO finish

#### `data.digests`
```yaml
  digests:
    data: "sha256:decafbad"
```

### `matrix.data.parquet`

The `matrix.data.parquet` file contains the core dataset of the matrix package. Each row in this Parquet file closely mirrors Chroma database content and includes the following columns:

- **Original Document Source**: This field holds the original source data, which could be text, documents, or any other relevant information that forms the basis of the dataset.
- **Computed Embeddings**: Computed embeddings are generated from the original document source using a specific model. These embeddings represent the data in a numerical format suitable for various analytical and machine learning tasks.
- **Metadata**: Metadata associated with each row provides additional context or information about the data. This metadata could include timestamps, data source references, or any other relevant details.

The `matrix.data.parquet` file is marked as required for most scenarios within the matrix package format. However, there is one specific case in which it may become optional: when introducing support for remote artifacts, facilitated by the use of OCI registries. In this scenario, the actual dataset might be stored remotely, and the matrix package would include references or pointers to the remote location rather than including the dataset itself. This approach would reduce package size and enhance efficiency in distributing and managing large datasets, making the data file optional in such cases.

### Chroma Client & CLI Changes

For matrix packages to be useful, the procedure for producing and consuming them must be trivial. This section describes changes to both the Chroma client and CLI for

TODO lollll so much to fill in

#### Import and `chrome import-matrix`

Idempotence is probably critical here

TODO finish

#### Export and `chrome export-matrix`

TODO finish

## Alternatives Considered

TODO finish

### Alternate naming patterns

Other naming patterns for matrix packages are possible. The form of name chosen has far-reaching impacts on how packages are distributed and consumed.

TODO finish

### Multi-dataset packages
The proposal allows for only a single dataset in a matrix package.

TODO finish

### Multi-embedding datasets
The proposal allows for only a single set of embeddings in a matrix package.

TODO finish

### Appendix: On Applying Package Management to AI

Traditional package ecosystems facilitate a relationship where the particular domain expertise of one person or group, as expressed in the contents of a package they maintain, can be composed into larger programs by others without requiring them to fully comprehend the details the particular domain.

For example, i can rely on `pyarrow.parquet`'s `read_table()` to read parquet files, delegating to the experts knowledge of exactly how that's done. Easy enough - but only because we all, package author and consumer, already know what a "python function" is and can possibly be responsible for, and are expecting to share objects of this type.

This question of exactly what is being shared is the implicit substrate on which every package ecosystem is built. It must be obvious, to the point that discussing it directly seems a bit absurd, for a package ecosystem to be functional. Only then can it impose a sufficiently low cognitive load on both package authors and consumers that open sharing and free exchange becomes maintainable and realistic.

For matrix, what's being shared is:

- A dataset
- An expert opinion on what the optimal representation of embeddings and metadata is for that dataset
- The actual embeddings (and some other computed results)

This is immediately problematic. The best traditional software packages find a way to define "optimal implementation" for their domain problem in a way that's largely independent of their dependers' use case. In interacting with LLMs, however, it's unclear the extent to which an "optimal embedding representation" can be expressed independent of a particular use case - domains don't nest tidily.

That we are pursuing matrix packages at all is, therefore, a bet: we think that it's possible to reach a point of ease, clarity, and utility in these packages such that an ecosystem can take flight.
