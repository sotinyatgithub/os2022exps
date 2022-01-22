实验二 Hello World
=====================

print函数是学习几乎任何一种软件开发语言时最先学习使用的函数，同时该函数也是最基本和原始的程序调试手段，但该函数的实现却并不简单。本实验的目的在于理解操作系统与硬件的接口方法，并实现一个可打印字符的宏（非系统调用），用于后续的调试和开发。

了解virt机器
--------------------------

操作系统介于硬件和应用程序之间，向下管理硬件资源，向上提供应用编程接口。设计并实现操作系统需要熟悉底层硬件的组成及其操作方法。

本系列实验都会在QEMU模拟器上完成，首先来了解一下模拟的机器信息。可以通过下列两种方法：

1. 查看QEMU关于 `virt的描述 <https://www.qemu.org/docs/master/system/arm/virt.html>`_ ， 或者查看QEMU的源码，如github上的 `virt.h <https://github.com/qemu/qemu/blob/master/include/hw/arm/virt.h>`_ 和 `virt.c <https://github.com/qemu/qemu/blob/master/hw/arm/virt.c>`_。
   
2. 通过QEMU导出设备树 

  1. 安装设备树格式转换工具

  Mac下安装

  .. code-block:: console
    
    
    $  brew install dtc

  Linux下安装

  .. code-block:: console

    $ apt-get install device-tree-compiler


  2. 通过QEMU导出设备树并转成可读格式

  .. code-block:: console

    $ qemu-system-aarch64 -machine virt,dumpdtb=virt.dtb -cpu cortex-a53 -nographic 
    $ dtc -I dtb -O dts -o virt.dts virt.dtb

  .. note::
    -machine virt 指明机器类型为virt，这是QEMU仿真的虚拟机器。

  virt.dtb转换后生成的virt.dts中可找到如下内容

  .. code-block::

    pl011@9000000 {
        clock-names = "uartclk\0apb_pclk";
        clocks = <0x8000 0x8000>;
        interrupts = <0x00 0x01 0x04>;
        reg = <0x00 0x9000000 0x00 0x1000>;
        compatible = "arm,pl011\0arm,primecell";
    };
        
    chosen {
        stdout-path = "/pl011@9000000";
        kaslr-seed = <0xcbd0568d 0xb463306c>;
    };

  由上可以看出，virt机器包含有pl011的设备，该设备的寄存器在0x9000000开始处。pl011实际上是一个UART设备，即串口。可以看到virt选择使用pl011作为标准输出，这是因为与PC不同，大部分嵌入式系统默认情况下并不包含VGA设备。


实现println!宏
--------------------------

我们参照 `Writing an OS in Rust <https://os.phil-opp.com/vga-text-mode/>`_ （ `中文版 <https://github.com/rustcc/writing-an-os-in-rust/blob/master/03-vga-text-mode.md>`_ ）来实现println!宏，但与之不同，我们使用串口来输出，而不是通过操作VGA的Frame Buffer。

新建 src/uart_console.rs 文件，完成

- 定义一个Writer结构，实现字节写入和字符串写入。

.. code-block:: rust

  //嵌入式系统使用串口，而不是vga，直接输出，没有颜色控制，不记录列号，也没有frame buffer，所以采用空结构
  pub struct Writer;

  //实现字节写入和字符串写入
  impl Writer {
      pub fn write_byte(&mut self, byte: u8) {
          const UART0: *mut u8 = 0x0900_0000 as *mut u8;
          unsafe {
              ptr::write_volatile(UART0, byte);
          }
      }

      pub fn write_string(&mut self, s: &str) {
          for byte in s.chars() {
              self.write_byte(byte as u8)        
          }
      }

  }

通过往串口的寄存器中写入字符，实现输出。

.. note::
  如何操作硬件通常需要阅读硬件制造商提供的技术手册。如pl011串口设备（PrimeCell UART）是arm设计的，其技术参考手册可以通过其 `官网 <https://developer.arm.com/documentation/ddi0183/latest/>`_ 查看。也可以通过顶部的下载链接下载pdf版本，如下图所示。

  .. image:: down-pl011-ref.png

  .

  依据之前virt.dts中的描述，pl011的寄存器在virt机器中被映射到了0x9000000的内存位置。通过访问pl011的技术参考手册中的“Chapter 3. Programmers Model”中的”Summary of registers“一节可知，第0号寄存器是pl011串口的数据寄存器，用于数据的收发。其详细描述参见 `这里 <https://developer.arm.com/documentation/ddi0183/g/programmers-model/register-descriptions/data-register--uartdr?lang=en>`_。

  注意到我们只是向UART0写入，而没从UART0读出（如果读出会读出其他设备通过串口发送过来的数据，而不是刚才写入的数据，注意体会这与读写内存时是不一样的，详情参见pl011的技术手册），编译器在优化时可能对这部分代码进行错误的优化，如把这些操作都忽略掉（因为编译器认为这些写入的数据不会再使用，所以可以合理地剔除这些数据写入代码而不对结果产生影响）。使用ptr::write_volatile库的目的是告诉编译器，这些写入有特定目的，不应将其优化。

