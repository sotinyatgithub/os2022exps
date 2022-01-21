实验一 环境配置 
=====================

安装工具链
--------------------------


安装rust
^^^^^^^^^^^^^^^^^^^^^^^^^^

如果你已经安装了一个版本的Rust，需补充安装相关工具： 
::

	cargo install cargo-binutils rustfilt

如果你想要全新安装：
::

	curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
	source $HOME/.cargo/env
	cargo install cargo-binutils rustfilt

如果你想安装指定的版本：
::
	rustup install nightly-2018-01-09


.. attention:: 
	可以将rust默认设置为stable或nightly版本
	::
		rustup default stable
		rustup default nightly
	或者仅将当前项目设为nightly
	::
		rustup override set nightly

查看当前项目的rust版本
::
	rustc -V

如果你使用 Visual Studio Code，强烈推荐你安装 `Rust Analyzer 扩展 <https://marketplace.visualstudio.com/items?itemName=matklad.rust-analyzer>`_

为rust增加armv8支持
^^^^^^^^^^^^^^^^^^^^^^^^^^^

列出 rust支持的目标三元组（CPU架构、平台供应者、操作系统和应用程序二进制接口ABI）
:: 
	rustup target list

增加 armv8支持
:: 
	rustup target add aarch64-unknown-none-softfloat


安装QEMU模拟器
^^^^^^^^^^^^^^^^^^^^^^^^^^^

请参考 https://wiki.qemu.org/Hosts/Mac 或者 https://wiki.qemu.org/Hosts/Mac


安装交叉编译工具链 (aarch64)
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Linux
::
	wget https://developer.arm.com/-/media/Files/downloads/gnu-a/10.2-2020.11/binrel/gcc-arm-10.2-2020.11-x86_64-aarch64-none-elf.tar.xz 
	tar -xf gcc-arm-10* 
	cp gcc-arm-10*/bin/aarch64-none-elf-objdump gcc-arm-10*/bin/aarch64-none-elf-readelf gcc-arm-10*/bin/aarch64-none-elf-nm /usr/local/bin/ 
	rm -rf gcc-arm-10*

Mac
::
	brew tap SergioBenitez/osxct
	brew install aarch64-none-elf


创建裸机(Bare Metal)程序
--------------------------

用cargo创建项目
^^^^^^^^^^^^^^^^^^^^^^^^^^

创建新项目：
::
	cargo new rui_armv8_os --bin --edition 2021


在src/下创建main.rs, panic.rs, start.s三个文件

main.rs源码

.. code-block:: rust
    :linenos:

	// 不使用标准库
	#![no_std]
	// 不使用预定义入口点
	#![no_main]
	#![feature(global_asm)]

	use core::ptr;

	mod panic;

	global_asm!(include_str!("start.s"));

	#[no_mangle] // 不重整函数名
	pub extern "C" fn not_main() {
	    const UART0: *mut u8 = 0x0900_0000 as *mut u8;
	    let out_str = b"AArch64 Bare Metal";
	    for byte in out_str {
	        unsafe {
	            ptr::write_volatile(UART0, *byte);
	        }
	    }
	}

panic.rs源码

.. code-block:: rust
    :linenos:


	use core::panic::PanicInfo;

	#[panic_handler]
	fn on_panic(_info: &PanicInfo) -> ! {
		loop {}
	}


start.s源码

.. code-block:: asm
    :linenos:

	.globl _start
	.extern LD_STACK_PTR

	.section ".text.boot"

	_start:
		ldr     x30, =LD_STACK_PTR
		mov     sp, x30
		bl      not_main


	.equ PSCI_SYSTEM_OFF, 0x84000008
	.globl system_off
	system_off:
		ldr     x0, =PSCI_SYSTEM_OFF
		hvc     #0	


创建链接文件aarch64-qemu.ld
::

	ENTRY(_start)
	SECTIONS
	{
	    . = 0x40080000;
	    .text.boot : { *(.text.boot) }
	    .text : { *(.text) }
	    .data : { *(.data) }
	    .rodata : { *(.rodata) }
	    .bss : { *(.bss) }

	    . = ALIGN(8);
	    . = . + 0x4000;
	    LD_STACK_PTR = .;
	}


