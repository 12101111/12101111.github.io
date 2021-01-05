+++
title = "什么, 你不喜欢main函数?"
date = 2020-12-14
[taxonomies]
categories = ["Rust"]
tags = ["Rust","Linux"]
+++

要想编译一个rust的可执行程序, 必须在crate顶层定义[`main`函数](https://doc.rust-lang.org/reference/crates-and-source-files.html#main-functions). `main`函数没有参数, 没有trait或生命周期修饰, 返回值为`()`(或者不写返回值类型), `!`, `Result<(), E> where E: Error`或其他实现[std::process::Termination](https://doc.rust-lang.org/nightly/std/process/trait.Termination.html) Trait的类型.

虽然大多数情况下rust的用户只需要老实的写一个`main`函数就能让他们的程序跑起来了,但是总有一些奇怪的情况使得`bin`类型的crate没有rust要求的`main`函数`.

## main太难听了

有一种情况是用户单纯的不喜欢`main`这个名字, 因此希望使用其他的函数名. 将属性[`#[main]`](https://github.com/rust-lang/rust/issues/29634)标准在希望成为入口点的函数上, 该函数就代替main函数的功能了, 该函数的签名需要满足rust中`main`函数的要求. 这是一个**已经被废弃**且**不稳定**的功能, 由于该功能实在没有什么价值且用户极少, 因此被弃用了, 不会再稳定.

## 谁动了我的main

但`main`函数真的是程序的入口点吗, 并不是. 无论是rust的main函数还是C的main函数都不是程序的入口点, 在过去版本的rust中, 如果设置了`RUST_BACKTRACE=1`的环境变量, panic发生后默认输出的backtrace会包含很多无用的信息, 在crate的main函数之下, 还有一些函数(现在的rustc版本编译的程序可以通过`RUST_BACKTRACE=full`输出同样的信息):

- `std::rt::lang_start::{{closure}}`
- `std::rt::lang_start_internal::{{closure}}`
- `std::panicking::try::do_call`
- `std::panicking::try`
- `std::panic::catch_unwind`
- `std::rt::lang_start_internal`
- `std::rt::lang_start`
- `main`
- `__libc_start_main`
- `_start`

这些代码位于文档中隐藏的[`std::rt`](https://github.com/rust-lang/rust/blob/master/library/std/src/rt.rs)模块, 这一模块设置了栈溢出的检测, 设置了线程的名称, 存储了main参数的位置(因此rust的main没有参数, 而命令行参数则是从`std::env`中获取), 并运行crate定义的main函数, 当其panic时则输出backtrace.

从启动流程上, Linux是`_start -> __libc_start_main -> main -> std::rt::lang_start -> ... -> crate::main`, 其中`_start`是由libc的`crt1.o`提供的, 并通过调用`__libc_start_main`来调用用户的`main`函数.Windows类似, 但是msvc和mingw64的crt(C语言运行时)逻辑各不相同. 共同点是crt都会调用`main`, 这个C语言规定的用户程序入口点(虽然在Windows上有`main`和`WinMain`的区分, 但是所有rust程序都使用main作为入口点).
更多有关C运行时的信息参见[`rustc`所用到的所有crt目标文件的表格](https://github.com/rust-lang/rust/blob/master/compiler/rustc_target/src/spec/crt_objects.rs), [Gentoo开发者对crt目标文件的解释](https://dev.gentoo.org/~vapier/crt.txt), [GCC对crt的解释](https://gcc.gnu.org/onlinedocs/gccint/Initialization.html)

那么rust程序中, crt要调用的`main`函数是从哪里来的呢, 因为rustc在一些目标不需要安装C编译器即可工作, 那么肯定不是来自于C代码. 先看一下`hello world`的rust程序的main函数反汇编长什么样子.

```asm
0000000000001260 <main>:
    1260:	48 83 ec 18          	sub    $0x18,%rsp
    1264:	8a 05 a6 0d 00 00    	mov    0xda6(%rip),%al        # 2010 <__rustc_debug_gdb_scripts_section__>
    126a:	48 63 cf             	movslq %edi,%rcx
    126d:	48 8d 3d ac ff ff ff 	lea    -0x54(%rip),%rdi        # 1220 <_ZN5hello4main17h5d50f6a5814ba113E>
    1274:	48 89 74 24 10       	mov    %rsi,0x10(%rsp)
    1279:	48 89 ce             	mov    %rcx,%rsi
    127c:	48 8b 54 24 10       	mov    0x10(%rsp),%rdx
    1281:	88 44 24 0f          	mov    %al,0xf(%rsp)
    1285:	e8 16 00 00 00       	callq  12a0 <_ZN3std2rt10lang_start17hfc7218ca4db8cb26E>
    128a:	48 83 c4 18          	add    $0x18,%rsp
    128e:	c3                   	retq
    128f:	90                   	nop
```

可以看到, 除了诡异的__rustc_debug_gdb_scripts_section__地址被放到栈上(0xf(%rsp))之外, C的main函数将rust crate的main(`hello::main`)函数地址作为第一个参数(%rdi), 原本的第一个参数argc(%edi)先备份到%rcx, 然后作为第二个参数(%rsi), 原本的第二个参数argv(%rsi)先备份到栈上(0x10(%rsp)), 然后再作为第三个参数(%rdx), 最后调用了`std::rt::lang_start`.这正好对应了`std::rt::lang_start`的签名:

```rust
#[lang = "start"]
fn lang_start(
    main: fn() -> T,
    argc: isize,
    argv: *const *const u8,
) -> isize;
```

但实际上这个简单的函数并没有对应的C代码, main这个符号和代码是[动态生成的](https://github.com/rust-lang/rust/blob/331e74014a0a2bf18aae7f02495f97958bf9767d/compiler/rustc_codegen_ssa/src/base.rs#L361).在生成了main函数后, rustc调用链接器, 链接到libc和c运行时目标文件.我们称这个自动生成并且直接调用rust函数的main函数为main wrapper.

## 我要我的main

处于某些目的或某些场景, 我们可能不想用`std::rt::lang_start`启动`main`函数, 这时候可以使用一个不稳定的功能: [`#[start]`](https://github.com/rust-lang/rust/issues/29633). 这个属性指定一个签名为`fn (argc: isize, argv: *const *const u8) -> isize`的函数为程序的入口点.如下所示:

```rust
#![feature(start)]

#[start]
fn rustmain(argc: isize, argv: *const *const u8) -> isize {
    println!("Hello, world!");
    3
}
```

注意到这个函数的函数名可以是任意的, 并且不需要使用`extern "C" fn`,但是签名目前必须为`fn (argc: isize, argv: *const *const u8) -> isize`(之所以是"目前是", 因为有人认为该签名不应为C语言的main函数签名).生成的代码反汇编如下:

```asm
0000000000001170 <_ZN5hello8rustmain17hffba4a0cd1d51340E>:
    1170:	48 83 ec 48          	sub    $0x48,%rsp
    1174:	48 8d 05 6d 2c 00 00 	lea    0x2c6d(%rip),%rax        # 3de8 <__do_global_dtors_aux_fini_array_entry+0x8>
    117b:	48 8d 0d 8e 0e 00 00 	lea    0xe8e(%rip),%rcx        # 2010 <__rustc_debug_gdb_scripts_section__>
    1182:	31 d2                	xor    %edx,%edx
    1184:	41 89 d0             	mov    %edx,%r8d
    1187:	48 89 7c 24 38       	mov    %rdi,0x38(%rsp)
    118c:	48 89 74 24 40       	mov    %rsi,0x40(%rsp)
    1191:	48 8d 7c 24 08       	lea    0x8(%rsp),%rdi
    1196:	48 89 c6             	mov    %rax,%rsi
    1199:	ba 01 00 00 00       	mov    $0x1,%edx
    119e:	e8 7d ff ff ff       	callq  1120 <_ZN4core3fmt9Arguments6new_v117ha823aa65b8f1952aE>
    11a3:	48 8d 7c 24 08       	lea    0x8(%rsp),%rdi
    11a8:	ff 15 3a 2e 00 00    	callq  *0x2e3a(%rip)        # 3fe8 <_ZN3std2io5stdio6_print17hcf65f36aa62c14c4E>
    11ae:	b8 03 00 00 00       	mov    $0x3,%eax
    11b3:	48 83 c4 48          	add    $0x48,%rsp
    11b7:	c3                   	retq
    11b8:	0f 1f 84 00 00 00 00 	nopl   0x0(%rax,%rax,1)
    11bf:	00

00000000000011c0 <main>:
    11c0:	50                   	push   %rax
    11c1:	8a 05 49 0e 00 00    	mov    0xe49(%rip),%al        # 2010 <__rustc_debug_gdb_scripts_section__>
    11c7:	48 63 ff             	movslq %edi,%rdi
    11ca:	88 44 24 07          	mov    %al,0x7(%rsp)
    11ce:	e8 9d ff ff ff       	callq  1170 <_ZN5hello8rustmain17hffba4a0cd1d51340E>
    11d3:	59                   	pop    %rcx
    11d4:	c3                   	retq
    11d5:	66 2e 0f 1f 84 00 00 	nopw   %cs:0x0(%rax,%rax,1)
    11dc:	00 00 00
    11df:	90                   	nop
```

可以看到`#[start]`仍然会生成main wrapper, 其中直接调用了`#[start]`指定的函数, rustc没有报错`main function not found in crate`, 也没有使用`libstd`中的`std::rt::lang_start`的逻辑, 因此`#[start]`的优先级高于通常的main函数查找.

需要注意的是, 使用`#[start]`启动的程序将无法使用`std::env::args`获得程序的命令行参数,而是需要手动处理`argc`和`argv`, 读取`argv`指向的的C风格字符串数组, 就像C语言一样.

## no_std

[`#![no_std]`](https://doc.rust-lang.org/reference/crates-and-source-files.html#preludes-and-no_std)使得crate默认不链接到[`libstd`](https://doc.rust-lang.org/std/index.html), 不插入libstd的[`prelude`](https://doc.rust-lang.org/std/prelude/index.html), 而是使用[`libcore`](https://doc.rust-lang.org/core/index.html)的[`prelude`](https://doc.rust-lang.org/core/prelude/v1/index.html). 由于`libcore`只定义了语言无关的项目, 因此main函数这类`libstd`提供的功能就不存在了,
这样就可以在`std`没有提供支持的非标准环境编写rust程序, 例如嵌入式和wasm开发.

通常只使用`#![no_std]`会得到一些编译错误:

### error: language item required, but not found: `eh_personality`

这需要在Cargo.toml中设置, 关闭panic后的栈回溯:

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

当然, 指定一个空的函数也可以

```rust
#[lang = "eh_personality"]
fn my_eh_personality() {}
```

### error: `#[panic_handler]` function required, but not found

这需要添加一个函数, picnic时会指向该函数. 在这里我们使程序死循环:

```rust
#[panic_handler]
fn my_panic_handler(_: &core::panic::PanicInfo) -> ! {
    loop {}
}
```

也可以使进程终止:

```rust
#![feature(core_intrinsics)]

#[panic_handler]
fn my_panic_handler(_: &core::panic::PanicInfo) -> ! {
    core::intrinsics::abort()
}
```

这样panic时程序就会直接退出: `illegal hardware instruction (core dumped)`, 实际上`core::intrinsics::abort()`会被编译为不存在的`ud2`指令.

如果该程序不在操作系统中执行, 而是在裸机上执行, 这个不存在的指令通常会导致CPU重置, 如果设置了中断处理程序, 则会执行该程序.

### error: requires `start` lang_item

这里要求提供一个`#[lang = "start"]`函数, 根据前面的知识, `#[start]`标注的函数优先级高于通常的main函数查找裸机, 因此可以使用`#[start]`标注main函数来屏蔽这个错误.

在Linux上运行`no_std`的程序, 和操作系统交互最方便的方式就是`libc` crate了, `libc`具有`no_std`模式, 可以在这里使用.将`libc = { version = "0.2", default-features = false }`加入`Cargo.toml`中.一个简单的`no_std`+`libc`的"hello world"长这个样子:

```rust
#![no_std]
#![feature(lang_items)]
#![feature(start)]
#![feature(core_intrinsics)]

#[link(name = "c")]
extern "C" {}

#[start]
fn rustmain(argc: isize, argv: *const *const u8) -> isize {
    unsafe {
        libc::puts(b"Hello World!\0" as *const u8 as *const i8);
    }
    //panic!();
    42
}

#[lang = "eh_personality"]
fn my_eh_personality() {}

#[panic_handler]
fn my_panic_handler(_: &core::panic::PanicInfo) -> ! {
    core::intrinsics::abort()
}
```

注意, 有些时候(例如在musl系统上)rustc会找不到`libc`的库, 我们使用`#[link(name = "c")] extern "C" {}`强制链接libc.

## 什么是`start` lang_item, 和`#[start]`有什么区别

`#[lang = "start"]` 指定另一个函数作为`std::rt::lang_start`的替代.

其签名为

```rust
#[lang = "start"]
fn lang_start(
    main: fn() -> T,
    argc: isize,
    argv: *const *const u8,
) -> isize;
```

其第一个参数为`main`函数的返回值, 在过去必须为`()`, 但rust现在的main函数支持一些其他的返回值类型.在文章开头说过, 在`std`中这些类型需要实现`std::process::Termination`.注意我们在`no_std`下并不能访问到`std::process::Termination`, 但是这其实是`rustc`提供的另一个lang_item, `#[lang = "termination"]` .用其标记一个Trait, 那么实现这个Trait的类型就可以作为`main`函数的返回值.

由于lang item只能指定一次, 因此必须使用no_std.

一个使用了`#[lang = "start"]`的hello world如下, 这里我们定义一个叫`Return`的trait, 并用`#[lang = "termination"]`标记, 这样在这个程序中, main函数的签名就必须是实现了`Return`的类型. 我们为isize实现该trait.

```rust
#![no_std]
#![feature(lang_items)]

#[lang = "termination"]
trait Return{}

impl Return for isize{}

#[lang = "start"]
fn start(
    main: fn() -> isize,
    argc: isize,
    argv: *const *const u8,
) -> isize {
    main()
}

fn main() -> isize{
    unsafe {
        libc::puts(b"Hello World!\0" as *const u8 as *const i8);
    }
    1
}

#[panic_handler]
fn my_panic_handler(_: &core::panic::PanicInfo) -> ! {
    loop {}
}
```

可以看到`start` lang_item, 和`#[start]`的区别在于, 生成main wrapper时会优先使用`#[start]`标注的函数, 如果存在`#[start]`, 则不再使用`start` lang_item标注的函数.`start` lang_item标注的函数具有固定的签名, 其目的在于执行用户crate中的`main`函数, 并为其初始化`std::env::args`等功能.`start` lang_item起作用时, rustc会按照正常的逻辑寻找用户crate里的`main`函数, 或者是`#[main]`标注的函数.

## 最终方案: no_main

`#![no_main]` 实际上是广大`no_std`用户用的最广的解决方案, 因为该方案并不像lang_item一样高度不稳定, 这是一个稳定的功能(虽然许多no_std的库还是需要nightly编译器开feature).

no_main会完全跳过main wrapper的生成, 用户需要自行提供`main`的符号, 否则链接器会链接失败. 与`cdylib`不同的是, 使用`no_main`会使链接器生成二进制程序而不是库.在一些目标, 例如uefi下, 我们并不使用`main`作为入口点, 而是使用

```rust
#[no_mangle]
extern "efiapi" efi_main fn(
    handle: uefi::Handle, 
    st: uefi::table::SystemTable<uefi::table::Boot>
) -> uefi::Status
```

这时我们就需要自行提供了符号`efi_main`供链接器使用.

将前面的例子稍作修改,就可以得到链接libc的`no_main`程序.

```rust
#![no_std]
#![no_main]
#![feature(lang_items)]
#![feature(core_intrinsics)]
extern crate libc;
#[link(name = "c")]
extern "C" {}

#[no_mangle]
extern "C" fn main(argc: isize, argv: *const *const u8) -> isize {
    unsafe {
        libc::puts(b"Hello World!\0" as *const u8 as *const i8);
    }
    //panic!();
    42
}

#[lang = "eh_personality"]
fn my_eh_personality() {}

#[panic_handler]
fn my_panic_handler(_: &core::panic::PanicInfo) -> ! {
    core::intrinsics::abort()
}
```

再进行反汇编, 这里的main就是我们提供提供的main函数, 而不是rustc生成的wrapper了.

```asm
0000000000001720 <main>:
    1720: 48 83 ec 18                  	subq	$24, %rsp
    1724: 48 89 7c 24 08               	movq	%rdi, 8(%rsp)
    1729: 48 89 74 24 10               	movq	%rsi, 16(%rsp)
    172e: 48 8d 3d 43 ee ff ff         	leaq	-4541(%rip), %rdi  # 578 <puts+0x578>
    1735: ff 15 35 12 00 00            	callq	*4661(%rip)  # 2970 <puts+0x2970>
    173b: b8 2a 00 00 00               	movl	$42, %eax
    1740: 48 83 c4 18                  	addq	$24, %rsp
    1744: c3                           	retq
```

当然, 这种方式对main的签名和ABI完全不做限制, 因此这种程序也可以编译,但是不能运行

```rust
#![no_main]

#[no_mangle]
pub fn main(args: Vec<String>) {
    for arg in args {
        println!("{}", arg);
    }
}
```

因此一些no_std库提供了宏用于阻止不正确的签名编译成功.例如`cortex-m-rt`使用`#[entry]`标记入口点

```rust
#[entry]
fn main() -> ! {
    loop {
        /* .. */
    }
}
```

正因为`#[start]`无法为最大的`no_std`用户: 嵌入式和wasm开发提供合适的支持,因此该功能仍然需要讨论和开发才会稳定.而`start` lang_item只会被std开发者用到, 不可能稳定.

### 我也不想要C语言的main(aka. main符号)

由于libc和crt是绑定的, 因此main符号是必须存在于链接了crt1.o的程序的, 否则连接器会直接报错.而不使用crt, 则也用不了libc.

在嵌入式开发上, [基于embedded-hal的各种库](https://github.com/rust-embedded/awesome-embedded-rust)发展出了一套完整的`no_std`生态, 包括运行时(rt), CPU指令, 使用svd2rust自动生成的寄存器访问库(Peripheral access API), 各种板级支持包和驱动库.而这一切, 只需要少量的汇编和连接器脚本, 剩下的99%都是rust, 没有C语言的踪影.

然而在桌面领域, 情况则有所不同.

#### Windows

在Windows诞生时, C语言第一个标准ISO C90还不存在, Windows SDK也并不使用C标准库来实现其功能.以至于直到现在, 在Windows上开发程序并不需要用到一行c标准库的内容, 而是需要使用Windows API(win32)或者更高层的封装(例如COM或者.NET), 甚至可以使用不稳定但更加底层的NT native API做一些~~奇怪的~~事情(比如开发驱动程序). 虽然Windows本身是使用C++和C编写的, 但是整个win32 api使用微软自造的C ABI(stdcall), 因此其他语言也可以直接使用win32 api.[winapi-rs](https://github.com/retep998/winapi-rs)就几乎完整的提供了Windows 10SDK中定义的各个win32 api, 而且支持no_std模式. [tinywin](https://github.com/janiorca/tinywin)就是一个使用winapi-rs的rust项目, 且开启了no_std, 不链接到msvcrt, 并且调用win32 api创建了一个窗口.

#### macOS

macOS是一个闭源操作系统, 虽然其基础Darwin系统和XNU内核是开源的, 但是整个用户态几乎都是闭源的.而且, 不像Linux, XNU并没有一个稳定的syscall接口, 也不像win32, macOS的API经常变动(每一年的macOS都会产生许多breaking change).而且, macOS的基础库, 闭源的libSystem.dylib(包括了c标准库和POSIX的实现)只能动态链接, 因此所有的macOS程序都必须使用libc.

#### Linux

为什么最后提Linux呢, 因为Linux是开源软件, 而且Linux内核开发者承诺对用户态的ABI保持不变.因此通过直接调用syscall的方式可以使用Linux内核提供的各种功能, 甚至实现一个[不需要libc的std](https://github.com/rust-lang/rfcs/issues/2610).实际上, go语言也是这么做的, go的标准库并不依赖libc.但是目前rust并没有一个这样的库.

如何从汇编编写Linux userspace的初始化代码是一件复杂的事情, @fasterthanlime的[制作Linux的elf自解压程序系列文章](https://fasterthanli.me/series/making-our-own-executable-packer)介绍了其中的知识.
