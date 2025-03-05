## Level_26.DoubleEntryPoint

要求：

> 部署一个Bot，找到` CryptVault `的bug并且保护其` DET `不被扫空;

合约：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/access/Ownable.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

interface DelegateERC20 {
    function delegateTransfer(address to, uint256 value, address origSender) external returns (bool);
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}

contract Forta is IForta {
    mapping(address => IDetectionBot) public usersDetectionBots;
    mapping(address => uint256) public botRaisedAlerts;

    function setDetectionBot(address detectionBotAddress) external override {
        usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
    }

    function notify(address user, bytes calldata msgData) external override {
        if (address(usersDetectionBots[user]) == address(0)) return;
        try usersDetectionBots[user].handleTransaction(user, msgData) {
            return;
        } catch {}
    }

    function raiseAlert(address user) external override {
        if (address(usersDetectionBots[user]) != msg.sender) return;
        botRaisedAlerts[msg.sender] += 1;
    }
}

contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying;

    constructor(address recipient) {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}

contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate;

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}

contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(address legacyToken, address vaultAddress, address fortaAddress, address playerAddress) {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if (forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(address to, uint256 value, address origSender)
        public
        override
        onlyDelegateFrom
        fortaNotify
        returns (bool)
    {
        _transfer(origSender, to, value);
        return true;
    }
}
```

### 分析

题目很长，首先一个一个合约的看：

> ` interface DelegateERC20 `:有一个` delegateTransfer(to,value,originSender) `接口函数；
>
> ` interface IDetectionBot `:有一个` handleTransaction(user,msgData) `接口函数，也是我们需要实现的函数；
>
> ` interface IForta `:Forta的接口函数，有3个函数需要实现；
>
> 
>
> ` contract Forta `:
>
> 映射` usersDetectionBots `存放我们实现的Bot地址；
>
> 映射` botRaisedAlerts `存放Bot发出的警告数；
>
> 方法` setDetectionBot(BotAddress) `设置并存放Bot地址；
>
> 方法` notify(user,msgData) `Bot运行` handleTransaction(user,msgData) `检查处理交易；
>
> 方法` raiseAlert(user) `增加警告数；
>
> 
>
> ` contract CryptoVault `（持有100` LGT `和100` DET `）:
>
> 通过一个` recipient `地址来构造，有一个` underlying `token（题目中说了是` DET `，后面就只写` DET `了）；
>
> 方法` setUnderlying(lastToken) `可以设置一个新token；
>
> 方法` sweepToken(token) `可以向` recipient `发送该合约所拥有的该token，但不能` sweap `token` DET `；
>
> 
>
> ` contract LegacyToken `:
>
> ERC20代币` LGT `的合约；
>
> 有一个实现` DelegateERC20 `接口的` delegate `地址，可以用来调用` delegateTransfer(to,value,originSender) `；
>
> 方法` delegateToNewContract(newContract) `用来设置新的` delegate `；
>
> 重写的` transfer `方法使用了` delegate.delegateTransfer `，注意这不是` delegatecall `，下一个合约中实现了该函数，意味着只是` originSender `转帐到` to `；
>
> 
>
> ` contract DoubleEntryPoint `:
>
> ERC20代币` DET `的合约，同时实现了` DelegateERC20 `接口的` delegateTransfer `函数；
>
> 存储着` cryptoVault `、` player `、` delegatedFrom `、` forta `的地址；
>
> 修饰器` onlyDelegateFrom() `判断sender是否为` delegateFrom `；
>
> 修饰器` fortaNotify() `调用我们自己实现的Bot检查并触发警告；
>
> 实现的` delegateTransfer `函数从` originSender `转帐到` to `；

我们所拿到的题目地址是` DoubleEntryPoint `的，首先将几个状态变量的地址都找到：

```javascript
const DETAddress = await contract.address
// 0x57a6B590b6Ef7dfF5142e661159ee35a68B1E8A4

const cryptoVaultAddress = await contract.cryptoVault()
// 0x1CFe2eff88a0dE3A6889ff9556fC00FB7b593e99

const delegateFromAddress = await contract.delegatedFrom()
const LGTAddress = delegateFromAddress
// 由于delegatedFrom = legacyToken;所以代币LGT的地址也是这个
// 0xbFf599F7b3c0c76ab0AAeDaaB554cfDFe547D3C1

const fortaAddress = await contract.forta()
// 0xA2A5aBdB66b8F3E01277Ac43268328b4e0EB71a5
```

通过上面的这些地址，再深入看看各个合约的内部信息：

```javascript
// CryptoVault合约的sweptTokensRecipient地址，就是我们自己
await web3.eth.getStorageAt(cryptoVaultAddress, 0)
// 0x0000000000000000000000007148c25a8c78b841f771b2b2eead6a6220718390

// CryptoVault合约的underlying地址，就是DET的地址
await web3.eth.getStorageAt(cryptoVaultAddress, 1)
// 0x00000000000000000000000057a6b590b6ef7dff5142e661159ee35a68b1e8a4

// LGT代币中delegate的地址
await web3.eth.call({from: player,to: LGTAddress,data: '0xc89e4361'/*delegate()*/})
// 0x00000000000000000000000057a6b590b6ef7dff5142e661159ee35a68b1e8a4
```

> 上面最后获得delegate的地址时，为什么直接用call？
>
> 直接执行` await web3.eth.getStorageAt(LGTAddress, 0) `的结果为全0；
>
> 因为` LegacyToken `继承了` ERC20 `和` Ownable `，` slot 0 `处并不是` delegate `；
>
> 前面分别有` mapping _balances `、` mapping _allowances `、` uint256 _totalSupply `、` string _name `、` string _symbol `、` address _owner `；
>
> 根据计算，正好每个占一个slot，所以执行` await web3.eth.getStorageAt(LGTAddress, 6) `又可以拿到正确地址；

观察上面的地址，发现` LGT `代币中的` delegate `居然就是` DET `代币的地址；

再看看LGT合约中的` transfer `方法，` delegate.delegateTransfer(to, value, msg.sender); `，会直接将` DET `转给` to `；

而如果用` LGT `合约的地址作为` CryptoVault `合约中` sweapToken `方法的参数，` token.transfer() `便会执行上面的流程；

> CryptoVault.sweapToken(LGT) --> 
>
> ​    LGT.transfer() --> LGT.delegate.delegateTransfer() --> 
>
> ​        DET.delegateTransfer() --> DET._transfer()

所以，这样` CryptoVault `中的` DET `会被全部转走；

由于只有` DET.delegateTransfer() `这一步实现了` fortaNotify `，所以我们只能检测这部分的` calldata `；

` to `和` value `这两个值都可以随意更改，无法准确检测，只有` origSender `可以判断，因为此时这边的` origSender `一定是` CryptoVault `的地址；

下面开始推测发出的` calldata `，[calldata的布局](https://docs.soliditylang.org/en/v0.8.15/abi-spec.html#abi)；

调用` delegateTransfer `应该是：

```solidity
// 0x
// 9cd1a121 --> keccak(delegateTransfer(address,uint256,address))的前4个字节
// 32个字节 --> to
// 32个字节 --> value
// 0x0000000000000000000000001CFe2eff88a0dE3A6889ff9556fC00FB7b593e99 --> origSender，也就是我们需要检测的地方
```

但是，` delegateTransfer `有两个修饰器` onlyDelegateFrom `和` fortaNotify `；

修饰器并不是函数调用，不需要calldata，其中` _; `只是单纯的将代码移植到后面来；

所以` onlyDelegateFrom `无需关注，但` fortaNotify `需要注意一下，它其中调用了` notify `，并且是在这传入了调用` notify() `的` calldata `，然后方法中调用了` handleTransaction `方法，此时这边才是我们需要的` msg.data `；

> 传入执行` delegateTransfer `的calldata --> DET.delegateTransfer() --> 
>
> 传入执行` notify `的calldata --> Forta.notify() --> 
>
> 真正需要的calldata，执行` handleTransaction `的calldata --> IDetectionBot.handleTransaction()

所以这才是传入的calldata：

| 偏移 |                             数据                             |                             备注                             |
| :--: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| 0x00 |                           220ab6aa                           |     keccak(handleTransaction(address,bytes))的前4个字节      |
| 0x04 |                        地址(32个字节)                        |                             user                             |
| 0x24 |                     0x00..0020(32个字节)                     | msgData的偏移(bytes的开始包括长度信息)，从自身开始算，即0x24开始，一共0x20个字节就是长度信息 |
| 0x44 |                     0x00..0064(32个字节)                     | msgData的长度，即下面的几个字段，0x4 + 0x20 + 0x20 + 0x20 = 0x64 |
| 0x64 |                           9cd1a121                           | keccak(delegateTransfer(address,uint256,address))的前4个字节 |
| 0x68 |                        地址(32个字节)                        |                              to                              |
| 0x88 |                      uint256(32个字节)                       |                            value                             |
| 0xA8 | 0x0000000000000000000000001CFe2eff88a0dE3A6889ff9556fC00FB7b593e99 |             origSender，也就是我们需要检测的地方             |
| 0xC8 |                    00..00(32-4=28个字节)                     |      填充部分，因为从偏移0x4开始的后面需要为32字节倍数       |

我们只需检测` 0x68 `处的值是否为` CryptoVault `的地址即可；



### 攻击

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./Ethernaut.sol";

contract Exp is IDetectionBot {
    address private vault;
    constructor(address _vault){
        vault = _vault;
    }
    function handleTransaction(address user, bytes calldata) external override {
        address to;
        uint256 value;
        address originSender;

        assembly {
            to := calldataload(0x68)
            value := calldataload(0x88)
            originSender := calldataload(0xa8)
        }
        if (originSender == vault){
            Forta(msg.sender).raiseAlert(user);
        }
    }
}
```

在` Forta `中设置Bot即可；

