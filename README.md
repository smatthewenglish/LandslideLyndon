# Optimized Computational Cost Profile 🏗⛽


The process of refactorization entails two essential prerequisites, the desired outcome and an established baseline. In this case the desired outcome is the optimization of the computational resources consumed in NFT instantiation/allocation, resulting in reduced gas usage. The metric against which the proposed changes described in this document are assessed is a modified version of  `NiftyBuilderMaster`, deployed to Rinkeby Testnet [here](https://rinkeby.etherscan.io/address/0xab6c1f49989020c31942d12014dcab14b81f99ff#code), and a modified version of `NiftyBuilderInstance` also on Rinkeby, found [here](https://rinkeby.etherscan.io/address/0x64997Ad14666d8e2abc8891602Ac76E4072A065F#code). The refactored smart contracts can be found in [this](https://github.com/smatthewenglish/LandslideLyndon) GitHub repository.

## Storage

The optimizations described in the following section realize performance improvements by enhancing the way information is allocated. 

### _IPFSHashHasBeenSet
The `_IPFSHashHasBeenSet` mapping indicates whether a `niftyType` has been assigned an IPFS hash. 
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
By implementing this change we can safely remove the `_IPFSHashHasBeenSet` mapping, eliminating the cost associated with storage of that data. 

As an aside, additional gas savings can be realized here and elsewhere by removing the message string associated with the `require()` statement. 

### _niftyIPFSHashes

The function `_setTokenIPFSHash` from `ERC721Metadata` is biased to `1/1` tokens.
```javascript
_setTokenIPFSHash(tokenId, ipfsHash);
 ```
For nifties such as *The OG* (n/50) we end up with the following: 
```json
{"6200050001":"QmdKektfKmAEw7f692D7G5yQbacQomeaRmgMDnpqG9bcJv",
 "6200050002":"QmdKektfKmAEw7f692D7G5yQbacQomeaRmgMDnpqG9bcJv",
 ...
 "6200050050":"QmdKektfKmAEw7f692D7G5yQbacQomeaRmgMDnpqG9bcJv"}
```
Since the association of an edition with an IPFS hash is likewise stored in `_niftyIPFSHashes[niftyType]` we could, without loss of generality, set this `ERC721Metadata` property with the `niftyType` exclusively, or without reference to the `specificTokenId` like so:

```json
{"62000500":"QmdKektfKmAEw7f692D7G5yQbacQomeaRmgMDnpqG9bcJv"}
```

As the metadata extension is considered optional for `ERC721` compliance, if deemed desirable, the call to `_setTokenIPFSHash(tokenId, ipfsHash)` might be eliminated entirely. 

**Note**: Notwithstanding the consolidated mapping from type to IPFS hash, assigning individualized token identifiers has emergent utility, as observed by the fact that *The OG* [#8/50](https://niftygateway.com/itemdetail/secondary/0xf924fed62a15c879213e677dada6cf7db5174620/6200050008) is listed on the marketplace for $888.00 (*a substantial markup from others of the same edition*), likely in reference to the tokenID of `6200050008`. This is a clear differentiator from the "Multi Token Standard" ERC-1155. 

## Execution

This section details changes that simplify or remove the computational costs entailed in the modification of data. 

### Bitwise Operation × Assembly 

The mathematical operations performed in `encodeTokenId()` of `NiftyBuilderMaster` can be performed using EVM Opcodes directly. For instance, instead of using the Opcode associated with multiplication we could perform multiplication by bitwise manipulation, using the less expensive [shift operators](https://eips.ethereum.org/EIPS/eip-145). 

```
function encodeTokenId(uint contractIdCalc, uint niftyType, uint specificNiftyNum) public view returns (uint) {
    assembly {
        let result := add((contractIdCalc * topLevelMultiplier), (niftyType * midLevelMultiplier))
        mstore(0x0, result)
        return(0x0, 32) 
    }
    return result + specificNiftyNum;
}
```
**Note**: The above code doesn't yet implement bit string manipulation, where the greatest savings of computational resources would be realized. More testing is necessary to determine how the `assembly` block above differs from compiled solidity code using higher level facets of the language. 

### Batch minting

Using a `for` loop to batch mint nifties exposes us to `Out of Gas` and `Timeout` exceptions. 
One way to mitigate this would be to employ a function that takes the number of iterations as a parameter to be adjusted to a targeted value ensured, based on prior empirical validation, to complete successfully. 

## Design

This section comprises possible simplifications of the design patterns used with an eye toward efficiency and comprehensibility. 

### tokenID 
The prompt defines the `tokenId` as comprised by the concatenation of three values:
```
{contract_id}{nifty_type}{edition #}
```
If we defined them as a number where significant digits corresponded to critical information we'd be able to access important information about each token using operations less resource intensive than string manipulation, e.g. rounding a number to the nearest significant digit to remove the `edition #` as opposed to truncating the last digits of a string. 


### Data Model

The prompt specifies that the nifty type "*gives no information about the edition size*". If the nifty type did correspond to the number of nifties associated with an edition it would simplify the number of parameters required to be passed to corresponding methods, specifically methods we might design related to *batch minting*. 


### isNiftySoldOut × niftyType

As the nifty type currently doesn't provide information about the edition size the following check can be removed: 
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


## Discussion
The changes proposed here represent an initial step toward a smart contract architecture with an optimized computational cost profile, before live deployment to main-net the assumptions they're based on need to be subjected to rigorous challenge. 










