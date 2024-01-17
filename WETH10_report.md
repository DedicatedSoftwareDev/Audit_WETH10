# QA Report 

## 1. DepositTo function

### Impact 

The deposits amounts will be locked in this WETH contact forever.

### POC

In depositTo function, there is no check for address(this), address(to).
It means that if user attacked by phishing, deposit amount will be lost forever.

```
function depositTo(address to) external override payable { //@audit no check address this
    // _mintTo(to, msg.value);
    balanceOf[to] += msg.value;
    emit Transfer(address(0), to, msg.value);
}


function depositToAndCall(address to, bytes calldata data) external override payable returns (bool success) {//@audit reentrancy, no check
    // _mintTo(to, msg.value);
    balanceOf[to] += msg.value;
    emit Transfer(address(0), to, msg.value);

    return ITransferReceiver(to).onTokenTransfer(msg.sender, msg.value, data);
}
```

### Recommended Mitigation Steps

```
require(address(to) == 0);
require(address(to) == address(this));
```
This part should be added before balanceOf[to] += msg.value;

## 2. solidity version

### Impact

overflow/underflow potentially passible in this contract.

### POC

```
pragma solidity 0.7.6;
```
This contract uses solidity version 0.7.6
As you know, overflow/underflow check is implemented since solidity version 0.8.
This contrat doesnt use well know security smart contact such as oppenzeplin library(safeERC20).
So potntialy possible.

### Recommended Mitigation Steps

Replace the code to this.
```
pragma solidity >= 0.8.0;
```
## 3. Reentrany Attack is potentially may be possible.

### Impact

In this function old flashMinted(the value before calling this function) should not be changed after call.
But may be possible to modify it by reentrancy attack.


### POC

I haven't check the flashLoan function fully.

```
flashMinted = flashMinted + value;//@audit overflow if flashminted is not 0

require(flashMinted <= type(uint112).max, "WETH: total loan limit exceeded");

// _mintTo(address(receiver), value);
balanceOf[address(receiver)] += value;//@audit can be possble overflow
```


```
require(
    receiver.onFlashLoan(msg.sender, address(this), value, 0, data) == CALLBACK_SUCCESS,
    "WETH: flash loan failed"
);
```


```
balanceOf[address(receiver)] = balance - value;
emit Transfer(address(receiver), address(0), value);

flashMinted = flashMinted - value;
```

receiver.onFlashLoan is called after increase balance, and flashMinted value.
And then after receiver.onFlashLoan function is called, balance and flashMinted value is decreased.
If receiver is a malicious contract, after increasing balance, flash loan function will be called.
So it may be possible to control balance and flashMinted value.

I thought overflow issue in this funtion.
But it is impossible.
Because, value should be less than uint112, but flashMinted is uint256 type.

```
require(value <= type(uint112).max, "WETH: individual loan limit exceeded");// @audit if value max -1, what things will be happened?

flashMinted = flashMinted + value;//@audit overflow if flashminted is not 0
```
Uptill now, if flashMinted can be over flowed by several reentrancy call.
But ```require(flashMinted <= type(uint112).max, "WETH: total loan limit exceeded");``` this require is added after that.
So overflow cannot be possible.

### Recommended Mitigation Steps

It would be better to use reentrancyGuard contract in openzepplin library.

```
function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 value, bytes calldata data) nonReentrant external override returns (bool)
```

# Auditor's View
This contract is more security that I expected.
In permit function, the contract handles replay attack for another chain.
And all funtions are very clear.
In several functions, I saw the preventing techs for a reentracny.
Totally, I am happy to read this contract.
