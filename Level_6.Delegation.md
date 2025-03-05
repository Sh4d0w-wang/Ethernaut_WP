## Level_6.Delegation

要求：

> 将合约所有权归于自己；

合约：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {
    address public owner;
    constructor(address _owner) {
        owner = _owner;
    }
    // 使owner变为msg.sender
    function pwn() public {
        owner = msg.sender;
    }
}

contract Delegation {
    address public owner;
    Delegate delegate;

    // 设置代理地址和owner
    constructor(address _delegateAddress) {
        delegate = Delegate(_delegateAddress);
        owner = msg.sender;
    }
    
    // 调用不存在函数时会执行
    // 此处不能接受ETH，因为没有payable关键字
    fallback() external {
        // 代理调用，参数是msg.data
        (bool result,) = address(delegate).delegatecall(msg.data);
        if (result) {
            this;
        }
    }
}
```

### 分析

两个合约中除构造函数外修改` owner `的就一个函数：` pwn() `，但目前不清楚哪会调用它；

往下看发现` Delegation `会设置代理合约，然后在` fallback `中执行` msg.data `；

首先来回顾一下` call `和` delegatecall `的区别，例子：` 钱包A 调用 合约B，合约B call/delegatecall 合约C的函数 `；

```solidity
// call
A ----------call----------> B ----------call----------> C
                     msg.sender = A              msg.sender = B
                     msg.data = A给的             msg.data = B给的
                     变量的变化 = 仅C中的变量变化    变量的变化 = 仅C中的变量变化

// delegatecall
A ----------call----------> B ----------delegatecall----------> C
                     msg.sender = A                  ****msg.sender = A****
                     msg.data = A给的                     msg.data = A给的
                     变量的变化 = 仅C中的变量变化            变量的变化 = 仅B中的变量变化
```

可以一句话总结，` delegatecall `就是` 用户A `用` 他的身份和内容 `，借用` 合约C中的函数 `更改` 合约B中的内容 `；

可以看到，给我们生成的示例是` Delegation `合约：

![image-20250103000610388](./assets/image-20250103000610388.png)

也就是希望我们用自己的钱包通过代理合约调用` pwn `，这样` owner `就会被设置成我们的；

当调用不存在的函数时，默认会调用` fallback `函数，只需在` msg.data `中填入` pwn `的函数选择器即可；



### 攻击

函数` pwn() `的函数选择器是：` 0xdd365b8b `；

![image-20250103002218467](./assets/image-20250103002218467.png)

