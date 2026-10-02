# primeLua
## 把 Lua 5.4 移植到 HP Prime G1

> 跑的是**官方 Lua 5.4.7 解释器**，与桌面 `lua` 的差异只有那些**物理上无法相同**的部分，如没有进程、没有环境变量等

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
    · 从固件堆 malloc 程序栈（首选 128 KB，失败逐级减半；Lua 的递归下降解析器需要）
    · plua_begin(r0)：登记 config、初始化控制台
    · plua_main() → lua.c 的 main()（原封不动，只把符号名改成
      plua_lua_standalone_main）
    · 结束后写 "RT_RET:<n>" 到环形缓冲、归还栈、恢复所有寄存器、返回
         │
         ▼
      Lua 5.4.7 VM ── print/io.write ──► PLUARING 环形缓冲（32 KB）
                                          ▲
计算器端 main.py ── 轮询 ─────────────────┘  边跑边打印
```

**程序跑在固件线程上**：`plua_entry` 用固件的 `svc 0x10000` 起一个线程
（`sys_create_thread(plua_thread_entry, &config, 512 KB)`），把运行状态结构体的地址交回启动器后**立刻返回**。于是启动器在程序
还在跑的时候就拿回了控制权，可以一边读环形缓冲一边打印（§7.5），也可以在用户按 ESC
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
| `plua_run.c` | **运行方式**：svc 0x10000 起固件线程跑解释器、运行状态（magic `PLUARUN`）、中断（abort）字与调试钩子检查 | 这是**流式输出**的全部秘密（§7.5）：入口立刻返回，启动器边跑边读；线程栈由固件给（512 KB），所以同步模式里那个 malloc 的程序栈只在 `nothread` 下才用 |
| `plua_format.c` | **自己的 C99 printf 引擎** | 见 §4.1。`%.14g` 是 Lua 打印每个数字用的格式 |
| `plua_strtod.c` | **自己的正确舍入 strtod** | 见 §4.2。Lua 词法分析器用它解析每个数字字面量 |
| `plua_file.c` | FILE 层：stdout/stderr → 环形缓冲，文件 → 固件宽字符 API | 必须实现 `ungetc` 与 `freopen`：`luaL_loadfilex` 靠它们嗅探 `#`/BOM/字节码 |
| `plua_libc.c` | `realloc`（固件堆，与 primetcc 分配器头布局互通）、`exit/abort`、`errno`、`locale`、`ctype`、`strerror`（glibc 同字面）、`strtoul` | 固件堆只保证 4 字节对齐，而 ARM926 上未对齐 LDRD/STRD 会复位 → realloc 里要做对齐搬迁 |
| `plua_math.c` | 标准名 → openlibm 的 `hp_*` | 复用 primetcc 预编译的 `rt_math.o`（1 ulp），另补 openlibm 缺的 `log2`（`port/openlibm-extra/`）与 `copysign` |
| `plua_time.c` | `gmtime/localtime/mktime/strftime/difftime/clock` | `%c/%x/%X/%e` 对齐 glibc 的 C locale；**strftime 必须容忍 `fmt == s`**（Lua 的 `os.date` 就是这么调的） |
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

