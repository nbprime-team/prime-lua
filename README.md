# primeLua
#### 把 Lua 5.4 移植到 HP Prime G1

> 跑的是**官方 Lua 5.4.7 解释器**，与桌面 `lua` 的差异只有那些**无法相同**的部分，如没有进程、没有环境变量等

---

## 1. 它是怎么跑起来的

```
primeLua.hpappdir
  └─ import main                       ← primeLua.hpappdir/main.py
       ├─ 固件文件 API 列出 C:\DATA\primeLua.hpappdir\*.lua（find_first/next）
       └─ 选中脚本后：
            ├─ shellcode ELF loader
            │    · malloc 一块 8 字节对齐的镜像内存，加载 PT_LOAD 段
            │    · 清零 .bss 尾巴、应用 R_ARM_RELATIVE 重定位
            ├─ 构造 struct plua_config {magic, argc, argv, epoch, heap}
            └─ dbg.call(plua_entry, &config, 0)        ← r0 = 配置指针
                 │
lua.elf          ▼
  plua_entry (port/plua_svc.s)
    · 保存 r4-r11/ip/lr（loader 的 shellcode 信任 AAPCS）
    · SP 对齐到 8（ARM926 上未对齐的 LDRD/STRD 会触发 Data Abort → 整机复位）
    · 从固件堆 malloc 程序栈
    · plua_begin(r0)：登记 config、初始化控制台
    · plua_main()
    · 结束后写 "RT_RET:<n>" 到环形缓冲、归还栈、恢复所有寄存器、返回
         │
         ▼
      Lua 5.4.7 VM ── print/io.write ──► PLUARING 环形缓冲（32 KB）
                                          ▲
计算器端 main.py ── 轮询 ─────────────────┘  边跑边打印
```

**程序跑在固件线程上**：`plua_entry` 用固件的 `svc 0x10000` 起一个线程
（`sys_create_thread(plua_thread_entry, &config, 512 KB)`），把运行状态结构体的地址交回启动器后**立刻返回**。于是启动器在程序
还在跑的时候就拿回了控制权，可以一边读环形缓冲一边打印，也可以在用户按 ESC
时把运行状态里的 `abort` 字置 1，让解释器自己的调试钩子把脚本 error 掉。

```
lua.elf          ▼
  plua_entry (port/plua_svc.s)
    · 保存 r4-r11/ip/lr（loader 的 shellcode 信任 AAPCS）
    · SP 对齐到 8（ARM926 上未对齐的 LDRD/STRD 会触发 Data Abort → 整机复位）
    · plua_run_thread_start(&config)（port/plua_run.c）
         ├─ 填运行状态 {state, mode, ring, ring_start, ...}（magic "PLUARUN"）
         ├─ svc 0x10000 起线程：r0=线程体、r1=0、r2=512 KB 栈
         ├─ svc 0x10003 设优先级：r0=create_thread 返回的句柄、r1=60
         └─ 返回运行状态地址 → r0 → 启动器
              │
   固件线程   ▼  plua_thread_entry → plua_thread_main(config)
    · plua_begin(config)：登记 config、初始化控制台、写 plua.log
    · plua_main() → lua.c 的 main()（原封不动，只把符号名改成
      plua_lua_standalone_main）
    · 结束后写 "RT_RET:<n>" 到环形缓冲、把 state 置 2（DONE）、线程返回
```

* **argv 从哪来**：计算器没有 `argv`。启动器把 `{"lua", "C:\\DATA\\...\\脚本.lua"}`
  写进配置块，`plua_main` 取出后原样交给 `lua.c` 的参数解析 —— 所以 `-e`、`-i`、
  脚本参数等行为与桌面一致。

### 内存需求

| 项 | 大小 |
|---|---|
| 镜像 | ~455 KB（两个 PT_LOAD 的文件字节：390,640 + 70,220） |
| 程序栈 | **线程栈 512 KB**，由固件在 `svc 0x10000` 时分配、线程结束时收回。旧的"自己 malloc 128 KB 程序栈"只在 `nothread` 的同步模式里还用（失败就 64/32 KB，最后退到 24 KB 静态缓冲） |
| Lua 堆（固件堆，按需） | 脚本决定；测试套件实测峰值约 400 KB |
| 输出环形缓冲（镜像 .data 内） | 32 KB |

---

## 2. primeLua 自己实现的平台层

