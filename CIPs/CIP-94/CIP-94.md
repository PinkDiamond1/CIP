---
cip: 94
title: NFT DID Method Specification
author: Joel Thorstensson (@oed)
discussions-to: https://github.com/ceramicnetwork/CIP/issues/XX
status: Draft
category: Standards
type: RFC
created: 2021-02-12
---

# NFT DID Method Specification

The NFT DID Method converts any non-fungible token on any blockchain into a decentralized identifier where the owner of the NFT is the controller of the DID. This is achieved by using the *Chain Agnostic Improvement Proposals* to describe NFT assets and blockchain accounts, as well as the Ceramic network to find the DID associated with the owner of the NFT.

## DID Method Name

The name string that shall identify this DID method is: `nft`.

A DID that uses this method MUST begin with the following prefix: `did:nft`. Per the [DID specification](https://w3c.github.io/did-core/), this string MUST be in lowercase. The remainder of the DID, after the prefix, is specified below.

## Method Specific Identifier

The method specific identifier is simply a [CAIP-19 Asset ID](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-19.md) where all `:` characters have been replaced by `.` and all `/` characters replaced by `_`, in order to comply with the DID spec.

### Example

In the example below we see a [cryptokitty](https://opensea.io/assets/0x06012c8cf97bead5deae237070f9587f8e7a266d/771769) being used as a DID. Cryptokitties is an ERC721 token on Ethereum and we can refer to it using the ERC721 asset namespace as defined in [CAIP-22](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-22.md).

```sh
did:nft:eip155.1_erc721.0x06012c8cf97BEaD5deAe237070F9587f8E7A266d_771769
```

## CRUD Operation Definitions

In this section the CRUD operations for an NFT DID are defined.

### Create

Mint an NFT on any blockchain.

### Read/Verify

Extract the Asset ID from the method specific identifier. From it you can learn which asset on which blockchain to look up the owner of the NFT. Once you have the owner account convert it to a [CAIP-10 Account ID](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-10.md) by combining it with the Chain ID from the Asset ID. Using the Ceramic `caip10-link` doctype we can now look up the DID which this account points to.

Here's and example of how to make this lookup using the javascript Ceramic api. This function will return either `null` or the DID related to the given account id.

```js
async function getControllerDid(accountId: string) {
  const caip10Doc = await ceramic.createDocument('caip10-link', {
    metadata: {
      family: 'caip10-link',
      controllers: [accountId]
    }
  })
  return caip10Doc?.content
}
```

Once we have retrieved the DID from the `caip10-link` we simply construct the DID document as follows (here `controller-did` is the DID returned by the function above).

```json
{
  "@context": "https://w3id.org/did/v1",
  "id": "<did>",
  "controller": "<controller-did>"
}
```

If the `caip10-link` returned `null` we simply omit the `controller` property.

##### DID Document Metadata

When resolving the DID document [DID Document Metadata](https://w3c.github.io/did-core/#did-document-metadata) should be provided. When resolving an NFT we should populate the following fields:

* `create` - populate using the blockchain timestamp from the block when the NFT was created
* `updated` - populate using the blockchain timestamp from the block of the most recent owner change
* `versionTime` - populate using the blockchain timestamp from the block of the most recent owner change
* `versionId` - integer, should equal the number of owners starting at 0

Note that the regular ERC721 api doesn't provide any way to query this data. An implementer is recommended to use [The Graph protocol](https://thegraph.com/) to create *subgraphs* for NFTs that are supported. Same thing should apply to NFT systems on other blockchains as well.

#### Resolving using the `versionId`/`versionTime` parameter

When the `versionId`/`versionTime` query parameter is given as a DID is resolved it means that we should try to resolve a specific version of the DID document. The resolution process requires the use of the *subgraph* explained in the paragraph above. From it retrieve the owner of the NFT at the given version. Once the owner is known the `caip10-link` can be looked up. Once this document is loaded go though the commits (use *AnchorCommits*) to find the state of the document at the time of the resolved version of the NFT DID, to retrieve the controller DID.

##### DID Document Metadata

* `create` - populate using the blockchain timestamp from the block when the NFT was created
* `updated` - populate using the blockchain timestamp from the block when the given versions owner became owner
* `versionTime` - populate using the blockchain timestamp from the block when the given versions owner became owner
* `versionId` - integer, should equal the number of owners until the looked up version starting at 0
* `nextUpdate` - populate using the blockchain timestamp from the block when the next owner became owner
* `nextVersionId` - integer, `versionId` + 1

### Update

Transfer the NFT to another account.

### Deactivate

Transfer the NFT to an account that no one controls, i.e. burn it.

## Security Requirements

The NFT DID derives most of its security properties from the blockchain which the given NFT lives on and the Ceramic protocol. The security of different blockchains may vary so implementers may choose to limit their implementations to specific NFT assets. Ceramic most notably provides *censorship resistance*, *decentralization*, and requiring a minimal amount of data to be synced to completely verify the integrity of the `caip10-link`s used to find the controller DID. For more details see the Ceramic [specification](https://github.com/ceramicnetwork/ceramic/blob/master/SPECIFICATION.md).

## Privacy Requirements

See [§ 10. Privacy Considerations](https://www.w3.org/TR/did-core/#privacy-considerations) in `did-core`.

## Extensibility

The NFT DID Method could currently supports ERC721 tokens, but could be easily extended to support any other NFT token givent that the token is registered as a [CAIP asset namespace](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-19.md).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).