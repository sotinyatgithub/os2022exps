实验五 输入
=====================

QEMU的virt机器默认没有键盘作为输入设备，但当我们执行QEMU使用 -nographic 参数（disable graphical output and redirect serial I/Os to console）时QEMU会将串口重定向到控制台，因此我们可以使用UART作为输入设备。

tock-registers库
--------------------------

在 :doc:`../exp4/index` 中，针对GICD，GICC，TIMER等硬件我们定义了大量的常量和寄存器值，这在使用时很不方便也容易出错。因此我们决定使用 tock-registers 库。


在 Cargo.toml 中的 ``[dependencies]`` 节中加入依赖：

.. code-block:: 

    tock-registers = { version = "0.7.x", default-features = false, features = ["register_types"], optional = true }


为了不至于使 uart_console.rs 文件过长，我们选择重构 uart_console.rs。首先创建 src/uart_console 目录，并将原 uart_console.rs 更名为 mod.rs ，且置于 src/uart_console 目录下， 最后新建 src/uart_console/pl011.rs 文件，然后依据tock_registers库的要求对pl011所涉及到的寄存器 [1]_ 进行描述，内容为：

.. code-block:: rust

    use tock_registers::{registers::{ReadOnly, ReadWrite, WriteOnly}, register_bitfields, register_structs};

    pub const PL011REGS: *mut PL011Regs = (0x0900_0000) as *mut PL011Regs;

    register_bitfields![
        u32,

        pub UARTDR [
            DATA OFFSET(0) NUMBITS(8) []
        ],
        /// Flag Register
        pub UARTFR [
            /// Transmit FIFO full. The meaning of this bit depends on the
            /// state of the FEN bit in the UARTLCR_ LCRH Register. If the
            /// FIFO is disabled, this bit is set when the transmit
            /// holding register is full. If the FIFO is enabled, the TXFF
            /// bit is set when the transmit FIFO is full.
            TXFF OFFSET(6) NUMBITS(1) [],

            /// Receive FIFO empty. The meaning of this bit depends on the
            /// state of the FEN bit in the UARTLCR_H Register. If the
            /// FIFO is disabled, this bit is set when the receive holding
            /// register is empty. If the FIFO is enabled, the RXFE bit is
            /// set when the receive FIFO is empty.
            RXFE OFFSET(4) NUMBITS(1) []
        ],

        /// Integer Baud rate divisor
        pub UARTIBRD [
            /// Integer Baud rate divisor
            IBRD OFFSET(0) NUMBITS(16) []
        ],

        /// Fractional Baud rate divisor
        pub UARTFBRD [
            /// Fractional Baud rate divisor
            FBRD OFFSET(0) NUMBITS(6) []
        ],

        /// Line Control register
        pub UARTLCR_H [
            /// Parity enable. If this bit is set to 1, parity checking and generation 
            /// is enabled, else parity is disabled and no parity bit added to the data frame. 
            PEN OFFSET(1) NUMBITS(1) [
                Disabled = 0,
                Enabled = 1
            ],
            /// Two stop bits select. If this bit is set to 1, two stop bits are transmitted 
            /// at the end of the frame. 
            STP2 OFFSET(3) NUMBITS(1) [
                Stop1 = 0,
                Stop2 = 1
            ],
            /// Enable FIFOs. 
            FEN OFFSET(4) NUMBITS(1) [
                Disabled = 0,
                Enabled = 1
            ],
            
            /// Word length. These bits indicate the number of data bits
            /// transmitted or received in a frame.
            WLEN OFFSET(5) NUMBITS(2) [
                FiveBit = 0b00,
                SixBit = 0b01,
                SevenBit = 0b10,
                EightBit = 0b11
            ]
        ],

        /// Control Register
        pub UARTCR [
            /// Receive enable. If this bit is set to 1, the receive
            /// section of the UART is enabled. Data reception occurs for
            /// UART signals. When the UART is disabled in the middle of
            /// reception, it completes the current character before
            /// stopping.
            RXE    OFFSET(9) NUMBITS(1) [
                Disabled = 0,
                Enabled = 1
            ],

            /// Transmit enable. If this bit is set to 1, the transmit
            /// section of the UART is enabled. Data transmission occurs
            /// for UART signals. When the UART is disabled in the middle
            /// of transmission, it completes the current character before
            /// stopping.
            TXE    OFFSET(8) NUMBITS(1) [
                Disabled = 0,
                Enabled = 1
            ],

            /// UART enable
            UARTEN OFFSET(0) NUMBITS(1) [
                /// If the UART is disabled in the middle of transmission
                /// or reception, it completes the current character
                /// before stopping.
                Disabled = 0,
                Enabled = 1
            ]
        ],

        pub UARTIMSC [
            RXIM OFFSET(4) NUMBITS(1) [
                Disabled = 0,
                Enabled = 1
            ]
        ],
        /// Interupt Clear Register
        pub UARTICR [
            /// Meta field for all pending interrupts
            ALL OFFSET(0) NUMBITS(11) [
                Clear = 0x7ff
            ]
        ]
    ];

    register_structs! {
        pub PL011Regs {
            (0x00 => pub dr: ReadWrite<u32, UARTDR::Register>),                   // 0x00
            (0x04 => __reserved_0),               // 0x04
            (0x18 => pub fr: ReadOnly<u32, UARTFR::Register>),      // 0x18
            (0x1c => __reserved_1),               // 0x1c
            (0x24 => pub ibrd: WriteOnly<u32, UARTIBRD::Register>), // 0x24
            (0x28 => pub fbrd: WriteOnly<u32, UARTFBRD::Register>), // 0x28
            (0x2C => pub lcr_h: WriteOnly<u32, UARTLCR_H::Register>), // 0x2C
            (0x30 => pub cr: WriteOnly<u32, UARTCR::Register>),     // 0x30
            (0x34 => __reserved_2),               // 0x34
            (0x38 => pub imsc: ReadWrite<u32, UARTIMSC::Register>), // 0x38
            (0x44 => pub icr: WriteOnly<u32, UARTICR::Register>),   // 0x44
            (0x48 => @END),
        }
    }