Lua 不依赖 libc，只依赖一组 C 函数。`port/` 下的文件提供它们；**移植的难点全在这里，Lua 本体一行没改**。

| 文件 | 作用 | 关键点 |
|---|---|---|
| `plua_svc.s` | 固件 SVC 包装 + `setjmp/longjmp` + 入口 `plua_entry` 与线程入口 `plua_thread_entry` | SVC 约定 `push{r0};push{lr};svc N`；`longjmp` 是 Lua 全部错误机制的基石（`ldo.c` 的 `LUAI_THROW`）；入口里 `bic sp,sp,#7` 与"压两个字"的栈对齐都是真机 Data Abort 换来的 |
| `plua_main.c` | 入口粘合：config → argc/argv、`exit()` 展开、`RT_RET` | `exit()` 没有可退之处，longjmp 回这里，让入口还能给启动器返回状态 |
| `plua_run.c` | **运行方式**：svc 0x10000 起固件线程跑解释器、运行状态（magic `PLUARUN`）、中断（abort）字与调试钩子检查 | 入口立刻返回，启动器边跑边读；线程栈由固件给（512 KB），所以同步模式里那个 malloc 的程序栈只在 `nothread` 下才用 |
| `plua_format.c` | **自己的 C99 printf 引擎** | `%.14g` 是 Lua 打印每个数字用的格式 |
| `plua_strtod.c` | **自己的正确舍入 strtod** | Lua 词法分析器用它解析每个数字字面量 |
| `plua_file.c` | FILE 层：stdout/stderr → 环形缓冲，文件 → 固件宽字符 API | 必须实现 `ungetc` 与 `freopen`：`luaL_loadfilex` 靠它们嗅探 `#`/BOM/字节码 |
| `plua_libc.c` | `realloc`、`exit/abort`、`errno`、`locale`、`ctype`、`strerror`（glibc 同字面）、`strtoul` | 固件堆只保证 4 字节对齐，而 ARM926 上未对齐 LDRD/STRD 会复位 → realloc 里要做对齐搬迁 |
| `plua_math.c` | 标准名 → openlibm 的 `hp_*` | 补 openlibm 缺的 `log2`与 `copysign` |
| `plua_time.c` | `gmtime/localtime/mktime/strftime/difftime/clock` | `%c/%x/%X/%e` 对齐 glibc 的 C locale
| `port/include/` | 遮蔽 newlib 的头文件（stdio/stdlib/string/math/errno/locale/time/ctype/setjmp/signal/assert） | 只留 GCC 自带的 `stddef/stdarg/stdint/float/limits` |

---

## 3. 构建与部署

### 构建（主机侧，需要 `arm-none-eabi-gcc`）

```sh
cd primeLua
make                 # lua.elf + 部署目录 ../primeLua.hpappdir/
make hostref         # 主机参考 lua（差分测试用，构建在 build/host/）
make test            # 第 4 层：固件模拟器端到端差分（8 并行，约 1.5 分钟）
make test-launcher   # 第 3 层：启动器本身（mock MicroPython + 模拟器）
make test-modes      # 固件文件模式翻译（纯主机，秒级）
make test-dd         # double-double 精度（mpmath 对照，主机，秒级）
make test-libs       # 硬件库 gfx/key/sys/rand/codec/fx（帧缓冲像素 + 注入按键）
make test-stream     # 流式输出：边跑边打印、ESC/ON 中断、32 KB 环绕、nothread 退路
make test-input      # 输入路径的安全性质 + 触摸帧解码（含抖动过滤）
make test-leak       # 连跑 + 连续重启：固件堆不许增长
make test-host       # 第 1 层：主机差分（秒级）
make test-arm        # 第 2 层：32 位 ARM + qemu-arm（需要 gcc-arm-linux-gnueabihf、qemu-user）
make deploy          # 重新生成平铺部署目录
```


### 输出是流式的：边跑边显示、ESC 能停

`plua_entry` 起一个固件线程跑解释器，把运行状态交回启动器后**立刻返回**，
启动器于是这样循环（`calc/main.py: stream_run`）：

```
每 25 ms（固件 os_sleep，让出 CPU 给正在跑的程序）：
  · 读环形缓冲里 count 之后的字节 → 按行打印（moreprint 彩色）
  · 状态不是「运行中」→ 收工
  · 看键盘：**ESC**（启动器这一侧是 4，`hpprime.keyboard()` 的位掩码
            `mask & (1<<4)`，不碰 PPL）→ 把运行状态的 abort 置 1
  · 收到 Python 的 KeyboardInterrupt → 那是 **ON** 键（见下表），同样置 abort
  · 自己让出 CPU：每轮一次固件 `os_sleep`（走调试通道）
```

