---
layout: post
title: 逆向工程实战（二）
description: 记录如何逆向一个 Unity3D WebGL 游戏的逆向过程和思路
---

# Description

最近在参与一款区块链的游戏，它是基于 Unity3D 做的网页游戏，使用 wasm 调用 WebGL 接口进行画面渲染。对于 wasm 的逆向网上的资料较少，中文资料就更
少了，因此操作起来有点困难。下面我记录一下整个过程。

## Goal

分析和发现这款游戏的通信协议加解密过程，从而可以自行编写脚本进行交互。

## Requirement

- [UnityPack](https://github.com/HearthSim/UnityPack)，用于从 data 文件中解包出 Unity 资源文件，包括 global-meta 文件。
- [Il2CppDumper](https://github.com/Perfare/Il2CppDumper)，用于从 wasm 打包中解析出 dll 和 global-meta 中解析出符号表。
- [Ghidra](https://github.com/NationalSecurityAgency/ghidra)，NSA 开源的一款逆向工具，主要是免费。
- [ghidra-wasm-plugin](https://github.com/nneonneo/ghidra-wasm-plugin)，Ghidra 的 wasm 插件。

# Steps

## Unpack resource assets

参考 UnityPack，解包出 data 文件的 Unity 资源。

```python
import os
import struct

SIGNATURE = 'UnityWebData1.0'

class BinaryReader:
    def __init__(self, buf, endian="<"):
        self.buf = buf
        self.endian = endian

    def align(self):
        old = self.tell()
        new = (old + 3) & -4
        if new > old:
            self.seek(new - old, os.SEEK_CUR)

    def read(self, *args):
        return self.buf.read(*args)

    def seek(self, *args):
        return self.buf.seek(*args)

    def tell(self):
        return self.buf.tell()

    def read_string(self, size=None, encoding="utf-8"):
        if size is None:
            ret = self.read_cstring()
        else:
            ret = struct.unpack(self.endian + "%is" % (size), self.read(size))[0]
        try:
            return ret.decode(encoding)
        except UnicodeDecodeError:
            return ret

    def read_cstring(self) -> bytes:
        ret = []
        c = b""
        while c != b"\0":
            ret.append(c)
            c = self.read(1)
            if not c:
                raise ValueError("Unterminated string: %r" % (ret))
        return b"".join(ret)

    def read_boolean(self) -> bool:
        return bool(struct.unpack(self.endian + "b", self.read(1))[0])

    def read_byte(self) -> int:
        return struct.unpack(self.endian + "b", self.read(1))[0]

    def read_ubyte(self) -> int:
        return struct.unpack(self.endian + "B", self.read(1))[0]

    def read_int16(self) -> int:
        return struct.unpack(self.endian + "h", self.read(2))[0]

    def read_uint16(self) -> int:
        return struct.unpack(self.endian + "H", self.read(2))[0]

    def read_int(self) -> int:
        return struct.unpack(self.endian + "i", self.read(4))[0]

    def read_uint(self) -> int:
        return struct.unpack(self.endian + "I", self.read(4))[0]

    def read_float(self) -> float:
        return struct.unpack(self.endian + "f", self.read(4))[0]

    def read_double(self) -> float:
        return struct.unpack(self.endian + "d", self.read(8))[0]

    def read_int64(self) -> int:
        return struct.unpack(self.endian + "q", self.read(8))[0]

class DataFile:
    def load(self, file):
        buf = BinaryReader(file, endian="<")
        self.path = file.name

        self.signature = buf.read_string()
        header_length = buf.read_int()
        if self.signature != SIGNATURE:
            raise NotImplementedError('Invalid signature {}'.format(repr(self.signature)))

        self.blobs = []
        while buf.tell() < header_length:
            offset = buf.read_int()
            size = buf.read_int()
            namez = buf.read_int()
            name = buf.read_string(namez)
            self.blobs.append({ 'name': name, 'offset': offset, 'size': size })
        if buf.tell() > header_length:
            raise NotImplementedError('Read past header length, invalid header')

        for blob in self.blobs:
            buf.seek(blob['offset'])
            blob['data'] = buf.read(blob['size'])
            if len(blob['data']) < blob['size']:
                raise NotImplementedError('Invalid size or offset, reading past file')

f = open('webglBuild_28.data', 'rb')
df = DataFile()
df.load(f)
for blob in df.blobs:
    print('extracting @ {}:\t{} ({})'.format(blob['offset'], blob['name'], blob['size']))
    dest = os.path.join('extracted', blob['name'])
    os.makedirs(os.path.dirname(dest), exist_ok=True)
    with open(dest, 'wb') as f:
        f.write(blob['data'])
```

解包出来的资源目录如下：

```text
extracted
├── Il2CppData
│   └── Metadata
│       └── global-metadata.dat
├── Managed
│   └── mono
│       └── 4.0
│           └── machine.config
├── Resources
│   └── unity_default_resources
├── RuntimeInitializeOnLoads.json
├── ScriptingAssemblies.json
├── boot.config
├── data.unity3d
└── resources.resource
```

这里重点是 global-metadata.dat 文件，这个包含了游戏核心的符号表，能对应每个函数的实现和地址偏移。

## Dump symbols

这里使用到 Il2CppDumper，使用就很简单了。这个工具需要 .NET 环境执行，我是在 mac 下安装虚拟机跑的，看 issue 下面有人说可以 mac 下安装 .NET
环境执行，感兴趣的可以折腾一下。

```shell
Il2CppDumper.exe webglBuild_28.wasm global-metadata.dat
```

## Decompile

我使用的是 Ghidra，理论上 IDA 也可以。使用 Ghidra 需要先安装 wasm 插件才能支持 wasm 格式的文件反编译。如果直接使用 Release 产出的版本，需要
严格对应 Ghidra 版本号。对我来说可能自己编译会更简单，参考 [Custom build](https://github.com/nneonneo/ghidra-wasm-plugin#custom-build)。

反编译后文件会是混乱的状态，全是有 function 偏移组成，因此需要关联上 Il2CppDumper 导出的符号表映射过来方便我们阅读代码。在 Script Manager 中

![Script Manager]({{ "/assets/images/2022-11-25-reverse-engineering-practice-ghidra-script-manager.png" | relative_url }})

这个脚本来源是 [Il2CppDumper](https://github.com/Perfare/Il2CppDumper/blob/master/Il2CppDumper/ghidra_wasm.py) 中。执行脚本后
functions 列表中的函数名称都会变成可读的了。

但是目前这个脚本仍有一些做不好，需要手动更新函数签名中各个参数的名字。举栗子，如 Ghidra 反编译出来的函数签名是：
`void Playground.App.LocalPlayer$$ScriptRequest<object,-object>(undefined4 __this,undefined4 type,undefined4 ars,undefined4 del,int method)`
通过 Il2CppDumper 的 `script.json`，通过 Name 中查找到 `Playground.App.LocalPlayer$$ScriptRequest` 对应的 Signature，对照填充回
去就好。

另外一个是，反编译后的代码中有一些 invoke 方法脚本并没有处理，需要手动去关联，如：`import::env::invoke_vi(&DAT_ram_000061ca,uVar2);`，
对应调用的函数偏移地址是 `0x61ca`，在 `script.json` 中找到 Address 是 25034，且 TypeSignature 是 `vi` 的函数签名，调用的函数就是这个了。

# Reference

- [浅谈逆向 Unity WebGL Il2Cpp 中 WebAssembly 函数的方法](https://www.cnblogs.com/algonote/p/15596459.html)
