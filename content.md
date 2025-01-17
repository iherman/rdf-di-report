# Introduction

The [Verifiable Credentials Working Group](https://www.w3.org/groups/wg/vc) is finalizing the [[[vc-data-integrity]]] [[vc-data-integrity]] specification and the related "cryptosuites" specifications ([[[vc-di-ecdsa]]] [[vc-di-ecdsa]] and [[[vc-di-eddsa]]] [[vc-di-eddsa]]), whose goal is to provide standard means to secure Verifiable Credentials [[vc-data-model-2.0]]. The details of Verifiable Credentials are not important here (the interested reader may want to look at the [[[vc-overview]]] [[vc-overview]] document); for our purposes it is enough to say that Verifiable Credentials are defined as RDF datasets [[rdf11-concepts]] serialized in [[json-ld11]]. These datasets are secured by generating
separate _proof graphs_ containing a cryptographic signatures of the dataset. The signatures themselves are expressed using a small [vocabulary](https://w3id.org/security) defined by the [[vc-data-integrity]] specification.

The [[[vc-data-integrity]]] standard uses a JSON-LD terminology and relies on some specificities of Verifiable Credentials (more about this later). However, from the very start, an unofficial goal was to make the standard, the underlying vocabulary, and the cryptosuites also usable for general RDF Datasets. The question is, therefore: _"Is it possible, using the [[[vc-data-integrity]]] specification, to secure, via cryptographic signatures, RDF Datasets in general?"_.  The goal of this document is to give some answers, and some additional comments, based on an experimental implementation [[rdfjs-di]] by the author of this report. As we will see later, the answer to the question is _"Almost"_.

## Technical basics

To make this document understandable, some high level overview on the technical approach taken by [[vc-data-integrity]] is necessary.

The operations of Data Integrity are conceptually simple. To create a cryptographic proof, the following steps are performed: 1) Transformation, 2) Hashing, and 3) Proof Generation.

