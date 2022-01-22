实验二 Hello World
=====================

Virt机器
--------------------------

本系列实验都会在QEMU模拟器上完成，首先来了解一下模拟的机器信息。

1. 查看QEMU的源码了解相关信息。在github上的 `virt.h <https://github.com/qemu/qemu/blob/master/include/hw/arm/virt.h>`_ 和 `virt.c <https://github.com/qemu/qemu/blob/master/hw/arm/virt.c>`_
   
2. 通过QEMU导出设备树
::

  qemu-system-aarch64 -machine virt,dumpdtb=virt.dtb -cpu cortex-a53 -nographic 
  brew install dtc
  dtc -I dtb -O dts -o a.dts a.dtb