配置Cargo.toml
::

	[package]
	name = "rui_armv8_os"
	version = "0.1.0"
	edition = "2018"
	authors = ["Rui Li <rui@hnu.edu.cn>"]

	# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html


	# [build]
	# 设定编译目标，cargo build --target aarch64-unknown-none-softfloat
	# target = "aarch64-unknown-none-softfloat"

	[dependencies]

	# eh_personality语言项标记的函数，将被用于实现栈展开（stack unwinding）。
	# 在使用标准库的情况下，当panic发生时，Rust将使用栈展开，来运行在栈上活跃的
	# 所有变量的析构函数（destructor）——这确保了所有使用的内存都被释放。
	# 如果不禁用会出现错误：language item required, but not found: `eh_personality`
	# 通过下面的配置禁用栈展开
	# dev时禁用panic时栈展开
	[profile.dev]
	panic = "abort"

	# release时禁用panic时栈展开
	[profile.release]
	panic = "abort"

创建aarch64-unknown-none-softfloat.json
::

	{
	  "abi-blacklist": [
	    "stdcall",
	    "fastcall",
	    "vectorcall",
	    "thiscall",
	    "win64",
	    "sysv64"
	  ],
	  "arch": "aarch64",
	  "data-layout": "e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128",
	  "disable-redzone": true,
	  "env": "",
	  "executables": true,
	  "features": "+strict-align,+neon,+fp-armv8",
	  "is-builtin": false,
	  "linker": "rust-lld",
	  "linker-flavor": "ld.lld",
	  "linker-is-gnu": true,
	  "pre-link-args": {
	    "ld.lld": ["-Taarch64-qemu.ld"]
	  },
	  "llvm-target": "aarch64-unknown-none",
	  "max-atomic-width": 128,
	  "os": "none",
	  "panic-strategy": "abort",
	  "relocation-model": "static",
	  "target-c-int-width": "32",
	  "target-endian": "little",
	  "target-pointer-width": "64",
	  "vendor": ""
	}


编译项目
^^^^^^^^^^^^^^^^^^^^^^^^

::

	cargo build --target aarch64-unknown-none-softfloat


调试支持
--------------------------

编译成功后就可以运行，这需要使用前面安装的QEMU模拟器。此外，为了查找并修正bug，我们需要使用调试工具。

QEMU进入调试，启动调试服务器，默认端口1234
::
	qemu-system-aarch64 -machine virt -m 1024M -cpu cortex-a53 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os -S -s

启动调试客户端
::
	aarch64-none-elf-gdb target/aarch64-unknown-none-softfloat/debug/rui_armv8_os

设置调试参数，开始调试
::
	(gdb) target remote localhost:1234 
	(gdb) disassemble 
	(gdb) n

.. hint:: 可以使用GDB dashboard进入可视化调试界面

将调试集成到vscode
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

打开一个.rs文件，点击 vscode左侧的运行和调试按钮，弹出对话框选择创建 launch.json文件，增加如下配置：
::
	{

	    "name": "aarch64-gdb",
	    "type": "cppdbg",
	    "request": "launch",
	    "program": "${workspaceFolder}/target/aarch64-unknown-none-softfloat/debug/rui_armv8_os",
	    "stopAtEntry": true,
	    "cwd": "${fileDirname}",
	    "environment": [],
	    "externalConsole": false,
	    "launchCompleteCommand": "exec-run",
	    "MIMode": "gdb",
	    "miDebuggerPath": "/usr/local/bin/aarch64-none-elf-gdb",
	    "miDebuggerServerAddress": "localhost:1234",
	    "setupCommands": [
	        {
	            "description": "Enable pretty-printing for gdb",
	            "text": "-enable-pretty-printing",
	            "ignoreFailures": true
	        }
	    ]     
	},

在左边面板顶部选择刚添加的 aarch64-gdb 选项，点击旁边的绿色 开始调试（F5） 按钮开始调试。

.. image:: vscode-debug.png

qemu执行结果

.. image:: qemu-result.png



