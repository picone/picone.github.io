---
layout: post
title: EVM bytecode 反编译
description: 反编译最近比较火的起源岛游戏的合约
---

# 背景

最近在玩起源岛链游时，发现交互的合约没有上传源码校验。其实我挺反感这种开发者的，因为在编译的时候就生成了 bytecode 和对应的源码 json 文件，只要顺手
上传一下就能大家都看到了，并且放心地使用这个合约，但是却不上传。不上传合约源码只会提高小白使用的门槛，并不会降低被攻击的风险。

# 过程

## 工具准备

目前 EVM bytecode 有很多工具可以反编译，比如 [ethervm](https://ethervm.io/decompile)。但是他们是在线的，对于较大的合约不支持，所以需要
本地的工具，多番查找后找到了比较好用的 [panoramix](https://github.com/palkeo/panoramix)。

安装很简单，只需要 `pip install panoramix-decompiler-abi` 即可。

## 反编译

反编译也非常简单，panoramix 默认是主网，如果使用一些侧链比如 Polygon，则要使用对应的 rpc，方法是
`export WEB3_PROVIDER_URI=https://rpc-mainnet.matic.quiknode.pro`。然后执行
`panoramix 0xcF6cD657dDf8f5e0BEAa2Ad4CA9550c59A32685f` 这样的命令就好了，等待反编译结果。首次运行会比较慢，它会下载签名词典让结果更可读。

## 结果

结果的阅读就比较繁琐了，以起源岛的合约为例，withdraw 叶子的函数签名是 `db909505`，对应反编译出来的代码是：

```text
def unknowndb909505(uint256 _param1, uint256 _param2, uint256 _param3, uint256 _param4, array _param5) payable:
  require calldata.size - 4 >=′ 160
  require _param5 <= 18446744073709551615
  require _param5 + 35 <′ calldata.size
  require _param5.length <= 18446744073709551615
  require _param5 + _param5.length + 36 <= calldata.size
  if not stor161:
      revert with 0, 'contract is not initialized'
  if paused:
      revert with 0, 'Pausable: paused'
  if _param1 != 101:
      revert with 0, 'invalid item id'
  if _param4 <= block.timestamp:
      revert with 0, 'param expired'
  if stor151[caller] > !stor159:
      revert with 0, 17
  if stor151[caller] + stor159 >= block.timestamp:
      revert with 0, 'withdraw locking.'
  if order[_param2]:
      revert with 0, 'repeated order id'
  require ext_code.size(stor153)
  call stor153.verify(bytes32 param1, bytes param2) with:
       gas gas_remaining wei
      args sha3(caller, _param1, _param2, _param3, _param4), Array(len=_param5.length, data=_param5[all])
  if not ext_call.success:
      revert with ext_call.return_data[0 len return_data.size]
  stor151[caller] = block.timestamp
  order[_param2] = block.timestamp
  require ext_code.size(stor157)
  call stor157.mint(address owner, uint256 value) with:
       gas gas_remaining wei
      args caller, _param3
  if not ext_call.success:
      revert with ext_call.return_data[0 len return_data.size]
  log 0x38484d77: _param2
  return 1, _param2
```

通过上下文阅读理解，可以确定 `param1` 是 item_id，这个方法固定是 101；`param2` 是 order_id，类似于 nonce 这样的存在，用于防止重复交易；
`param3` 是购买的数量；`param4` 是过期时间，用于防止过久的请求发生重放，增加合约升级风险；`param5` 是参数签名。有一些外部合约调用，比如：

```text
call stor153.verify(bytes32 param1, bytes param2) with:
  gas gas_remaining wei
  args sha3(caller, _param1, _param2, _param3, _param4), Array(len=_param5.length, data=_param5[all])
```

这里意思是 `stor153` 存放的地址，调用 `verify` 方法，传入两个参数，第一个参数是 `sha3(caller, _param1, _param2, _param3, _param4)` 
的 结果，第二个参数是 `param5` 转成 bytes 数组传进去，这里的 `sha3` 是 `keccak256` 的别名。为了找到 `stor153` 的地址，需要查看源码的其它
地方，推测是通过 init 函数赋值的。通过查找合约里其他代码，找到全文唯一一个赋值 `stor153` 的地方：

```text
def unknownf7013ef6(uint256 _param1, uint256 _param2, uint256 _param3, uint256 _param4, uint256 _param5) payable:
  ...
  stor153 = addr(_param1)
```

所以我们可以找到这个合约调用 `7013ef6` 的方法时传入的参数，找到唯一一次调用的 txn：[0xee4973c55f7be22ae75738aa69accbbc01039d1c5bf2ebd57eced1c1d53fa39e](https://polygonscan.com/tx/0xee4973c55f7be22ae75738aa69accbbc01039d1c5bf2ebd57eced1c1d53fa39e)。
通过这个 transaction，我们找到 `param1` 的值是 `0x8b239FAfaBCE0a27e62789738D0c98aDdb7B5815`，所以 `stor153` 的地址就是这个地址。
继续对其进行反编译 `panoramix 0x8b239FAfaBCE0a27e62789738D0c98aDdb7B5815`，可以找到 verify 方法的实现：

```text
def verify(bytes32 _param1, bytes _param2) payable: 
  require calldata.size - 4 >=′ 64
  require _param2 <= 18446744073709551615
  require _param2 + 35 <′ calldata.size
  require _param2.length <= 18446744073709551615
  require _param2 + _param2.length + 36 <= calldata.size
  mem[128 len _param2.length] = _param2[all]
  mem[_param2.length + 128] = 0
  require 65 == _param2.length
  mem[ceil32(_param2.length) + 224] = mem[128]
  if Mask(8, -(('mask_shl', 256, 0, -3, ('mem', ('range', 192, 32))), 0) + 256, 0) << (('mask_shl', 256, 0, -3, ('mem', ('range', 192, 32))), 0) - 256 >= 27:
      signer = erecover(_param1, 0, mem[128], mem[160]) # precompiled
  else:
      signer = erecover(_param1, 27, mem[128], mem[160]) # precompiled
  if not erecover.result:
      revert with ext_call.return_data[0 len return_data.size]
  mem[ceil32(_param2.length) + 192 len 42] = call.data[calldata.size len 42]
  mem[ceil32(_param2.length) + 193 len 8] = Mask(8, -(6784692728748995825599862402852807100777538164002376799186967812963659939840, 0) + 256, 0) << (6784692728748995825599862402852807100777538164002376799186967812963659939840, 0) - 256
  idx = 41
  s = addr(signer)
  while idx > 1:
      if s % 16 >= 16:
          revert with 0, 50
      if idx >= 42:
          revert with 0, 50
      mem[idx + ceil32(_param2.length) + 192 len 8] = Mask(8, -(0, 0) + 256, 0) << (0, 0) - 256
      if not idx:
          revert with 0, 17
      idx = idx - 1
      s = Mask(252, 0, s) * 0.0625
      continue 
  if addr(signer) + 10240:
      revert with 0, 'Strings: hex length insufficient'
  mem[ceil32(_param2.length) + 288] = 'verify failed: account '
  mem[ceil32(_param2.length) + 311 len 64] = 0, mem[ceil32(_param2.length) + 193 len 63]
  mem[ceil32(_param2.length) + 353] = 0x206973206d697373696e6720726f6c6520000000000000000000000000000000
  if unknown248a9ca3[0x5860476b5a14ec223973c49cf384357b5eb5f6e5d3c9264c3b1a3dba97f3f33][addr(signer)].field_0:
      stop
  mem[ceil32(_param2.length) + 370] = 0x8c379a000000000000000000000000000000000000000000000000000000000
  mem[ceil32(_param2.length) + 374] = 32
  mem[ceil32(_param2.length) + 406] = mem[160]
  mem[ceil32(_param2.length) + 438 len ceil32(mem[160])] = mem[ceil32(_param2.length) + 288 len ceil32(mem[160])]
  if ceil32(mem[160]) > mem[160]:
      mem[ceil32(_param2.length) + mem[160] + 438] = 0
  revert with 0, 32, mem[160], mem[ceil32(_param2.length) + 438 len ceil32(mem[160])]
```

好了，看到 `erecover` 就可以停手了，这是一个用签名恢复地址的方法，可以用来验证签名是否正确。可以参考 OpenZeppelin 的关于 
[ECDSA](https://docs.openzeppelin.com/contracts/2.x/api/cryptography#ECDSA) 的介绍。我们跟踪 `signer` 发现它最终是查找
`unknown248a9ca3` 是否存在这个地址，这个变量的定义是 `def storage: unknown248a9ca3 is mapping of struct at storage 0`。同样地，我
们找上下文看看是哪里赋值的，最后找到这个函数：

```text
def unknown2f2ff15d(uint256 _param1, uint256 _param2) payable:
  require calldata.size - 4 >=′ 64
  require _param2 == addr(_param2)
  if unknown248a9ca3[unknown248a9ca3[_param1].field_256][caller].field_0:
      if not unknown248a9ca3[_param1][addr(_param2)].field_0:
          unknown248a9ca3[_param1][addr(_param2)].field_0 = 1
          log 0x2f878811: _param1, addr(_param2), caller
      stop
```

必须先 role 存在才能设置为 `.field_0 = 1`。哎退了吧。这段代码十分像是 OpenZeppelin 里面的 AccessControl 的代码，可以参考[这里](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/AccessControl.sol)大概就知道代码
在说什么了。

我们可以执行 `2f2ff15d` 这个函数试一试，可以 fork 一个本地节点来调试，执行
`npx hardhat node --fork https://rpc-mainnet.matic.quiknode.pro`，然后就会显示本地 rpc 的地址。我们使用 python 进行调试：

```python
from web3 import HTTPProvider, Web3

w3 = Web3(HTTPProvider('http://127.0.0.1:8545'))

account = w3.eth.account.from_key('0xxxxxxx')
params = w3.codec.encode(
    ['uint256', 'address'],
    [w3.to_int(hexstr='05860476b5a14ec223973c49cf384357b5eb5f6e5d3c9264c3b1a3dba97f3f33'), account.address],
)
data = w3.to_bytes(hexstr='2f2ff15d') + params
res = w3.eth.call({
    'to': '0x8b239FAfaBCE0a27e62789738D0c98aDdb7B5815',
    'data': w3.to_hex(data),
})
print(res)
```
执行结果是 revert 了，返回
`AccessControl: account 0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266 is missing role 0x0000000000000000000000000000000000000000000000000000000000000000`。
