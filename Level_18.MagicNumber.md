## Level_18.MagicNumber

要求：

> 使用10字节以内的大小写一个solver contract；

合约：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {
    address public solver;

    constructor() {}

    function setSolver(address _solver) public {
        solver = _solver;
    }

    /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
    */
}
```

### 分析

需要用EVM的字节码写出一个合约，其中有一个` whatIsTheMeaningOfLife() `函数需要返回正确的32字节数字；

没咋清楚，因为看代码完全不需要写一个Slover合约，直接写个calldata调用` setSolver `就行了，而且这个` whatIsTheMeaningOfLife() `也不知道是干啥的；

反编译合约也没什么线索：

```solidity
// Decompiled by library.dedaub.com
// 2025.02.04 10:35 UTC
// Compiled using the solidity compiler version 0.8.12


// Data structures and variables inferred from the use of storage instructions
address _solver; // STORAGE[0x0] bytes 0 to 19

function fallback() public payable { 
    revert();
}

function setSolver(address newSolver_) public payable { 
    require(msg.data.length - 4 >= 32);
    _solver = newSolver_;
}

function solver() public payable { 
    return _solver;
}

// Note: The function selector is not present in the original solidity code.
// However, we display it for the sake of completeness.

function __function_selector__( function_selector) public payable { 
    MEM[64] = 128;
    require(!msg.value);
    if (msg.data.length >= 4) {
        if (0x1f879433 == function_selector >> 224) {
            setSolver(address);
        } else if (0x49a7a26d == function_selector >> 224) {
            solver();
        }
    }
    fallback();
}
```

继续查查部署该合约时的代码，地址0x2132C7bc11De7A90B87375f282d36100a29f97a9，注意到这个函数：

```solidity
function 0xd38def5b(address varg0, address varg1) public nonPayable { 
    require(msg.data.length - 4 >= 64);
    v0, v1 = varg0.solver().gas(msg.gas);
    require(bool(v0), 0, RETURNDATASIZE()); // checks call status, propagates error data on error
    require(MEM[64] + RETURNDATASIZE() - MEM[64] >= 32);
    require(v1 == address(v1));
    v2, v3 = address(v1).staticcall(uint32(0x650500c1)).gas(msg.gas);
    require(bool(v2), 0, RETURNDATASIZE()); // checks call status, propagates error data on error
    require(MEM[64] + RETURNDATASIZE() - MEM[64] >= 32);
    if (v3 == 42) {
        if (v1.code.size <= 10) {
            v4 = v5 = 1;
        } else {
            v4 = v6 = 0;
        }
    } else {
        v4 = v7 = 0;
    }
    return bool(v4);
}
```

其中` v2, v3 = address(v1).staticcall(uint32(0x650500c1)).gas(msg.gas); `调用一个方法获得的值存在v3；

然而` Keccak256("whatIsTheMeaningOfLife()") >> 224 `的值就为` 650500c1 `；

` if (v3 == 42) { `所以数字就为42；

` if (v1.code.size <= 10) { `这边是在检查合约代码长度；

捋一下逻辑，就是这个` solver `是我们部署的合约地址，其中有` whatIsTheMeaningOfLife() `这个函数，然后调用题目合约调用` setSolver `即可；

所以现在我们只需写一个小于10字节且包含指定函数的合约，如下：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Exp{
    function whatIsTheMeaningOfLife() public returns (uint256) {
        return 42;
    }
}
```

#### 运行时字节码

先写运行时的字节码，只要这部分不超过10字节即可；

返回一个值会用到` RETURN `，但它需要两个参数：

> 1. `offset`: byte offset in the [memory](https://www.evm.codes/about) in bytes, to copy what will be the [return data](https://www.evm.codes/about) of this [context](https://www.evm.codes/about).
> 2. `size`: byte size to copy (size of the [return data](https://www.evm.codes/about)).

所以我们得先将` 42 `存到内存中，这就需要用到` MSTORE `，它也需要两个参数：

> 1. `offset`: offset in the [memory](https://www.evm.codes/about) in bytes.
> 2. `value`: 32-byte value to write in the [memory](https://www.evm.codes/about).

所以，可以这么写：

```solidity
// 由于是栈结构，所以参数从右往左push
// 存储0x2a到内存中
PUSH 0x2a // value
PUSH 0x80 // offset,这边的位置可以随便选，选0x80是因为Solidity Memory中0x80才是真正的开始
MSTORE
// 返回值
PUSH 0x20 // size
PUSH 0x80 // offset
RETURN
```

` 602a60805260206080f3 `正好10字节；

#### 部署代码

部署代码需要将运行时代码返回给EVM；

可以用` CODECOPY `，它有三个参数：

> 1. `destOffset`: byte offset in the [memory](https://www.evm.codes/about) where the result will be copied.
> 2. `offset`: byte offset in the [code](https://www.evm.codes/about) to copy.
> 3. `size`: byte size to copy.

所以可以这么写：

```solidity
PUSH 0xa // size
PUSH 0xc // offset，此处需要计算运行时代码的开头在整个字节码的哪
PUSH 0x00// desOffset，查看其他合约部署的代码基本在0x00位置
CODECOPY
PUSH 0xa // size
PUSH 0x00// offset
RETURN
```

所以整个字节码就是` 600a600c600039600a6000f3602a60805260206080f3 `；

这边可以不用搞函数选择器，直接进来就是运行返回值；



### 攻击

![image-20250204211418589](./assets/image-20250204211418589.png)