解释器那边 abort 由一个 **Lua 调试钩子**发现（`port/plua_hp.c`，每 20000 条 VM 指令
一次），它做的就是在 Lua 里 `error("interrupted")`：脚本于是像自己报错一样解开——
`pcall`、`xpcall`、to-be-closed 变量全都按语言的规矩走，退出码和错误信息照样经环形
缓冲回到启动器：

```
> main.run_file("spin.lua")
[*] running spin.lua ...
spinning
lua: .../spin.lua:4: interrupted
stack traceback:
        ...
[!] interrupted (ON)
```

### 解释器镜像只加载一次

primeLua 把镜像位置记在 `lua.slot`（magic `AULP`）里：下次会话**先核对再复用**: 核 `PLUARING` 指纹、镜像首字的 ELF 头、入口点第一条指令——对得上就直接用，
对不上就丢掉记录重新加载。记录里还存着**磁盘上`lua.elf` 的指纹**（大小 + 头/中/尾三段各 32 字节的校验和），于是**换了一版 lua.elf
不会再被常驻的旧镜像挡住**：指纹对不上就把旧镜像交还固件堆、重新加载新的那份。


### 控制台接口

```python
import main
main.selector()                 # 进入屏幕列表
main.menu()                     # 没有屏幕时的控制台菜单（stdin 提示符）
main.run_file("fib.lua")        # 跑一个脚本，输出经 moreprint 彩色打印
main.run()                      # 跑 DEFAULT_SCRIPT
main.run_source("print(1/3)")   # 跑一段源码（写临时文件后运行）
main.list_files()               # 列出发现的脚本
main.probe()                    # 自检，逐项 pcall
main.selftest("sieve.lua")      # 逐段检查整条加载链
main.unload()                   # 归还解释器镜像占用的内存，并作废 lua.slot
```

---

## 4. 已知限制

这些是**设备物理限制**导致的：

| 项 | 说明 |
|---|---|
| **输出只有最后 32 KB** | 环形缓冲大小所限；`for i=1,1e6 do print(i) end` 只能看到尾部。写得太快时两次轮询之间被覆盖掉的字节里，第一条可能是半行——环形缓冲存的是字节，没有行边界（`make test-stream` 把这条也钉住了） |
| **启动器绝不写死地址** | 真机上写一个硬编码地址可能直接复位整机：`0x30000000` 在模拟器里是应用内存起点，在真机上属于固件。selftest 原来第一步就往那里写 0 → 真机**一运行 selftest 立即重启**；现在改为"向固件 malloc 给我们的块写/读回"，`test_launcher.py` 里有一条断言：调试通道的每一次写都必须在固件堆范围内 |
| **运行时可以中断，但不能交互输入** | **ESC（或 ON）** 会中断正在跑的程序，但控制台本身不接受输入：脚本要键盘就用 `key.getkey()`；真正的交互式终端需要 Python 侧与程序同时读写屏幕，那是另一件事 |
| `os.clock()` **只走时、不走 CPU 时间** | 计算器只有一个进程，所以它给的是"解释器起来以后过了多少秒"。固件时钟不可用时返回 0 —— 不编 |
| `os.date/os.time` 依赖固件或启动器 | 先是固件实时钟（svc 0x100A5），其次启动器每次运行前从 PPL `Date`/`Time` 读的那个 epoch；两个都没有时返回 0（1970-01-01），不做假 |
| `os.getenv()` 恒返回 nil | 计算器没有环境块（因此 `LUA_INIT` 不会生效） |
| `os.execute` / `package.cpath` / `require` C 模块 | 无进程、无动态库；`package.cpath` 为空串，`os.tmpname` 固定在应用目录 |
| `io` 是固件文件 API | 路径为 `C:\DATA\...` 风格；**相对路径会被解释器解析到脚本所在的 app 目录**（固件自己的 cwd 不是那里，所以 `io.open("x","w")` 若不这样处理会报 No such file or directory）；无 `rename`；无 `io.popen` |
| `os.rename` 返回失败 | 固件没暴露对应服务 |
| 字符串哈希种子固定 | 为可复现性放弃了 Lua 的防哈希洪泛随机化（计算器上无关紧要） |
| 没有真正的 REPL | 逐行求值的 REPL 需要输入源；CLI 菜单里的 `= <lua>` 是可用的单行 REPL，多行用 `main.run_source()` |
| 图形/键盘 API | 已提供 `gfx`/`key`。图形程序**运行期间屏幕照常刷新** |
| 输出为 UTF-8 字节 | 计算器屏幕对非 ASCII 的显示取决于启动器字体支持 |