1. _Transformation_ takes the input data and prepares it for the next step. For RDF Datasets this involves the canonicalization of the RDF Datasets using the [[[rdf-canon]]] [[rdf-canon]] specification.
2. _Hashing_ calculates a hash value for the transformed data using a [cryptographic hash function](https://en.wikipedia.org/wiki/Cryptographic_hash_function). For RDF Datasets, the details are also defined by the [[rdf-canon]] specification.
3. _Proof Generation_ means to generate a cryptographic signature, using the secret key of an asymmetric crypto key pair, of the hash value calculated in the previous step. The details, i.e., the general algorithmic steps, the definition of the necessary vocabulary, etc., are defined in [[vc-data-integrity]] specification. The details of the algorithms, using standard cryptographic schemes, are defined in [[vc-di-eddsa]] and [[vc-di-ecdsa]]. Using other cryptographic schemes (e.g., RSA variants) is also possible by adapting, say, the [[vc-di-eddsa]] specification.

The result of these steps is a _proof graph_ that contains the public key for the signature (or a reference thereof), the signature value itself, and some metadata. 

_Verification_ of a proof involves repeating steps (1) and (2) above yielding the hash value, extract the proof value and the public key from the proof graph, and cryptographically check, using the public key, the validity of the signature.

<p class=note>
As part of the experimentation, [[rdfjs-di]] also offers an RSA based cryptosuite. The only major discrepancy, compared to [[vc-di-eddsa]] and [[vc-di-ecdsa]], is the fact that no specification exists to store RSA keys in <a data-cite="controller-document#Multikey">Multikey</a> [[controller-document]]; as a consequence, only <a data-cite="controller-document#JsonWebKey">JWK keys</a> are allowed for RSA (whereas the EDDSA/ECDSA keys are supposed to be stored in <a data-cite="controller-document#Multikey">Multikey</a>).
</p>
<p class=note>
The Verifiable Credentials specifications also define cryptosuites with selective disclosure facilities, namely <a data-cite="vc-di-ecdsa#ecdsa-sd-2023">ecdsa-sd-2023</a> [[vc-di-ecdsa]] and <a data-cite="vc-di-bbs#bbs-2023">bbs-2023</a> [[vc-di-bbs]]. These have not (yet?) been implemented in [[rdfjs-di]]. The reason, apart from the complexity of these algorithms, is that their specifications do not translate to RDF Datasets without a change. Indeed, these cryptosuites are inherently dependent on JSON: the choice of the disclosed properties is based on JSON Pointers [[rfc6901]]. That part of the specifications should be re-defined for RDF Datasets. (E.g., using a subset of SPARQL [[sparql11-overview]] as a query language.)
</p>

# Generating proof graphs

Generating proof graphs, following the general principles and the standard specifications is possible without further ado. There is nothing Verifiable Credential or JSON-LD specific in those steps.

A simple example is as follows. Consider the following, very simple RDF graph:

```turtle
@prefix foaf: <http://xmlns.com/foaf/0.1/> .
@prefix earl: <http://www.w3.org/ns/earl#> 

<https://www.ivan-herman.net/foaf#me> a foaf:Person, earl:Assertor;
  foaf:name     "Ivan Herman";
  foaf:title    "Implementor";
  foaf:homepage <https://www.ivan-herman.net/> .
```

The corresponding proof graph, as generated by [[rdfjs-di]], and using the EdDSA signature scheme [[RFC8032]], is as follows:

```turtle
@prefix sec: <https://w3id.org/security#>.
@prefix xsd: <http://www.w3.org/2001/XMLSchema#>.

<urn:uuid:507c894d-588e-41db-94a8-356d2a4d7426> a sec:DataIntegrityProof;
    sec:verificationMethod <urn:uuid:b5b6a487-9ea9-43da-9ea3-c56e85d519c4>;
    sec:created "2024-09-11T13:37:46.904Z"^^xsd:dateTime;
    sec:proofPurpose sec:authenticationMethod, sec:assertionMethod;
    sec:cryptosuite "eddsa-rdfc-2022";
    sec:proofValue "z4Dy2pmoHKRaAxKuweHJDc6yTkJFh5yi47AJcryaPAD19me6MdjtbEn6yJLyQNvUJAWYZJsKgTyym3X5AnTCjjAta".  

<urn:uuid:b5b6a487-9ea9-43da-9ea3-c56e85d519c4> a sec:Multikey;
    sec:controller <https://example.org/key/#ivan_eddsa>;
    sec:expires "2055-02-24T00:00:00Z"^^xsd:dateTime;
    sec:publicKeyMultibase "z6MkqPUPcdvvixcfgGqqEZJ4WZTiDwaCsqF8jHqR5UZA2iae".
```

The first resource identifies the signature itself (the cryptographic value being the object of the `sec:proofValue` property) with some additional metadata. The second resource identifies the public key (encoded using the <a data-cite="controller-document#multikey">Multikey</a> [[controller-document]] format). The `sec:verificationMethod` provides a "bridge" between these two resources, connecting the signature to the public key to be used for verification.

# Relating RDF Datasets with their signatures

The previous section may suggest that there is no real issue with the usage of Data Integrity for RDF Datasets, so the answer to the question of the introduction should be "Yes". However, there are questions that should also be answered and which do lead to problems for the general RDF case. These are:

1. How do we specify the triples to be signed?
2. How do we connect that Dataset to its proof graph?

For the case of a Verifiable Credentials, the answers are straightforward and are based on the nature of Credentials. The graph of a Credential is usually "tree-like", with the credentials' identifier identifying the root node; it is therefore perfectly fine to consider the tree itself as the content to be signed, and link the root of the tree with the proof graph. Consider the following, simple Credential (where the `id` property is an alias to the `@id` JSON-LD keyword):

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://www.w3.org/ns/credentials/examples/v2"
  ],
  "id": "http://university.example/Credential123",
  "type": ["VerifiableCredential", "ExampleAlumniCredential"],
  "issuer": "did:c276e12ec21ebfeb1f712ebc6f1",
  "validFrom": "2010-01-01T10:32:24Z",
  "credentialSubject": {
    "id": "did:example:ebfeb1f712ebc6f1c276e12ec21",
    "name": "Pat",
    "alumniOf": {
      "id": "did:example:c276e12ec21ebfeb1f712ebc6f1",
      "name": "Example University"
    }
  }
}
```

`http://university.example/Credential123` refers to the Credential itself, and the triples emanating, directly or indirectly, from that resource form the RDF graph to be signed. Furthermore, the Data Integrity specification introduces a `proof` property: the subject of the triple is the Credential itself, and object is a _named graph_ that contains a single proof graph. [](#info-graph-vc) (borrowed from the [[vc-data-model-2.0]] specification) shows the resulting RDF graphs with a signature for the Credential above:

