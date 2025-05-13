---
layout: post
title: 使用 IDA 反编译 IL2CPP
description: 结合 frida-il2cpp-bridge 的一次应用实践
---

# 前菜

拿到 apk 后直接 unzip 即可解包，解包发现 libil2cpp.so 和 gobal-metadata.dat。但是 global-metadata.dat 的文件头是
`0x444F4447`，明显被加密了，所以最简单的方法是使用动态方法来解析，其原理可以参考[这篇文章](https://www.perfare.net/archives/1741)。
大体是说所有使用 il2cpp 打包的应用，最后都会留有一些 il2cpp_x 的函数可以作为后门获取整个应用的函数签名及地址。

![Il2Cpp functions]({{ "/assets/images/2025-05-12-il2cpp-functions.png" | relative_url }})

# 导出 libil2cpp.so 和符号表

导出符号表就比较简单了，直接 `npx frida-il2cpp-bridge -U -f xxx dump` 就行了。导出的符号表类似这样：

```csharp
// mscorlib
class System.String : System.Object
{
    System.Int32 _stringLength; // 0x10
    System.Char _firstChar; // 0x14
    static System.String Empty; // 0x0
    static System.Boolean EqualsHelper(System.String strA, System.String strB); // 0x03a597f0
}
```

同时它的 libil2cpp.so 也加密了，需要动态 dump，就稍微写一下脚本问题也不大：

```ts
import "frida-il2cpp-bridge";

Il2Cpp.perform(() => {
    const m = Process.getModuleByName('libil2cpp.so');
    Memory.protect(m.base, m.size, 'rwx');
    const buf = Memory.readByteArray(m.base, m.size);
    const f = new File('/data/local/tmp/libil2cpp.so', 'wb'); // 注意权限
    f.write(buf);
    f.close();
    console.log('Dump to libil2cpp.so');
});
```
执行后 so 文件被写到手机本地，使用 adb pull 拉回来即可。导出的 so 如果导入 IDA 打开失败的话，需要使用 [SoFixer](https://github.com/F8LEFT/SoFixer)
修复后再导入。

至此，你已经拥有了 libil2cpp.so 和符号表，接下来就可以使用 IDA 进行反编译了。

# 开始分析

我的目标是找到网络协议的编码规则。 导出的符号信息里面，我们可以看到基类地址，无论是 Ghidra 还是 IDA 都有 struct 的功能，我们可以手动重建
struct 来实现更可读的逆向代码。首先我找到 `NetClient.SendMsg`，这看起来就很可疑，直接使用 Frida hook 一下：

```ts
Il2Cpp.perform(() => {
    const arrayBufferToHex = (buffer) => {
        return Array.from(new Uint8Array(buffer))
            .map(byte => byte.toString(16).padStart(2, '0'))
            .join('');
    }
    const XXXNative = Il2Cpp.domain.assembly('XXXNative');

    const NetClientSendMsg = XXXNative.image.class('XXX.NetClient').method('SendMsg');

    NetClientSendMsg.implementation = function (buf, simpleEnc): boolean {
        const bufHex = arrayBufferToHex(ptr(buf.elements).readByteArray(buf.length)); 
        console.log('SendMsg:' + bufHex + ', simpleEnc:' + simpleEnc);
        return this.method('SendMsg')<boolean>().invoke(buf, simpleEnc);
    };
});
```

这样就可以看到每条发出去的消息，但是存在两个问题：

1. 不知道发出去的消息是什么方式编码的；
2. 发出去的消息和实际网络里抓包看到的不一样，最后还存在一层加密。

![Msg pack]({{ "/assets/images/2025-05-12-ida-msg-pack.png" | relative_url }})

经过一番符号表回填及分析，发现了消息 pack 的方法，即序列化方法，后面只需要把对应的 struct 解出来就行了，这个先放一下，接下来需要找到加密协议

![Encode buf]({{ "/assets/images/2025-05-12-ida-encode-buf.png" | relative_url }})

然后继续沿着这个信息走，找到了 simpleEnc 为 true/false 的时候不同的加密方法。首先看比较不 simple 的方法：

![Tdr buf encode]({{ "/assets/images/2025-05-12-ida-tdr-encode.png" | relative_url }})

关键是这个加密函数，直接找出来然后自己复原一下。我使用 py 的话需要注意溢出问题，可以使用 `np.int32` 等类型来模仿溢出行为。另外可以使用 AI，
把反汇编出来的代码让它直接翻译，然后自己再稍微改改就能用，还是挺好用的。

![Encrypt func]({{ "/assets/images/2025-05-12-ida-encrypt-func.png" | relative_url }})

接下来比较头疼的是，接收的信息里面有 cmd id，但是 cmd id 对应会使用不同的 struct 来解析，这需要一个映射表，经过分析它把映射表放到多个函数里，
每个函数里有 20 个 switch 来组成。所以需要解析这个 switch 并拿到对应的 struct。

![Switch case]({{ "/assets/images/2025-05-12-ida-switch-case.png" | relative_url }})

我们可以稍微写点脚本来自动化遍历所有 switch case，新的 IDA 有 ida_hexrays 更方便去获取 switch case 分支不需要管背后的 jtable 了。

```python
import json

import ida_funcs
import ida_hexrays

class CmdIDPackFuncVisitor(ida_hexrays.ctree_visitor_t):
    def __init__(self):
        ida_hexrays.ctree_visitor_t.__init__(self, ida_hexrays.CV_FAST)
        self.cmd_func_map = {}

    def visit_insn(self, insn: ida_hexrays.cinsn_t) -> int:
        if insn.op == ida_hexrays.cit_switch:
            switch: ida_hexrays.cswitch_t = insn.cswitch
            cases: ida_hexrays.ccases_t = switch.cases
            for case in cases:
                cmd_ids: [int] = case.values
                blocks: ida_hexrays.cblock_t = case.cblock
                for block in blocks:
                    if block.op == ida_hexrays.cit_expr:
                        expr: ida_hexrays.cexpr_t = block.cexpr
                        if expr.op == ida_hexrays.cot_asg:
                            y: ida_hexrays.cexpr_t = expr.y
                            if y.op == ida_hexrays.cot_call and y.a.size() > 2:
                                y_x: ida_hexrays.cexpr_t = y.x
                                for cmd_id in cmd_ids:
                                    self.cmd_func_map[cmd_id] = y_x.obj_ea
                                #print(f'Find {','.join(map(lambda x: str(x), cmd_ids))}, unpack by {hex(y_x.obj_ea)}')
        return 0

class FuncVisitor(ida_hexrays.ctree_visitor_t):
    def __init__(self):
        ida_hexrays.ctree_visitor_t.__init__(self, ida_hexrays.CV_FAST)
        self.cmd_func_map = {}

    def visit_expr(self, expr: ida_hexrays.cexpr_t) ->int:
        if expr.op == ida_hexrays.cot_asg:
            y: ida_hexrays.cexpr_t = expr.y
            if y.op == ida_hexrays.cot_call and len(y.a) == 5:
                y_x: ida_hexrays.cexpr_t = y.x
                sub_func: ida_funcs.func_t = ida_funcs.get_func(y_x.obj_ea)
                sub_cfunc: ida_hexrays.cfuncptr_t = ida_hexrays.decompile_func(sub_func)
                if sub_cfunc is None:
                    raise Exception('decompile func failed')
                visitor = CmdIDPackFuncVisitor()
                visitor.apply_to(sub_cfunc.body, None)
                self.cmd_func_map.update(visitor.cmd_func_map)
        return 0


def main():
    if not ida_hexrays.init_hexrays_plugin():
        raise Exception('hexrays not loaded')

    func: ida_funcs.func_t = ida_funcs.get_func(0x2B20104)
    if func is None:
        raise Exception

    func_ptr: ida_hexrays.cfuncptr_t = ida_hexrays.decompile_func(func)
    if func_ptr is None:
        raise Exception('decompile func failed')

    visitor = FuncVisitor()
    visitor.apply_to(func_ptr.body, None)

    msg_map = dict(sorted(visitor.cmd_func_map.items()))
    print(len(msg_map))
    print(json.dumps(msg_map))

if __name__ == "__main__":
    main()
```

通过这个脚本，我们找到了所有 cmd_id 的 pack function 了，但遗憾地，原始的符号表就已经混淆了防了一手。好在我拿到旧版本的 apk，发现它的不仅
没有加密 global-meta，符号表也是没有混淆的。如果找不到这一步，那只能更费力地去推测每个字段的含义。

![Mixed fields]({{ "/assets/images/2025-05-12-mixed-proto-fields.png" | relative_url }})

# References

[Il2CppDumper](https://github.com/Perfare/Il2CppDumper)

[Android安全-frida-il2cpp-bridge使用](http://www.yxfzedu.com/article/12473)

[IDA pro Python API](https://python.docs.hex-rays.com/)