---

## 5. 目录结构

```
primeLua/
├── Makefile                     构建/测试/部署
├── README.md                    本文
├── api.md                       **Lua 侧 API 参考**
├── src/lua-5.4.7/               
├── port/                        primeLua 平台层
│   ├── plua_svc.s               固件 SVC + setjmp/longjmp + 入口
│   ├── plua_main.c              启动粘合、程序栈、exit 展开、RT_RET
│   ├── plua_run.c / .h          **跑在固件线程上**：svc 0x10000 起线程、
│   │                             运行状态（magic PLUARUN）、abort 字（1.0 新增）
│   ├── plua_format.c            C99 printf 引擎（精确十进制）
│   ├── plua_strtod.c            正确舍入的 strtod
│   ├── plua_file.c              FILE 层 + 环形缓冲 + 固件文件 API
│   ├── plua_libc.c              realloc/exit/errno/locale/ctype/strerror
│   ├── plua_math.c              标准名 → openlibm
│   ├── plua_time.c              64 位 time_t、gmtime/mktime/strftime、**固件实时钟**
│   ├── plua_hp.c                硬件库绑定（gfx/key/sys/rand/codec/fx/font/dd → Lua）
│   ├── plua_input.c / .h        **输入层**：get_event 槽位钩子、
│   │                             事件/触摸解析、槽位恢复与验证
│   ├── plua_font_montserrat_data.c  Montserrat 16/24/32 px 字模（generated：tools/make_font_montserrat.py）
│   ├── plua_font_cascadia_data.c    Cascadia Code Light 16/24/32 px 字模（generated：tools/make_font_cascadia.py）
│   ├── plua_font_big_data.c        钟表的 56 px 数字面（generated：tools/make_font_big.py，`make font-big`）
│   ├── plua_dd.c / plua_dd.h    double-double 算术与超越函数（新库 dd）
│   ├── plua_dd_NOTES.md         dd 的算法、实测精度与已知取舍
│   ├── plua_port.h              启动契约（struct plua_config）与内部声明
│   ├── include/                 遮蔽 newlib 的头文件
│   └── openlibm-extra/e_log2.c  primetcc 的 openlibm 子集缺的那个文件
├── calc/
│   ├── main.py                  计算器端启动器（MicroPython）
│   ├── timeprobe.py             真机时间源探针
│   └── timeprobe.log            2025-09-25 那次真机测量的原始输出
├── examples/                    示例脚本
├── tests/
│   ├── harness_lua.py           端到端差分
│   ├── test_format_host.c       printf 差分（主机 + ARM）
│   ├── test_strtod_host.c       strtod 差分（主机 + ARM）
│   ├── test_time_host.c         strftime 差分 + 真机时钟结构解码（主机 + ARM）
│   ├── emu_time.py              svc 0x100A5（模拟器侧的固件时钟）
│   ├── emu_fixes.py             
│   ├── test_launcher.py         启动器测试（mock MicroPython：拒绝绝对路径 + 屏幕列表 + 控制台菜单）
│   ├── test_hp_libs.py          硬件库过模拟器（帧缓冲像素 + 注入按键）
│   ├── test_input.py            输入路径的安全性质（跑完/跑挂都不许留下补丁）
│   ├── leak_check.py            连跑/重启不涨内存
│   ├── qemu_arm/run_arm_tests.py 在 qemu-arm 下跑上面三套
│   ├── scripts/*.lua            12 个差分脚本（hp_libs.lua 是设备专用，自检）
│   └── modules/pluamod.lua      require 用模块
└── tools/
    ├── fix_relocs.py            构建期把符号重定位折叠成 R_ARM_RELATIVE
    ├── make_font_montserrat.py  重新生成 Montserrat 字模
    ├── make_font_cascadia.py    重新生成 Cascadia 字模（离线栅格化 → port/）
    ├── gen_dd_expected.py       dd 的 mpmath 60 位期望值
    └── make_dist.py             打 dist/ 分发包

```