- 为Write结构实现core::fmt::Write trait，该trait会自动实现write_fmt方法，支持格式化。

.. code-block:: rust

  impl core::fmt::Write for Writer {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        self.write_string(s);

        Ok(())
    }
  }

基于Rust的core::fmt实现格式化控制，可以使我们方便地打印不同类型的变量。实现core::fmt::Write后，我们就可以使用Rust内置的格式化宏write!和writeln!，这使你瞬间具有其他语言运行时所提供的格式化控制能力。
   
- 测试一下是否正常工作
  
  .. code-block:: rust

    pub fn print_something() {
        //一定要引用core::fmt::Write;，否则报错：no method named `write_fmt` found for struct `Writer` in the current scope。
        pub use core::fmt::Write;

        let mut writer = Writer{};
        let display: fmt::Arguments = format_args!("hello arguments!\n");

        writer.write_byte(b'H');
        writer.write_string("ello ");
        writer.write_string("Wörld!\n");
        writer.write_string("[0] Hello from Rust!");

        // 通过实现core::fmt::Write自动实现的方法
        writer.write_fmt(display).unwrap();
        // 使用write!宏
        write!(writer, "The numbers are {} and {} \n", 42, 1.0/3.0).unwrap();
    }

- 实现全局接口

现在我们已经可以采用print_something函数通过串口输出字符了。但为了输出，我们需要两个步骤：（1）创建Writer类型的实例，（2）调用实例的write_byte或write_string等函数。

为了方便在其他模块中调用，我们希望可以直接执行步骤（2）而不是首先执行上述步骤（1）再执行步骤（2）。一般情况下可以通过将步骤（1）中的实例定义为static类型来实现，但Rust暂不支持Writer这样类型的静态（编译时）初始化，需要使用lazy_static来解决。此外，为了保证访问安全还引入了自旋锁（spin）。

在Cargo.toml中加入如下依赖：

.. code-block:: 

  [dependencies]
  # device_tree = "1.1.0"
  spin = "0.9.2"

  [dependencies.lazy_static]
  version = "1.0"
  features = ["spin_no_std"]

在 src/uart_console.rs 中加入：

.. code-block:: rust

  use lazy_static::lazy_static;
  use spin::Mutex;

  lazy_static! {
      /// A global `Writer` instance that can be used for printing to the VGA text buffer.
      ///
      /// Used by the `print!` and `println!` macros.
      pub static ref WRITER: Mutex<Writer> = Mutex::new(Writer { });
  }

- 实现println!宏
  
最后在 src/uart_console.rs 中实现print!和println!宏。

.. code-block:: rust

  /// Like the `print!` macro in the standard library, but prints to the VGA text buffer.
  #[macro_export]
  macro_rules! print {
      ($($arg:tt)*) => ($crate::uart_console::_print(format_args!($($arg)*)));
  }

  /// Like the `println!` macro in the standard library, but prints to the VGA text buffer.
  #[macro_export]
  macro_rules! println {
      () => ($crate::print!("\n"));
      ($($arg:tt)*) => ($crate::print!("{}\n", format_args!($($arg)*)));
  }

  /// Prints the given formatted string to the VGA text buffer through the global `WRITER` instance.
  #[doc(hidden)]
  pub fn _print(args: fmt::Arguments) {
      use core::fmt::Write;

      WRITER.lock().write_fmt(args).unwrap();
  }

- 使用println!宏

在main.rs中使用println!宏。

.. code-block:: rust

  mod uart_console;

  #[no_mangle]
  pub extern "C" fn not_main() {
    // ...

    println!("[0] Hello from Rust!");

    // ...
  }

至此，我们获得了一个基本的输出和调试手段，如我们可以在系统崩溃时调用print!或者println!宏进行输出。

我们可以利用print!宏来打印一个文本banner让我们写的OS显得专业一点😁。 `manytools.org <https://manytools.org/hacker-tools/ascii-banner/>`_ 可以创建ascii banner，选择你喜欢的样式和文字，然后在main.rs的not_main函数中通过print!宏输出。

.. code-block:: rust

  mod uart_console;

  #[no_mangle]
  pub extern "C" fn not_main() {
    // ...

    let banner = r#"
      ___  ____                     _    ____  __  __       ___      ____  _   _ _   _ _   _ 
      / _ \/ ___|    ___  _ __      / \  |  _ \|  \/  |_   _( _ )    / __ \| | | | \ | | | | |
    | | | \___ \   / _ \| '_ \    / _ \ | |_) | |\/| \ \ / / _ \   / / _` | |_| |  \| | | | |
    | |_| |___) | | (_) | | | |  / ___ \|  _ <| |  | |\ V / (_) | | | (_| |  _  | |\  | |_| |
      \___/|____/   \___/|_| |_| /_/   \_\_| \_\_|  |_| \_/ \___/   \ \__,_|_| |_|_| \_|\___/ 
                                                                    \____/                      
      "#;

    print!("{}",banner);

    // ...
  }