<figure id="info-graph-vc">
<img style="margin: auto; display: block; width: 100%;" src="diagrams/vc-graph.svg" alt="Diagram with a collections of
claims for a 'verifiable credential graph' on top
connected via a proof property (or predicate) to a 'verifiable credential proof
graph' on the bottom. The claims for a verifiable credential include 'Credential
123' as a subject with 4 properties: 'type' of value ExampleAlumniCredential,
'issuer' of Example University, 'validFrom' of 2010-01-01T19:23:24Z, and
credentialSubject of Pat, who also has an alumniOf property with value of
Example University.  The verifiable credential proof graph has an object
'Signature 456' subject with 5 properties: 'type' of DataIntegrityProof,
'verificationMethod' of Example University Public Key 7, 'created' of
2017-06-18T21:19:10Z, a 'nonce' of 34dj239dsj328, and 'proofValue' of
'zBavE110…3JT2pq'. The verifiable credential graph is also annotated with the
parenthetical remark '(the default graph)', the verifiable credential proof
graph is annotated with the parenthetical remark '(a named graph)'.">
<figcaption style="text-align: center;">
Information graphs associated with a basic verifiable credential
</figcaption>
</figure>

In fact, what the [[vc-data-integrity]] specification defines is an "embedded" proof: the original dataset and its proof graph are presented together as one dataset with the `proof` property connecting the two.

However, for general RDF Datasets, there isn't any obvious way to _identify_ (as a resource) a set of triples or quads. The graph may be a forest or a more complex directed graph; it may be part of a Dataset as a separate named graph or not, it may be in a separate file or part of a triple store, etc. The identification of a graph or a dataset is not really part of the RDF Model.

<p class=note>
A pragmatic approach would be to use the <em>URL of, e.g., a file</em> as an identification of a graph or a dataset contained therein (when applicable). This is, obviously, a valid approach, but also raises RDF modeling questions: does that URL really <strong><em>identify</em></strong> the graph or the dataset? What happens if the URL changes but the content remains unchanged? How does this relate to the notion of a Default Graph for RDF datasets (if the resource contains a single graph)?
</p>

It is for this reason that the answer to the general question, i.e., "Is it possible, using the [[[vc-data-integrity]]] specification, to secure, via cryptographic signatures, RDF Datasets in general?" is "Almost". Any specification that aims to generalize the [[vc-data-integrity]] approach for general RDF Datasets, must provide some answers to those questions and should provide means to connect a dataset to its proof and, vice versa, a proof to the dataset it signs.

<p class=note>
This issue was raised by Henry Story on a GitHub repository of the Verifiable Credentials Working Group: see <a href="https://github.com/w3c/vc-data-model/issues/1248">issue #1248</a>. The issue was closed without any resolution: it was recognized as a general problem whose solution would go beyond what the Verifiable Credential Working Group was chartered to do, namely to provide a securing mechanism <em>for Verifiable Credentials</em>.
</p>

## The [[rdfjs-di]] approach

The approach taken by [[rdfjs-di]] to handle these issues is pragmatic: let the application control these problems. 

