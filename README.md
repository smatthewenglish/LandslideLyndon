# Gas Usage Reduction 🏗⛽

With the smart contract deployed [here](https://etherscan.io/address/0xf924fed62a15c879213e677dada6cf7db5174620#code) as a baseline, this document describes a number of modifications, implemented by the smart contract in [this](https://github.com/smatthewenglish/LandslideLyndon) GitHub repository.

## Storage

The optimizations described in the following section realize performance improvements by enhancing the way that data is allocated. 

### _IPFSHashHasBeenSet
The `_IPFSHashHasBeenSet` mapping indicates whether a `niftyType` has been allocated an IPFS hash. 
```javascript
mapping (uint => bool) public _IPFSHashHasBeenSet;`
```
The mapping can be made redundant by setting `_niftyIPFSHashes[niftyType]` to a default value which indicates that it hasn't yet been assigned, e.g. `0000000000000000000000000000000000000000000000`. Resulting in the following: 

```javascript
function setNiftyIPFSHash(uint niftyType, string memory ipfs_hash) onlyValidSender public {    
    require(_niftyIPFSHashes[niftyType] != "0000000000000000000000000000000000000000000000", "Can only be set once.");
    _niftyIPFSHashes[niftyType] = ipfs_hash;
}
```
Allowing us to safely remove the `_IPFSHashHasBeenSet` mapping. As an aside, additional gas savings can be achieved by removing the error message from the `require()` statement. 

### _niftyIPFSHashes

The function `_setTokenIPFSHash` from `ERC721Metadata` is biased to `1/1` tokens.
```javascript
_setTokenIPFSHash(tokenId, ipfsHash);
 ```
For nifties such as *The OG* (n/50) we end up with the following: 
```json
{"6200050001":"QmdKektfKmAEw7f692D7G5yQbacQomeaRmgMDnpqG9bcJv",
 "6200050002":"QmdKektfKmAEw7f692D7G5yQbacQomeaRmgMDnpqG9bcJv",
 "6200050050":"QmdKektfKmAEw7f692D7G5yQbacQomeaRmgMDnpqG9bcJv"}
```
Since the association of an edition with an IPFS hash is likewise stored in `_niftyIPFSHashes[niftyType]` we could, without loss of generality, set this `ERC721Metadata` property with the `niftyType` exclusively, or without reference to the `specificTokenId` like so:

```json
{"62000500":"QmdKektfKmAEw7f692D7G5yQbacQomeaRmgMDnpqG9bcJv"}
```

As the metadata extension is considered optional for `ERC721` compliance the call to `_setTokenIPFSHash(tokenId, ipfsHash)` could be removed entirely. 










## Execution

### Assembly 

Requires more testing, could be optimized away by the compiler but...

### Batch minting

Possibility of out fo gas errors...

## Design

This section comprises possible simplifications of the design patterns used with an eye toward efficiency and comprehensibility. 

### isNiftySoldOut × niftyType

The prompt mentions that a nifty type "*gives no information about the edition size*", which mean the following check could be removed: 
```javascript
function isNiftySoldOut(uint niftyType) public view returns (bool) {
    if (niftyType > numNiftiesCurrentlyInContract) {...}
}
```
The new `isNiftySoldOut()` method body would look like so: 
```javascript
if (_numNiftyMinted[niftyType].current() > _numNiftyPermitted[niftyType]) {
    return (true);
}
return (false);
```


### Inheritance

There are two instances of the `niftyRegistryContract` address, once in `BuilderShop` and once in `NiftyBuilderInstance`.
```javascript
contract NiftyCore {
   address public addressRegistry = 0x6e53130dDfF21E3BC963Ee902005223b9A202106;
}
```
The value could be abstracted into a super class, providing for more efficient and less resource intensive architecture.  