`make deploy` 会把部署目录**先整个删掉再重建**（它是构建产物，不是仓库）。自己的脚本请放`C:\DATA\`，别放这里。


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

解释器那边，abort 由一个 **Lua 调试钩子**发现（`port/plua_hp.c`，每 20000 条 VM 指令
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

### 解释器镜像只加载一次（`lua.slot`）

一台机器上连续跑几次，固件剩余内存是这样的（用户实测，**每跑一次少 0.4–2 MB**）：
`15386 KB → 13194 KB → 12808 KB`。原因是每次重新 `import main` 都会再往固件堆里
**加载一份 500 KB 的 lua.elf**，而 MicroPython 的模块对象随 `sys.modules` 一起
消失，旧镜像就再也没人释放了（primetcc 用 `tcc.slot` 解决的是同一个问题）。

primeLua 现在把镜像位置记在 `lua.slot`（magic `AULP`）里：下次会话**先核对再复用**
——核 `PLUARING` 指纹、镜像首字的 ELF 头、入口点第一条指令——对得上就直接用，
对不上（重启过、内存被固件重新分配过）就丢掉记录重新加载。记录里还存着**磁盘上
`lua.elf` 的指纹**（大小 + 头/中/尾三段各 32 字节的校验和），于是**换了一版 lua.elf
不会再被常驻的旧镜像挡住**：指纹对不上就把旧镜像交还固件堆、重新加载新的那份。


### 控制台接口

```python
import main
main.selector()                 # 进入屏幕列表（import main 默认就是这个）
main.menu()                     # 没有屏幕时的控制台菜单（stdin 提示符）
main.run_file("fib.lua")        # 跑一个脚本，输出经 moreprint 彩色打印
main.run()                      # 跑 DEFAULT_SCRIPT
main.run_source("print(1/3)")   # 跑一段源码（写临时文件后运行）
main.list_files()               # 列出发现的脚本
main.probe()                    # 38 项自检（含硬件库），逐项 pcall
main.selftest("sieve.lua")      # 逐段检查整条加载链
main.unload()                   # 归还解释器镜像占用的内存，并作废 lua.slot
```

---

## 4. 与桌面 Lua 的差异 / 已知限制

这些都是**设备物理限制**导致的，不是没做完：

| 项 | 说明 |
|---|---|
| **输出只有最后 32 KB** | 环形缓冲大小所限；`for i=1,1e6 do print(i) end` 只能看到尾部。写得太快时两次轮询之间被覆盖掉的字节里，第一条可能是半行——环形缓冲存的是字节，没有行边界（`make test-stream` 把这条也钉住了） |
| **启动器绝不写死地址** | 真机上写一个硬编码地址可能直接复位整机：`0x30000000` 在模拟器里是应用内存起点，在真机上属于固件。selftest 原来第一步就往那里写 0 → 真机**一运行 selftest 立即重启**；现在改为"向固件 malloc 给我们的块写/读回"，`test_launcher.py` 里有一条断言：调试通道的每一次写都必须在固件堆范围内 |
| **运行时可以中断，但不能交互输入** | **ESC（或 ON）** 会中断正在跑的程序（§7.5），但控制台本身不接受输入：脚本要键盘就用 `key.getkey()`；真正的交互式终端需要 Python 侧与程序同时读写屏幕，那是另一件事 |
| `os.clock()` **只走时、不走 CPU 时间** | 计算器只有一个进程，所以它给的是"解释器起来以后过了多少秒"（固件计数器现场标定，见 §4.4）。固件时钟不可用时返回 0 —— 不编 |
| `os.date/os.time` 依赖固件或启动器 | 先是固件实时钟（svc 0x100A5），其次启动器每次运行前从 PPL `Date`/`Time` 读的那个 epoch；两个都没有时返回 0（1970-01-01），不做假 |
| `os.getenv()` 恒返回 nil | 计算器没有环境块（因此 `LUA_INIT` 不会生效） |
| `os.execute` / `package.cpath` / `require` C 模块 | 无进程、无动态库；`package.cpath` 为空串，`os.tmpname` 固定在应用目录 |
| `io` 是固件文件 API | 路径为 `C:\DATA\...` 风格；**相对路径会被解释器解析到脚本所在的 app 目录**（固件自己的 cwd 不是那里，所以 `io.open("x","w")` 若不这样处理会报 No such file or directory）；无 `rename`；无 `io.popen` |
| `os.rename` 返回失败 | 固件没暴露对应服务 |
| 字符串哈希种子固定 | 见 §5：为可复现性放弃了 Lua 的防哈希洪泛随机化（计算器上无关紧要） |
| 没有真正的 REPL | 逐行求值的 REPL 需要输入源；CLI 菜单里的 `= <lua>` 是可用的单行 REPL，多行用 `main.run_source()` |
| 图形/键盘 API | 已提供 `gfx`/`key`（见 §10）。图形程序**运行期间屏幕照常刷新**（帧缓冲由 Lua 自己 blit，固件 LCD 一直在扫），输出现在也是边跑边显示（§7.5） |
| 输出为 UTF-8 字节 | 计算器屏幕对非 ASCII 的显示取决于启动器字体支持 |

---

## 5. 目录结构

```
primeLua/
├── Makefile                     构建/测试/部署（唯一入口）
├── README.md                    本文（移植、测试、真机排错）
├── api.md                       **Lua 侧 API 参考**（所有库：签名/常量/例子/限制）
├── src/lua-5.4.7/               上游 Lua 源码（未打任何补丁）
├── port/                        primeLua 平台层（移植的难点都在这里）
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
│   ├── timeprobe.py             真机时间源探针（怎么量出 svc 0x100A5 的，见 §4.4）
│   └── timeprobe.log            2025-09-25 那次真机测量的原始输出
├── examples/                    示例脚本（随部署目录一起拷）
├── tests/
│   ├── harness_lua.py           端到端差分（模拟器 vs 参考 lua，多进程）
│   ├── test_format_host.c       printf 差分（主机 + ARM）
│   ├── test_strtod_host.c       strtod 差分（主机 + ARM）
│   ├── test_time_host.c         strftime 差分 + 真机时钟结构解码（主机 + ARM）
│   ├── emu_time.py              svc 0x100A5（模拟器侧的固件时钟）
│   ├── emu_fixes.py             给 primetcc 模拟器的三个补丁（不修改 primetcc）
│   ├── test_launcher.py         启动器测试（mock MicroPython：拒绝绝对路径 + 屏幕列表 + 控制台菜单）
│   ├── test_hp_libs.py          硬件库过模拟器（帧缓冲像素 + 注入按键）
│   ├── test_input.py            输入路径的安全性质（跑完/跑挂都不许留下补丁）
│   ├── leak_check.py            连跑/重启不涨内存（lua.slot 复用镜像）
│   ├── qemu_arm/run_arm_tests.py 在 qemu-arm 下跑上面三套
│   ├── scripts/*.lua            12 个差分脚本（hp_libs.lua 是设备专用，自检）
│   └── modules/pluamod.lua      require 用模块
└── tools/
    ├── fix_relocs.py            构建期把符号重定位折叠成 R_ARM_RELATIVE
    ├── make_font_montserrat.py  重新生成 Montserrat 字模（primetcc 表 → port/）
    ├── make_font_cascadia.py    重新生成 Cascadia 字模（离线栅格化 → port/）
    ├── make_font_big.py         重新生成钟表数字面（56 px / `BIGSIZE=n`）
    ├── gen_dd_expected.py       dd 的 mpmath 60 位期望值
    └── make_dist.py             打 dist/ 分发包

../primeLua.hpappdir/            平铺部署目录（拷到计算器 C:\DATA\）
../_attic/primeLua-ti/           ti 库（TI-Nspire 兼容层），这一版不做，见 §1
../_attic/primeLua-touch/        原始多点触摸帧的测试与模拟器支持，这一版不带，见 §12
```

---

## 6. 硬件库（`gfx` / `key` / `sys` / `rand` / `codec` / `fx`）

```lua
local gfx, key = gfx, key

gfx.init()                            -- 建默认全屏 GROB 并接管绘制目标
gfx.clear(gfx.BLACK)
gfx.fillrect(10, 10, 100, 40, gfx.RED)
gfx.line(0, 0, 319, 239, gfx.WHITE)
gfx.circle(160, 120, 40, gfx.YELLOW)
gfx.text(4, 4, "hello", gfx.WHITE)
gfx.blit()                            -- 后缓冲整屏送到屏幕（一次拷贝，无闪烁）

local g = gfx.newgrob(64, 48)         -- 离屏位图 = 双缓冲
gfx.select(g)  gfx.clear(gfx.CYAN)    -- 画到 GROB 上
gfx.select()                          -- 回到默认后缓冲
gfx.blit(g, 240, 160)                 -- 把 GROB 拷到当前目标
gfx.blit()                            -- 再整屏呈现
gfx.freegrob(g)

key.install()                         -- 挂上按键钩子（默认就是开的；退出时自动还原固件槽）
local kc = key.getkey()               -- 设备键码；无按键时是 nil（不是 0）
local ev = key.event()                -- 键盘/触摸都在这条队里（触摸带 x y dx dy）
local down, x, y = key.touch()        -- 当前有没有手指按着、按在哪
key.remove()
```

| 模块 | 主要内容 |
|---|---|
| `gfx` | `init/width/height/rgb/clear/pixel/getpixel/line/hline/vline/rect/fillrect/circle/fillcircle/triangle/filltriangle/text/textbg/textwidth/blit/blitpixels/newgrob/freegrob/select/grobwidth/grobheight`，颜色常量 `BLACK…GRAY`、`WIDTH`/`HEIGHT` |
| `key` | `install/remove/hooked/hookstats/event/getkey/gettouch/touch/sleep`，按键常量见下面的**键码表**；触摸的语义（phase 1/2/8、dx/dy、抖动过滤）见 §7 的输入小节 |
| `sys` | `memory`（剩余内存）、`maxalloc`、`sleep`、**`time()`**（固件实时钟的 Unix 秒，没有则 nil）、**`clock()`**（解释器起来以后过了多少秒）、**`log(text)`**（写进持久日志 `plua.log`，复位也不丢）、**`version()`**（解释器 API 版本，见下） |
| `rand` | `seed/u32/range/noise` |
| `codec` | `crc32/adler32/hex/unhex/base64/unbase64` |
| `fx` | Q16.16 定点（无 FPU 时更快）：`int/trunc/round/todouble/fromdouble/neg/abs/add/sub/mul/div/lerp/sqrt/sin/cos/tan/atan2` | — |
| **`font`** | 真正的字体渲染：`prop mono cascadia height width advance draw drawbg` | 见下面的字体小节 |
| **`dd`** | double-double（≈31 位十进制）**新库**，`math` 一行没动：构造 `dd(x)`、运算符 `+ - * / ^ == < <=`、`pi/e/ln2/ln10`、`sqrt exp log log10 sin cos tan asin acos atan atan2 pow fmod hypot abs floor ceil neg`、`tostring(x[,位数]) tonumber parts isnan isinf new` | 见下面的 dd 小节 |

#### 绘制模型（`gfx.init()` 之后就在屏幕上了）

| 调用 | 作用 |
|---|---|
| `gfx.init()` | 初始化（读固件 LCD 结构），**绘制目标＝屏幕本身**：画了就能看见，不需要 blit，也**不分配** 307 KB 后缓冲 |
| `gfx.select()` | 按需**分配**默认后缓冲（307 KB），并用**当前屏幕内容做种子**（所以第一次 `blit()` 不会把之前画的东西抹掉）；之后画到缓冲、`blit()` 上屏 |
| `gfx.select(false)` | 回到直接画屏幕 |
| `gfx.select(g)` | 画到某个 GROB（离屏合成） |
| `gfx.blit()` | 把默认后缓冲整屏拷贝上屏（没有后缓冲时是空操作——画本来就在屏幕上） |
| `gfx.blit(g, x, y)` | 把 GROB 拷到**当前绘制目标**（不切目标） |
| `gfx.freebackbuf()` | 释放后缓冲（307 KB 还给固件堆），回到直接画屏幕 |
| `gfx.target()` | `"screen"` / `"default"` / `"grob"`：现在画到哪儿 |
| `gfx.fb()` | 固件 LCD 结构里的 `framebuffer, w, h`——排错用（整屏绘制出事时第一个要问的就是这个地址对不对） |

后缓冲**按需分配**是故意的：307 KB 在固件堆里不是小数目，而只画屏幕的脚本根本不需要它；
primetcc 的注释里也记着这块固件对"大块 alloc/free"最不可靠。写回缓冲时目标绝不会悬空：
`freegrob()` 释放的正是当前目标时会先切回默认后缓冲（否则下一次 `clear()` 会写进已回收的内存）。

`key.getkey([wait_ms])` 返回**固件的设备键码**；没有按键时返回 **`nil`**（会跳过
按键释放事件，等 `wait_ms` 毫秒后仍无键也返回 nil）。所以取键循环写成
`local kc = key.getkey(); if not kc then ... end`——写成 `== 0` 会永远不成立。
`key.gettouch()` 读触摸事件（坐标系与 `gfx` 相同）。

#### 键码：项目里有**两张不同的表**

| 键 | **设备键码**（Lua `key.getkey()` 返回、`key.*` 常量） | PPL `GETKEY` / `hpprime.keyboard()` 位号（Python 侧用） |
|---|---|---|
| esc | `0x01` (1) | 4 |
| ← 左 | `0x02` (2) | 7 |
| ↑ 上 | `0x03` (3) | 2 |
| → 右 | `0x04` (4) | 8 |
| ↓ 下 | `0x05` (5) | 12 |
| 退格 | `0x0C` (12) | 19 |
| 回车 | `0x0D` (13) | 30 |
| 空格 | `0x20` (32) | 49 |
| ON | `0x83` | **46**（系统中断键，见下） |
| APPS / SYMB / PLOT / NUM / VIEW / CAS / MENU | `0xB1/0x91/0xB2/0xB3/0xB4/0xB5/0x93` | 0/1/6/11/9/10/13 |



**触摸在这台机器上长什么样**（`make test-input` 用注入的触点钉死）：

```lua
key.install()
local ev = key.event()          -- {type="touch", phase=1, x=100, y=50, dx=0, dy=0}
                                -- phase: 1 按下, 2 移动, 8 抬起（固件自己的编号）
local down, x, y = key.touch()  -- 当前有没有手指按着，以及最近一次被接受的位置
```

#### 文本：5×8 点阵 与 `font` 库

两种文字，各有用处：

| | `gfx.text` / `gfx.textbg` | `font` 库（真字体） |
|---|---|---|
| 字形 | hp_gfx 内置 5×8 点阵，640 B | primetcc 引擎 + **Montserrat（比例）/ Cascadia Code Light（等宽）**，**各 16 / 24 / 32 px**；另有钟表用的 **`font.big()` 56 px 数字面**。全部 4-bit 抗锯齿 |
| 占用 | 0 B（已在镜像里） | 引擎 892 B + 两族三个尺寸的字模 ≈ **43 KB**（占镜像 .rodata 的一半）；`font.big()` 另加 6.6 KB |
| 宽度 | 固定 6 px/字符 | 真实宽度（`font.width`），比例字体按字形推进；等宽字体每格同宽 |
| 接口 | `gfx.text(x,y,s,c)`、`gfx.textbg(x,y,s,fg,bg)`、`gfx.textwidth(s)` | `font.prop()/font.mono()/font.cascadia()/font.big()` 取句柄，`font.height/w/handle`、`font.width(h,s)`、`font.advance(h,ch)`、`font.draw(h,x,y,s,c)`、`font.drawbg(h,x,y,s,fg,bg)` |

```lua
local h  = font.prop()                    -- Montserrat，或 font.mono() / font.cascadia()
font.drawbg(h, 8, 40, "Mandelbrot", gfx.WHITE, gfx.BLUE)
local w, lh = font.width(h, "Mandelbrot"), font.height(h)   -- 21 px 行高

local mid = font.mono(32)                 -- 同一族的最大档：行高 38 px，格子 19 px
font.draw(mid, 20, 100, "12:34:56", gfx.CYAN)   -- 152 px 宽

local big = font.big()                    -- 钟表的数字面：行高 66 px，格子 33 px
font.draw(big, 28, 73, "12:34:56", gfx.CYAN)    -- 264 px 宽，占满 320 px 的屏
```

`font.prop(20)` / `font.mono(14)` 这类尺寸参数**取最近的一档**（primetcc 引擎自己的
规则），所以 `prop(20)` 画 24 px、`mono(14)` 画 16 px，`prop(64)` 落到最大的 32 px ——
照着 primetcc 四档写的脚本照样能跑。

#### `font.big()`：钟表用的 56 px 数字面（不是第三套正文字体）

「要大字」和「不能糊」是矛盾的：把 32 px 的面画进 GROB 再放大 3 倍，抗锯齿边缘也
一起放大 —— 每个 1 px 的灰边变成 2~3 px 的灰带，真机上肉眼可见地糊。所以大数字
**按它要显示的尺寸重新栅格化**：

* `port/plua_font_big_data.c`：Cascadia Code Light 原生 56 px，只有
  `'-' '.' '/' '0'-'9' ':'`（0x2D..0x3A）这 14 个字形，**6.6 KB**（全 ASCII 要 ~40 KB）；
  格子 33 px、行高 66 px，`"HH:MM:SS"` 正好 264 px；
* 生成器 `tools/make_font_big.py`（`make font-big` / `make font-big BIGSIZE=64`），
  走的是同一个 `fontgen2.py`，所以 nibble 顺序、行补齐、等宽居中和其他字面完全一致；
* **它只认这 14 个字**：画别的字符**什么都不画**（`hp_fonts.c` 直接跳过，不是画方块）。
  正文字还是用 16/24/32 px 的两族。
* `examples/clock.lua` 用它画时间；`make test-libs` 里有一条专门验证"字母用这个面画
  出来是 0 个像素"，另一条把 `"12:34"` 的 1868 个像素与生成数据逐像素对齐。

**两族是怎么来的（`tools/make_font_montserrat.py` / `make_font_cascadia.py`）**：
primetcc 的 `rt/hp_fonts_data.c` 把 8 档字面（4 尺寸 × 比例/等宽）放在一个翻译单元里，
取字面的 `hp_font_prop_data(i)` 用的是**运行期下标**，链接器没法按需丢弃，一链就是
~70 KB。所以 primeLua 自己生成两份字模文件：

* **Montserrat**：从 primetcc 的表里逐字提取 16/24/32 三档 → `port/plua_font_montserrat_data.c`；
* **Cascadia Code Light**：primetcc 现在提交的那份等宽表里是**别的字体**，所以这份是
  用同一个离线栅格化器（`rt/tools/fontgen2.py` + `CascadiaCode-Light.ttf`）现栅格化出
  16/24/32 三档 → `port/plua_font_cascadia_data.c`。

**primetcc 本体一行没改**（栅格化器是只读调用，输出取完即弃），引擎 `rt/hp_fonts.c`
照原样编译进来，字面访问器（`hp_font_prop_data()` / `hp_font_mono_data()`）由这两份
生成文件提供，所以 `font.mono()` 就是 Cascadia，`font.cascadia()` 是同一个字面的显式
名字。生成脚本可重复执行，输出进仓库，正常构建既不需要 primetcc 的字体数据也不需要
PIL。

**验证**：模拟器里把 `font.draw` 的结果与"从字模数据反推的期望像素"逐像素比对
（含 4-bit alpha 混合的精确值）——16 px 的 `prop`/`mono` 各 104/124 个像素、24 px 的
`prop` 188 个、32 px 的 `mono` 373 个、56 px 的 `big` 1868 个，**全部完全一致**；
`font.drawbg` 的背景方块尺寸、颜色、边界外不着色也都对得上；脚本里还钉死了各档行高
（prop 21/31/40、mono 19/29/38、big 66）、`font.mono()` 与 `font.cascadia()` 在每一档
上指标相同、等宽格的 `":"` 与 `"1"` 推进量相等（钟表数字不会跳）、以及大字体面
画字母 0 像素。指标也对得出 primetcc 注释里的 16 px 值（`"Hello, World!"` = 101 px）。

#### `dd`：31 位十进制的高精度库（新库，不动 `math`）

```lua
local dd = dd
local x = dd(2) * dd.pi()                  -- 2π，31 位
print(x)                                   -- 6.28318530717958647692528676655901e+00
print(dd.sqrt(dd(2)), dd.sin(dd(1)))       -- 都是 33 位
print(dd.tostring(dd.pi(), 20))            -- 也可以只要 20 位
print(dd("3.14159265358979323846264338327950288"))   -- 从长字符串解析
print(dd(9007199254740993) - dd(9007199254740992))   -- 1（double 会给 0）
local hi, lo = dd.parts(dd(1)/dd(3))       -- 精确的两个 double
print(dd(1) < dd(2), dd(3) == dd(3))       -- 比较运算符
print(dd.pi():sqrt())                      -- 方法写法也可以
```

`dd` 是**独立的库**：Lua 的 `math` 与数字类型保持与桌面 Lua 逐位一致（差分测试依赖这一点），
高精度放在自己的命名空间里。技术细节、实测精度表与实现坑（Cody-Waite 归约、dd 不能按
分量除以整数、打印必须两个 double 都满宽输出）都写在 **`port/plua_dd_NOTES.md`**，
主机端对照测试是 `make test-dd`（mpmath 60 位真值，680 个函数用例 + 10 个字符串往返）。

实测（dd-ulp，1 ulp = 2^-106 ≈ 1.08e-32 相对误差）：sin 4.1 / cos 2.2 / tan 3.3 /
asin 4.6 / acos 2.4 / atan 1.5 / atan2 4.3 / **exp 45.6**（|x|>300 时约 30 位）/
log 2.7 / log10 3.7 / sqrt 1.3 / pow 10.0 / hypot 1.2，常数 π/e/ln2/ln10 精确。
参数归约用三字 Cody-Waite，所以 sin/cos 在 |x| 到 1e6 仍保持精度。

已知取舍：`dd("0.1") + dd("0.2") == dd("0.3")` 是 **false**——`dd("0.1")` 是*十进制* 0.1
（比 double 的 0.1 差 5.55e-18，这正是它的意义），两个值各带约 1e-32 舍入，和比 0.3 低
约 2e-32；打印出来能看到 `2.99999999999999999999999999999998e-01`。


#### 版本号：确认设备上的 lua.elf 是不是新的

`sys.version()`（`PLUA_VERSION`，手写常量，Lua 可见 API 一变就 +1）会由解释器在每次启动时
写进 `plua.log`（`interp: 0.5`），列表的 `log` 行能看到；诊断脚本第一行也会记它。
**换脚本一定要连 `lua.elf` 一起换**——用新的 `.lua` 配旧的 `lua.elf`，症状是
`attempt to call a nil value (field 'target')` 这类"字段不存在"，看着像脚本 bug，
其实是设备上跑着旧解释器。

#### 文件名与模式：固件只认 `"rb"` / `"wb+"`

固件的 `fopen` 只可靠支持 `"rb"` 和 `"wb+"`（primetcc 的 diag 日志就是这么写的，并且
记录了 `"ab"` 不可靠）。Lua 和 C 库说的是普通 C 模式，**原样透传在模拟器里没事，在真机上
`io.open(name, "w")` 直接返回 nil**——诊断脚本的日志文件因此在计算器上一个都没出现。
现在 `port/plua_fwmode.c` 做翻译（`r*`→`rb`；`w*`→`wb+`；`a*`→`wb+` 并把旧内容先写回再
定位到末尾；`r+`→`wb+` 保留内容），`make test-modes` 用 15 条断言把它钉死。

---

## 7. 常见疑问

**为什么 `print(math.sin(1))` 只有 13 位小数，Python 有 16 位？**

不是精度问题，是**打印格式**：Lua 5.4 的 `print`/`tostring` 用 `%.14g`
（`luaconf.h` 的 `LUA_NUMBER_FMT`，桌面 Lua 一样），即 14 位有效数字；Python 的
`repr` 打印"最短可往返"表示。值本身是满精度 double：

```lua
print(string.format("%.17g", math.sin(1)))   --> 0.8414709848078965
print(string.format("%a",    math.sin(1)))   --> 0x1.aed548f090ceep-1
```
