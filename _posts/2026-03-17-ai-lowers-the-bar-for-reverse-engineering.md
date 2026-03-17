---
layout: post
title: 记一次使用 AI 降低逆向分析门槛的过程
description: 记录一次借助 AI 处理 Android 脱壳和代码补全的过程
---

# 背景

这两年我越来越明显地感觉到，AI 会对逆向分析造成很大的影响。在 1 年前，我遇到一个 App，它使用了 native 加载 so 后动态加载 dex 的方式，并且
有自定义 VM 层来执行代码，并且有 Frida 等反动态跟踪，我对它无从下手。毕竟我不是专业的安全分析。

对于这种例子，传统的办法一般是：
1. 动态加载 Dex 使用 /prc/{pid}/maps 读取内存段，通过 dump 内存的方式恢复 dex 文件
2. 自定义 VM 则需要反编译 so 文件，即分析 VM 执行机制。一般 VM 都是重新映射了一遍 opcode，需要分析每一个 opcode 的行为并推测重新映射
3. Dex 文件也有可能被加密，需要找到解密的方法，这也需要分析 so 文件。

这一切的问题其实都指向了需要逆向 so 文件，阅读大量代码并分析推测才能做到。这也很符合 AI 能做的事，只要它的逻辑性足够强后，它就可以担负这个责任。
下面是我记录一次我重新使用 AI 逆向这个 App 的过程。

# 需要的工具

- IDA Pro。老牌的逆向分析工具。
- [IDA mcp](https://github.com/mrexodia/ida-pro-mcp)
  安装后，每次打开一个项目后需要点 Edit -> Plugins -> MCP 来运行 MCP server，这样在 Agent 工具中才能连接。
- [jadx](https://github.com/skylot/jadx)。专门做 Java / Android 的逆向分析工具。
- 一台 root 了的手机，并且已经装好 Xposed 框架

# 一些经验

1. 要提供告诉 AI 足够的上下文以及支持的能力，这样可以解放自己的双手：
   ```text
   你可以使用的工具：
   1. 我已使用 IDA 加载 libkadp.so，你可以使用 IDA mcp 来操作。
   2. 我已连接 adb，并且有 root 权限。
   3. 这手机有安装 Xposed，你可以使用 adb install 安装 Xposed 插件，安装后 kill 掉 app 进程然后重新 start activity 即可生效。 
   ```
   告诉它有 adb shell 权限后它就可以完成绝大部分操作，完全在那自己调试。
2. 及时记录已经完成了的一些中间成果。因为逆向分析会涉及大量的上下文，经过多次上下文自动压缩后可能会失真。一些已完成的成果比如 dump 内存插件等
   可以单独抽出来，从新起一个干净的 session，让它专注于每一步逆向过程。

```py
#!/usr/bin/env python3
"""
Decrypt KiwiSec-protected classes.dex chunks.
key_i = enc_i[1]  (second encrypted word of each chunk)
plain[j] = ROTR32(enc[j] XOR key, 6)
"""
import struct, zlib, hashlib, pathlib, sys

def rotr32(v, n): return ((v >> n) | (v << (32-n))) & 0xFFFFFFFF

def decrypt_chunk(data: bytes) -> bytes:
    """Decrypt one chunk: ROTR32(word XOR key, 6), key = words[1]"""
    words = list(struct.unpack_from(f'<{len(data)//4}I', data))
    key = words[1]
    plain = [rotr32(w ^ key, 6) for w in words]
    return struct.pack(f'<{len(plain)}I', *plain)

def fix_dex_checksum(data: bytearray) -> bytearray:
    sha1 = hashlib.sha1(bytes(data[32:])).digest()
    data[12:32] = sha1
    checksum = zlib.adler32(bytes(data[12:])) & 0xFFFFFFFF
    struct.pack_into('<I', data, 8, checksum)
    return data

def main():
    apk_path = pathlib.Path('dexs/001_base.apk')
    out_dir  = pathlib.Path('dexs/decrypted')
    out_dir.mkdir(exist_ok=True)

    import zipfile
    with zipfile.ZipFile(apk_path) as z:
        raw = z.read('classes.dex')
    data = bytearray(raw)
    print(f'classes.dex size: {len(data)//1024} KB')

    # Find KiwiSec header
    kiwi_magic = bytes([0x3a, 0xc2, 0xbd, 0xc2])
    hdr_off = data.find(kiwi_magic)
    if hdr_off < 0:
        print('ERROR: KiwiSec magic not found'); sys.exit(1)

    n_chunks, hdr_size = struct.unpack_from('<II', data, hdr_off + 8)
    data_start = hdr_off + hdr_size
    print(f'KiwiSec header at {hdr_off:#x}, n_chunks={n_chunks}, DATA_START={data_start:#x}')

    # Read chunk table: {out_size, cum_out}
    chunks = []
    for i in range(n_chunks):
        off = hdr_off + 16 + i * 8
        out_size, cum_out = struct.unpack_from('<II', data, off)
        chunks.append((out_size, cum_out))

    # Decrypt each chunk
    for i, (out_size, cum_out) in enumerate(chunks):
        chunk_off = data_start + cum_out
        chunk_data = bytes(data[chunk_off : chunk_off + out_size])

        if len(chunk_data) < 8:
            print(f'[{i:02d}] too small, skip'); continue

        decrypted = decrypt_chunk(chunk_data)

        # Verify DEX magic
        if decrypted[:4] == b'dex\n':
            # Valid DEX — fix checksum and save
            dec = bytearray(decrypted)
            file_size = struct.unpack_from('<I', dec, 0x20)[0]
            if file_size < 0x70 or file_size > len(dec):
                file_size = len(dec)
            dec = dec[:file_size]
            dec = fix_dex_checksum(dec)
            out_path = out_dir / f'chunk_{i:02d}.dex'
            out_path.write_bytes(dec)
            class_defs = struct.unpack_from('<I', dec, 0x60)[0]
            print(f'[{i:02d}] DEX  {len(dec)//1024:6d} KB  classes={class_defs:5d}  -> {out_path.name}')
        else:
            # Might be compressed — try zlib/lz4 later, save raw for inspection
            magic4 = decrypted[:4].hex()
            out_path = out_dir / f'chunk_{i:02d}.bin'
            out_path.write_bytes(decrypted)
            print(f'[{i:02d}] ???  {len(decrypted)//1024:6d} KB  magic={magic4}  -> {out_path.name}')

    print(f'\nDone. Output in {out_dir}/')

if __name__ == '__main__':
    main()
```

# 总结

现在 AI 对于逻辑分析类的操作非常强，传统的安全手段，加壳防护在这些逆向分析能力面前，原本需要大量的时间精力去分析，现在只需要支付大量的 token
即可解决，以后的应用安全该如何提升？