看起来好像比 :doc:`../exp4/index` 中对应的寄存器描述部分要复杂，但如果你熟悉了之后，基本上可以依据技术参考手册中的寄存器描述无脑写了。

.. hint:: register_structs 宏最后需加上(0x** => @END)，表示结束。

    register_bitfields 宏按照寄存器的位结构进行描述，注意最后要加分号“;”，只要注册自己想处理的位即可。

至此，目录结构看起来像这样：

.. code-block:: 

    .
    |____virt.dts
    |____virt.dtb
    |____Cargo.toml
    |____Cargo.lock
    |____.cargo
    | |____config.toml
    |____aarch64-qemu.ld
    |____.gitignore
    |____.vscode
    | |____launch.json
    |____aarch64-unknown-none-softfloat.json
    |____src
    | |____panic.rs
    | |____start.s
    | |____interrupts.rs
    | |____main.rs
    | |____uart_console
    | | |____mod.rs
    | | |____pl011.rs
    | |____exception.s


数据接收中断
--------------------------

在 src/uart_console/mod.rs 中引入库

.. code-block:: rust

    use tock_registers::{interfaces::Writeable};

    pub mod pl011;
    use pl011::*;

    // const PL011REGS: *mut PL011Regs = (0x08000000) as *mut PL011Regs;

    lazy_static! {
        /// A global `Writer` instance that can be used for printing to the VGA text buffer.
        ///
        /// Used by the `print!` and `println!` macros.
        pub static ref WRITER: Mutex<Writer> = Mutex::new(Writer::new());
    }

同时为 ``Writer`` 结构实现构造函数如下：

