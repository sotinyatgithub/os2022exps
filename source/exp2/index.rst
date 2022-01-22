实验二 Hello World
=====================

Virt机器
--------------------------

本系列实验都会在QEMU模拟器上完成，首先来了解一下模拟的机器信息。可以通过下列两种方法：

1. 查看QEMU的源码了解相关信息，如github上的 `virt.h <https://github.com/qemu/qemu/blob/master/include/hw/arm/virt.h>`_ 和 `virt.c <https://github.com/qemu/qemu/blob/master/hw/arm/virt.c>`_。 或者查看QEMU关于 `virt的描述 <https://www.qemu.org/docs/master/system/arm/virt.html>`_。
   
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
    $ dtc -I dtb -O dts -o a.dts a.dtb

  .. note::
    -machine virt 指明机器类型为virt，这是QEMU仿真的虚拟机器。











