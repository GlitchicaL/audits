# First Flight #12: Kitty Connect - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Anyone can call `KittyBridge::bridgeNftWithData()` to mint a NFT on another chain](#H-01)
    - ### [H-02. `KittyConnect::mintBridgedNFT()` incorrectly tracks `KittyConnect::s_ownerToCatsTokenId` for bridged NFTs](#H-02)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #12: Kitty Connect

### Dates: Mar 28th, 2024 - Apr 4th, 2024

[See more contest details here](https://www.codehawks.com/contests/clu7ddcsa000fcc387vjv6rpt)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 0
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Anyone can call `KittyBridge::bridgeNftWithData()` to mint a NFT on another chain            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/main/src/KittyBridge.sol#L53-L77

## Summary

Anyone can call `KittyBridge::bridgeNftWithData()` allowing them to mint a NFT on a different chain without actually owning a NFT provided by a shop partner.

## Vulnerability Details

The `KittyBridge::bridgeNftWithData()` function is responsible for building and sending the CCIP message: 

```javascript
    function bridgeNftWithData(uint64 _destinationChainSelector, address _receiver, bytes memory _data)
        external
        onlyAllowlistedDestinationChain(_destinationChainSelector)
        validateReceiver(_receiver)
        returns (bytes32 messageId)
    {
        // Create an EVM2AnyMessage struct in memory with necessary information for sending a cross-chain message
        Client.EVM2AnyMessage memory evm2AnyMessage = _buildCCIPMessage(_receiver, _data, address(s_linkToken));

        // Initialize a router client instance to interact with cross-chain router
        IRouterClient router = IRouterClient(this.getRouter());

        // Get the fee required to send the CCIP message
        uint256 fees = router.getFee(_destinationChainSelector, evm2AnyMessage);

        if (fees > s_linkToken.balanceOf(address(this))) {
            revert KittyBridge__NotEnoughBalance(s_linkToken.balanceOf(address(this)), fees);
        }

        messageId = router.ccipSend(_destinationChainSelector, evm2AnyMessage);

        emit MessageSent(messageId, _destinationChainSelector, _receiver, _data, address(s_linkToken), fees);

        return messageId;
    }
```

The ideal scenario would be that this function gets called from the `KittyConnect::bridgeNftToAnotherChain()` function since the function verifies the owner of the NFT, and burns the NFT, before finally calling the `KittyBridge::bridgeNftWithData()` with the appropriate data of the cat NFT.

However since `KittyBridge::bridgeNftWithData()` is external and only checks for onlyAllowlistedDestinationChain, and validateReceiver, any contract or account can call this function passing in any arbitrary data to represent a cat NFT that isn't provided by a shop partner.

## Impact

- Allows a user to mint their own version of a cat NFT on a different chain not provided by an official shop partner

## Tools Used

VS Code, Foundry

## Recommendations

Create a `onlyKittyConnect` modifier and add it to the `KittyBridge::bridgeNftWithData()` function:

```diff
+   modifier onlyKittyConnect() {
+       require(
+           msg.sender == kittyConnect,
+           "KittyConnect__NotKittyConnect"
+       );
+       _;
+   }

    function bridgeNftWithData(uint64 _destinationChainSelector, address _receiver, bytes memory _data)
        external
+       onlyKittyConnect()
        onlyAllowlistedDestinationChain(_destinationChainSelector)
        validateReceiver(_receiver)
        returns (bytes32 messageId)
    {
        // Create an EVM2AnyMessage struct in memory with necessary information for sending a cross-chain message
        Client.EVM2AnyMessage memory evm2AnyMessage = _buildCCIPMessage(_receiver, _data, address(s_linkToken));

        // Initialize a router client instance to interact with cross-chain router
        IRouterClient router = IRouterClient(this.getRouter());

        // Get the fee required to send the CCIP message
        uint256 fees = router.getFee(_destinationChainSelector, evm2AnyMessage);

        if (fees > s_linkToken.balanceOf(address(this))) {
            revert KittyBridge__NotEnoughBalance(s_linkToken.balanceOf(address(this)), fees);
        }

        messageId = router.ccipSend(_destinationChainSelector, evm2AnyMessage);

        emit MessageSent(messageId, _destinationChainSelector, _receiver, _data, address(s_linkToken), fees);

        return messageId;
    }
```
## <a id='H-02'></a>H-02. `KittyConnect::mintBridgedNFT()` incorrectly tracks `KittyConnect::s_ownerToCatsTokenId` for bridged NFTs            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-kitty-connect/blob/main/src/KittyConnect.sol#L174

## Summary

The `KittyConnect::mintBridgedNFT()` function fails to update `KittyConnect::s_ownerToCatsTokenId` mapping leading to an incorrect tracking of owned NFT tokenIds for a user.

## Vulnerability Details

The `KittyConnect::mintBridgedNFT()` function is called by the KittyBridge contract and is used to mint a NFT for when a user decides to bridge their NFT. First the function decodes the `data`, gets and updates the `kittyTokenCounter`, sets the `s_catInfo`, emits an event and finally mints the NFT as you can see here:

```javascript
    function mintBridgedNFT(bytes memory data) external onlyKittyBridge {
        (
            address catOwner, 
            string memory catName, 
            string memory breed, 
            string memory imageIpfsHash, 
            uint256 dob, 
            address shopPartner
        ) = abi.decode(data, (address, string, string, string, uint256, address));

        uint256 tokenId = kittyTokenCounter;
        kittyTokenCounter++;

        s_catInfo[tokenId] = CatInfo({
            catName: catName,
            breed: breed,
            image: imageIpfsHash,
            dob: dob,
            prevOwner: new address[](0),
            shopPartner: shopPartner,
            idx: s_ownerToCatsTokenId[catOwner].length
        });

        emit NFTBridged(block.chainid, tokenId);
        _safeMint(catOwner, tokenId);
    }
```

The issue currently lies around setting the CatInfo `idx`. The `idx` is set to be `s_ownerToCatsTokenId[catOwner].length` which isn't an issue of itself, but `s_ownerToCatsTokenId` is never updated here. Should a user bridge multiple NFTs over, the `idx` of those NFTs will be the same. Should a shop mint a new NFT via `KittyConnect::mintCatToNewOwner` with an existing bridged NFT, it will also share the same `idx` as the bridged NFT.

It's worth noting, since a bridged NFT isn't updated, the mapping `s_ownerToCatsTokenId[catOwner]` won't track bridged NFTs for a user.

## Impact

- Allows minted and bridged NFTs to share the same `idx`.
- Results in `KittyConnect::getCatInfo()` to return incorrect data.
- Results in `KittyConnect::getCatsTokenIdOwnedBy()` to not return the correct set of tokenIds owned by a address.

## POC

```javascript
    function test_incorrectTrackingForBridgedNFT() public {
        address owner = makeAddr("owner");
        uint256 tokenId = kittyConnect.getTokenCounter(); // Should be 0
        bytes memory data = abi.encode(
            owner,
            "meowdy",
            "ragdoll",
            "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62",
            block.timestamp,
            partnerA
        );

        // Pretend to be the bridge and mint for the owner
        vm.prank(address(kittyBridge));
        kittyConnect.mintBridgedNFT(data);

        // Let's get info our new bridged NFT
        KittyConnect.CatInfo memory catInfo = kittyConnect.getCatInfo(tokenId);
        uint256[] memory userCatTokenId = kittyConnect.getCatsTokenIdOwnedBy(
            owner
        );

        assert(userCatTokenId.length == 0); // Should be equal to 1, but KittyConnect hasn't pushed the tokenId...
        assert(catInfo.idx == 0); // The idx will be 0, which is technically okay since it's the first one...

        // However, now let's mint a NFT...
        vm.prank(partnerA);
        kittyConnect.mintCatToNewOwner(
            owner,
            "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62",
            "Hehe",
            "Hehe",
            block.timestamp
        );

        // Let's get info on our new minted NFT
        KittyConnect.CatInfo memory newCatInfo = kittyConnect.getCatInfo(
            tokenId + 1
        );
        uint256[] memory newUserCatTokenId = kittyConnect.getCatsTokenIdOwnedBy(
            owner
        );

        assert(newUserCatTokenId.length == 1); // KittyConnect has now pushed this tokenId (should really be 2)...
        assert(newCatInfo.idx == 0); // BUT the idx is still 0!
    }
```

## Tools Used

VS Code, Foundry

## Recommendations

Update `s_ownerToCatsTokenId` and push the `tokenId`:

```diff
    function mintBridgedNFT(bytes memory data) external onlyKittyBridge {
        (
            address catOwner, 
            string memory catName, 
            string memory breed, 
            string memory imageIpfsHash, 
            uint256 dob, 
            address shopPartner
        ) = abi.decode(data, (address, string, string, string, uint256, address));

        uint256 tokenId = kittyTokenCounter;
        kittyTokenCounter++;

        s_catInfo[tokenId] = CatInfo({
            catName: catName,
            breed: breed,
            image: imageIpfsHash,
            dob: dob,
            prevOwner: new address[](0),
            shopPartner: shopPartner,
            idx: s_ownerToCatsTokenId[catOwner].length
        });

+       s_ownerToCatsTokenId[catOwner].push(tokenId);

        emit NFTBridged(block.chainid, tokenId);
        _safeMint(catOwner, tokenId);
    }
```
		