The basic [API](https://iherman.github.io/rdfjs-di/modules/index.html) takes an abstract <a data-cite="rdfjs-dataset#dom-datasetcore">`DatasetCore` instance</a> (defined by the [[[rdfjs-dataset]]] specification) as an input parameter representing an RDF Dataset. This leaves it to the application layer to decide which triples or quads must be signed. Using this API the application generates a detached proof graph, which can then be used in an application-dependent manner.

Another API entry generates an "embedded" proof graph as defined in the [[[vc-data-integrity]]] specification: a new RDF Dataset that includes the original input Dataset _and_ the proof graph as a separate named graph. If the input arguments include an explicit reference to an "anchor" resource, it is used as the subject of a proof triple (using the `proof` property) linking the anchor to the proof. The following, embedded proof can be generated from the original example: 

```turtle
@prefix sec: <https://w3id.org/security#>.
@prefix xsd: <http://www.w3.org/2001/XMLSchema#>.
@prefix foaf: <http://xmlns.com/foaf/0.1/>.
@prefix earl: <http://www.w3.org/ns/earl#>.

<https://www.ivan-herman.net/foaf#me> a foaf:Person, earl:Assertor;
    foaf:name "Ivan Herman";
    foaf:title "Implementor";
    foaf:homepage <https://www.ivan-herman.net/>;
    sec:proof _:b0.

_:b0 {
    <urn:uuid:507c894d-588e-41db-94a8-356d2a4d7426> a sec:DataIntegrityProof;
        sec:verificationMethod <urn:uuid:b5b6a487-9ea9-43da-9ea3-c56e85d519c4>;
        sec:created "2024-09-11T13:37:46.904Z"^^xsd:dateTime;
        sec:proofPurpose sec:authenticationMethod, sec:assertionMethod;
        sec:cryptosuite "eddsa-rdfc-2022";
        sec:proofValue "z4Dy2pmoHKRaAxKuweHJDc6yTkJFh5yi47AJcryaPAD19me6MdjtbEn6yJLyQNvUJAWYZJsKgTyym3X5AnTCjjAta".  

    <urn:uuid:b5b6a487-9ea9-43da-9ea3-c56e85d519c4> a sec:Multikey;
        sec:controller <https://example.org/key/#ivan_eddsa>;
        sec:expires "2055-02-24T00:00:00Z"^^xsd:dateTime;
        sec:publicKeyMultibase "z6MkqPUPcdvvixcfgGqqEZJ4WZTiDwaCsqF8jHqR5UZA2iae".
}
```

The structure is identical to the one of [](#info-graph-vc): the proof graph is in a separate named graph referenced by `_:b0`; this reference is also the object of the triple with the `proof` property.

This pragmatic answer may be the best approach for RDF Datasets in general, with specific applications, or application classes, setting up their own rules. From that viewpoint, it is what the [[vc-data-integrity]] specification does for Verifiable Credentials.

# Algorithmic efficiency

Another aspect that must be considered is the efficiency of the data integrity approach for graphs in general. A fundamental difference between Verifiable Credentials and general RDF Datasets is the possible size of RDF Graphs. Verifiable Credentials are small, in the range of a few dozen RDF quads. Compare it to, say, some of the medical ontologies which, when represented in RDF, may contain several hundreds of thousands triples or more.

<p class=note>
The case of biomedical ontologies is interesting, because securing their integrity is a possible use case. 
For example, the <a href="https://bioportal.bioontology.org/ontologies">BioPortal</a> lists some of these ontologies. The portal shows the sizes in terms of classes and not in triples, but it is a reasonable assumption that, on the average, each class is represented with at least 5-6 RDF triples. This means that, for example, the <a href="https://bioportal.bioontology.org/ontologies/SNOMEDCT">SNOMED ontology</a> represents more than 2 million triples…
</p>

The RDF canonicalization step, and the hash calculation that follows, are the sources of significant complexity. Canonicalization means traversing the datasets possibly several times in search of blank nodes and eventually renaming them. Hashing means the serialization of all the triples into n-quads, and then sorting and concatenating them before calculating the dataset's hash. Everything may have to be performed through database queries if the Datasets are part of a triple store. While all these steps are fine for Verifiable Credentials due to their small size, they may become a source of problems for general Datasets.

One line of future work might be to reconsider some of the algorithmic steps for specific categories of datasets. For example, if an RDF Dataset consists of "b-node disjoint" named graphs (i.e., no two graphs share any b-nodes), the calculation of a hash can be done by using [Merkle Trees](https://en.wikipedia.org/wiki/Merkle_tree), widely used in the crypto community (see some further thoughts on that in a separate [gist](https://gist.github.com/iherman/40174d81be4b08ab43444c236d2181df)). Other categorizations and adaptations might be necessary for a wider usage of [[vc-data-integrity]] for RDF Datasets.

It may also be worth looking at the [[vc-data-integrity]] algorithmic steps themselves to see if efficiency problems might be avoided. The author has noted two, relatively minor, aspects that may be worth reconsidering: the hash function usage in the [[rdf-canon]] algorithm and the way proof chains are secured.

## Hash function in canonicalization

The hash function used in [[rdf-canon]] is, by default, SHA-256. This function is used in two places: it is the hash function used for the canonicalization itself, and also to calculate the final hash. However, the [[[rdf-canon]]] specification stipulates that a conforming implementation could be used with another hash function. Note that, strictly speaking, there is no requirement, in general, to use the same hash functions for these two roles.

This feature is exploited by [[[vc-di-ecdsa]]]: when calculating the hash of the data _and_ the P-384 curve is used, the requirement is that [[rdf-canon]] must use SHA-384. This may become a problem if, for the same dataset, several proofs are calculated to form <a data-cite="vc-data-integrity#proof-sets">_proof sets_</a> or  <a data-cite="vc-data-integrity#proof-chains">_proof chains_</a> that include both ECDSA using P-256 and P-384. This indeed forces the implementation to re-canonicalize the input dataset using two different hashing functions. While using SHA-384 for the hashing of the sorted, canonicalized n-quads might be explainable, it is unclear what the algorithm gains by enforcing SHA-384 on the canonicalization step itself. At first glance, it looks like an unnecessary load on the signing process with no clear gain.

<p class=note>It may be that there are regulatory reasons that require SHA-384 in every step of a signature. In which case there may be no choice…</p>

## Proof chains

The [[vc-data-integrity]] specification introduces the notion of  <a data-cite="vc-data-integrity#proof-chains">_proof chains_</a>: the same dataset is signed by several keys, and the order of those keys are significant. Conceptually, it defines the order in which the datasets must be signed/verified (as opposed to <a data-cite="vc-data-integrity#proof-sets">_proof sets_</a> which simply consists of several, independent signatures). Cryptographically, this means that a signature should "sign over" the previous signature in the chain (if applicable). 

Technically, this means the signature by <em>key<sub>i</em> should be used to verify _both_ the original dataset _and_ the signature provided by <em>key<sub>i-1</em>. In [[vc-data-integrity]] this is achieved by merging the original dataset <em>D</em> and the proof graph <em>PG<sub>i-1</sub></em> created for <em>key<sub>i-1</em>. In practice this means:

1. Refactor blank nodes in <em>PG<sub>i-1</sub></em> to avoid clashes with blank nodes in <em>D</em>;
2. Perform the canonicalization and the hash of <em>D + PG<sub>i-1</sub></em>;
3. Sign the hash value.

All steps, mostly the first two, are costly if <em>D</em> is large. For RDF Datasets in general it would be way better to do something like:

1. Perform the canonicalization and the hash of <em>PG<sub>i-1</sub></em>, resulting in <em>h<sub>i-1</sub></em>
2. Concatenate the hash of <em>D</em> (which has to be calculated only once) with <em>h<sub>i-1</sub></em>
3. Sign the concatenated hash value

This would achieve the same "signing over" requirement with, possibly, fraction of the costs (proof graphs are small, i.e., their canonicalization has practically no cost).