.. code-block:: rust

    pub fn new() -> Writer{
        unsafe {
            // 禁用pl011
            (*PL011REGS).cr.write(UARTCR::TXE::Disabled + UARTCR::RXE::Disabled + UARTCR::UARTEN::Disabled);
            // 清空中断状态
            (*PL011REGS).icr.write(UARTICR::ALL::Clear);
            // 设定中断mask，需要使能的中断
            (*PL011REGS).imsc.write(UARTIMSC::RXIM::Enabled);
            // IBRD = UART_CLK / (16 * BAUD_RATE)
            // FBRD = ROUND((64 * MOD(UART_CLK,(16 * BAUD_RATE))) / (16 * BAUD_RATE))
            // UART_CLK = 24M
            // BAUD_RATE = 115200
            (*PL011REGS).ibrd.write(UARTIBRD::IBRD.val(13));
            (*PL011REGS).fbrd.write(UARTFBRD::FBRD.val(1));
            // 8N1 FIFO enable
            (*PL011REGS).lcr_h.write(UARTLCR_H::WLEN::EightBit + UARTLCR_H::PEN::Disabled + UARTLCR_H::STP2::Stop1 
                + UARTLCR_H::FEN::Enabled);
            // enable pl011
            (*PL011REGS).cr.write(UARTCR::UARTEN::Enabled + UARTCR::RXE::Enabled + UARTCR::TXE::Enabled);
        }

        Writer
    }

在 src/interrupts.rs 中的 ``init_gicv2`` 函数中对UART的数据接收中断进行初始化：

.. code-block:: rust

    // 初始化UART0 中断
    // interrupts = <0x00 0x01 0x04>; SPI, 0x01, level
    set_config(UART0_IRQ, ICFGR_LEVEL); //电平触发
    set_priority(UART0_IRQ, 0); //优先级设定
    // set_core(TIMER_IRQ, 0x1); // 单核实现无需设置中断目标核
    clear(UART0_IRQ); //清除中断请求
    enable(UART0_IRQ); //使能中断

然后对UART的数据接收中断进行处理，并修改timer中断的处理方法，使之每隔2秒输出一个点。

.. code-block:: rust

    const UART0_IRQ: u32 = 33;


    fn handle_irq_lines(ctx: &mut ExceptionCtx, _core_num: u32, irq_num: u32) {
        if irq_num == TIMER_IRQ {
            handle_timer_irq(ctx);
        }else if irq_num == UART0_IRQ {
            handle_uart0_rx_irq(ctx);
        }
        else{
            catch(ctx, EL1_IRQ);
        }
    }

    fn handle_timer_irq(_ctx: &mut ExceptionCtx){
        
        crate::print!(".");

        // 每2秒产生一次中断
        unsafe {
            asm!("mrs x1, CNTFRQ_EL0");
            asm!("add x1, x1, x1");
            asm!("msr CNTP_TVAL_EL0, x1");
        }
        
    }

    fn handle_uart0_rx_irq(_ctx: &mut ExceptionCtx){
        use crate::uart_console::pl011::*;

        // crate::print!("R");
        unsafe{
            let mut flag = (*PL011REGS).fr.read(UARTFR::RXFE);
            while flag != 1 {
                let value = (*PL011REGS).dr.read(UARTDR::DATA);
            
                crate::print!("{}", value as u8 as char);
                flag = (*PL011REGS).fr.read(UARTFR::RXFE);
            }
        }
    }        


    #[no_mangle]
    unsafe extern "C" fn el1_irq(ctx: &mut ExceptionCtx) {
        // reads this register to obtain the interrupt ID of the signaled interrupt. 
        // This read acts as an acknowledge for the interrupt.
        let value: u32 = ptr::read_volatile(GICC_IAR);
        let irq_num: u32 = value & 0x1ff;
        let core_num: u32 = value & 0xe00;

        // 处理中断
        handle_irq_lines(ctx, core_num, irq_num);
        // catch(ctx, EL1_IRQ);

        // A processor writes to this register to inform the CPU interface either:
        // • that it has completed the processing of the specified interrupt
        // • in a GICv2 implementation, when the appropriate GICC_CTLR.EOImode bit is set to 1, to indicate that the interface should perform priority drop for the specified interrupt.
        ptr::write_volatile(GICC_EOIR, core_num | irq_num);
        clear(irq_num);
    }


.. [1] https://developer.arm.com/documentation/ddi0183/g/programmers-model/summary-of-registers?lang=en