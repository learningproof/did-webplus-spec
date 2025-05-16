`did:webplus` Method Specification
==================

**Specification Status:** Draft V0.1

**Latest Draft:**
  [https://identity.foundation/spec-up](https://identity.foundation/spec-up)

Authors:
- Victor Dods ([LedgerDomain](https://ledgerdomain.com/))
- Alex Colgan ([LedgerDomain](https://ledgerdomain.com/))

Editors:
~ [Juan Caballero](https://github.com/bumblefudge/)

Participate:
~ [GitHub repo](https://github.com/learningproof/did-webplus-spec)
~ [File a bug](https://github.com/learningproof/did-webplus-spec/issues)
~ [Commit history](https://github.com/learningproof/did-webplus-spec/commits/master)

------------------------------------

## Abstract

`did:webplus` extends `did:web` in ways that provide:

* Verifiable history for each DID
  * self-contained JWT-based cryptography library for generating and verifying these chains of DID documents
  * precise, reliable, and scalable historical resolution for audit-trail use-cases, whether over the web to the hosting server or from trusted caches
* Various additional levels of verifiability beyond did:web's guarantees, which can be achived via multiple different architecture (trusted resolver, witness network, etc)
* Cost-efficient, scalable "VDR Server" (multi-user, backwards-compatible `did:web` server)
  * Reference implementation available in Rust
* Full backwards-compatibility and fallback to `did:web` functionality (simply replace `webplus` with `web`!)

## Overview

The `did:web` method makes straightforward use of familiar tools across a wide range of use cases. However, heavily regulated ecosystems such as the pharmaceutical supply chain demand additional guarantees of immutability and auditability, including seamless key rotation and a key usage history. `did:webplus` is a proposed fit-for-purpose DID method for use within the pharma supply chain credentialing community, with an eye towards releasing it into the wild for those communities that are similarly situated.

### Terminology

Note that some terms and acronyms from the DID core specification are extended here to mean slightly different things in this context.

[[def: DID Controller, controller]]

~ (Controller): A/the DID controller is an entity capable of making valid changes to a DID document, represented by a public key; by default, a DID's only controller is its subject, but update capabilities can be delegated to devices, "restore keys", etc. Concretely, these keys are listed in the `capabilityInvocation` verification relationship in the DID document. For a DID update to be valid, the new DID document must be validly self-signed, and the `selfSignatureVerifier` field must correspond to one of the keys listed in the latest existing DID document's `capabilityInvocation` field.

[[def: DID Resolver, Resolver]]

~ A DID resolver is the actor that translates a DID URL into a DID document through a "resolution" process. As with `did:web`, this involves translating the DID into a specific URL (or set of URLs) that can be used to fetch the current or any historical DID document(s). Unlike `did:web`, however, resolution is assumed to be a little more active, in that resolution without a cache or trust requires recursive resolution through all backlinks to verify the method-specific identifier against the initial DID Document.

[[def: Verifiable Data Gateway, VDG]]

~ (VDG): A gateway works as a DID resolver "plus", in that it can not only return a current DID document and resolution data but also historical DID documents and additional metadata. The concept could be extended to translate between and abstract over additional DID methods, wherever equivalent historical verifiability, metadata, and semantics are achievable. For advanced capabilities such as Content Delivery Network (CDN)-style distributing caching, see the [Architectural Variations](#architectural-variations) appendix.

[[def: Verifiable Data Registrar, VDR]]

~ (VDR): While the original `did:web` specification essentially treats each HTTP server as its own "verifiable data registry" (as defined in the DID core specification), `did:webplus` refers to the hosting server as a "verifiable data **registrar**", facilitating updates and deletions cryptographically rather than simply being a publication medium.

[[def: Verifying Party, Verifier]]

~ (Verifier): A Verifying Party can verify a digitally signed artifact by resolving the DID of the signer to obtain and use the appropriate public key for signature verification.
A Verifying Party will use a [[ref: DID Resolver]] to resolve the DID. They may use a Full DID Resolver or a Thin DID Resolver, depending on their needs (see [Architectural Variations](#architectural-variations) appendix for details).

## Specification

### DID Scheme

The scheme is `webplus`; note that ANY valid `did:webplus` URL (without query parameters or matrix parameters) is a valid `did:web` URL, which can be resolved as such simply by removing the `plus`.

### Method-Specific Identifiers

The persistent identifier for each DID is the "self-hash" (see [Create](#create) section below) of the initial DID Document.
This identifier remains unchanged across all updates and deactivation.
Resolving the DID with no DID path will always return the current DID document, as will
`did:webplus:{permanent identifier}:{current versionId}:{self-hash of current DID document}`

### Resolution Mechanics and DID Paths

Resolving a DID without query parameters should return that DID's latest DID document. In order to determine the latest DID document, the resolver MUST query the VDR whose domain hosting the DID's microledger, which is the origin for that DID's microledger, and therefore the authority on which DID document is the latest.

#### DID Path to URL mapping

The DID-to-resolution-URL translation rules for the latest DID document are the same as those for did:web, for example:

```bash
did:webplus:example.com:EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ -> https://example.com/EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ/did.json
did:webplus:example.com:path-component:EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ -> https://example.com/path-component/EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ/did.json
did:webplus:example.com%3A3000:EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ -> https://example.com:3000/EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ/did.json
did:webplus:example.com%3A3000:path-component:EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ -> https://example.com:3000/path-component/EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ/did.json

did:webplus:localhost:EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ -> http://localhost/EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ/did.json
did:webplus:localhost:path-component:EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ -> http://localhost/path-component/EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ/did.json
did:webplus:localhost%3A3000:EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ -> http://localhost:3000/EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ/did.json
did:webplus:localhost%3A3000:path-component:EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ -> http://localhost:3000/path-component/EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ/did.json
```

##### TODO - partial version-specific paths possible?

can you request

`did:webplus:example.com:EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ:2`

or

`did:webplus:example.com:EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ:EgqvDOcj4HItWDVij-yHj0GtBPnEofatHT2xuoVD7tMY`

or ONLY 

`did:webplus:example.com:EjXivDidxAi2kETdFw1o36-jZUkYkxg0ayMhSBjODAgQ:2:EgqvDOcj4HItWDVij-yHj0GtBPnEofatHT2xuoVD7tMY`

?  If there are any rules for the VDR hoster (like, should the server 303 or 307 or otherwise forward from 

`/2/` to 

`/2/EgqvDOcj4HItWDVij-yHj0GtBPnEofatHT2xuoVD7tMY` ?)

#### TODO - Query Parameters

??? not sure how this works with static servers like gitpages, unless you optionally make dynamic servers translate `version=` or `versionId=` into the above?

### DID Fragments

### DID Operations

#### Create

#### Read

##### Levels of Verification and Supplemental Trust

#### Update

#### Deactivate

### DID Authorization

At minimum, each DID document contains at least one public key at inception, and updates to it can be authorized only by the matching private key.
Additional controllers (in the form of additional public keys) can be added at inception or by any valid update; see the update section above.

### Example 



## Appendices

### Weaknesses of DID:WEB

The ways in which a malicious HTTP server can falsify or distort DID documents in `did:web` are inherited by the basic configuration of `did:webplus`, but the latter has mitigations which can be provided as needed for different use-cases (see [Architectural Variations](#architectural-variations) below).

#### DID Forks

#### Deletion 

### Architectural Variations

### Security Considerations

### Privacy Considerations

#### Recommendations for Verifiable Credential usage

#### Full DID Resolver

#### Thin DID Resolver + VDG