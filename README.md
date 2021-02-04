# NG Gas Usage Reduction 🏗⛽


The process of refactorization entails two essential prerequisites, the desired outcome and an established baseline. In this case the desired outcome is the reduction of gas usage in NFT instantiation/allocation. The metric against which the proposed changes described in this document are assessed is a modified version of  `NiftyBuilderMaster`, deployed to Rinkeby Testnet [here](https://rinkeby.etherscan.io/address/0xab6c1f49989020c31942d12014dcab14b81f99ff#code), and a modified version of `NiftyBuilderInstance` also on Rinkeby, found [here](https://rinkeby.etherscan.io/address/0x64997Ad14666d8e2abc8891602Ac76E4072A065F#code). The refactored smart contracts can be found in [this](https://github.com/smatthewenglish/LandslideLyndon) GitHub repository.

## Storage

The optimizations described in the following section realize performance improvements by enhancing the way that data is allocated. 

### _IPFSHashHasBeenSet
The `_IPFSHashHasBeenSet` mapping indicates whether a `niftyType` has been allocated an IPFS hash. 
```javascript
mapping (uint => bool) public _IPFSHashHasBeenSet;
```
The mapping can be made redundant by evaluating whether or not `_niftyIPFSHashes[niftyType]` is an empty string. Resulting in the following method for `setNiftyIPFSHash()`: 

```javascript
function setNiftyIPFSHash(uint niftyType, string memory ipfs_hash) onlyValidSender public {      
    require(bytes(_niftyIPFSHashes[niftyType]).length == 0, "Can only be set once.");
    _niftyIPFSHashes[niftyType] = ipfs_hash;
}
```
Thereby allowing us to safely remove `_IPFSHashHasBeenSet`, eliminating the cost associated with storage of that information. 

As an aside, additional gas savings can be achieved by removing the error message from the `require()` statement. 

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

As the metadata extension is considered optional for `ERC721` compliance, if it was deemed desirable the call to `_setTokenIPFSHash(tokenId, ipfsHash)` could be eliminated entirely. 

Notwithstanding the consolidated mapping from type to IPFS hash, assigning individualized token identifiers has emergent utility, as observed by the fact that *The OG* [#8/50](https://niftygateway.com/itemdetail/secondary/0xf924fed62a15c879213e677dada6cf7db5174620/6200050008) is listed on the marketplace for $888.00 (*a substantial markup from others of the same edition*), likely in reference to the tokenID of `6200050008`. This is a clear differentiator from the "Multi Token Standard" ERC-1155. 












## Execution

### Assembly 

Requires more testing, could be optimized away by the compiler but...

### Batch minting

Possibility of out fo gas errors...

the waterfall method! 10 x 100 x 1000 x 100000

figure it out determinalisticly... 

using a loop could run out of gas, could also run out of time

## Design

This section comprises possible simplifications of the design patterns used with an eye toward efficiency and comprehensibility. 

### Data Model × tokenID 
The prompt defines the `tokenId` as comprised by the concatenation of three values:
```
{contract_id}{nifty_type}{edition #}
```
If we defined them instead as `{contract_id}{nifty_type}{edition #}` and treated the tokenID as a number e.g. `100010001` we would be able to execute operations on it like rounding `((3/30)*3)*100` to create mappings based on `nifty_type`, ignoring `edition #` completely, while also saving gas since string manipulation is more expensive than mathematical operations.

This could also help us in batch minting!

Cause we could use that as the parameter... thereby obviating the need to pass the number of tokens to our contract. !!!! yes!!!!


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














