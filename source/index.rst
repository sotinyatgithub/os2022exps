.. blogos_armv8 documentation master file, created by
   sphinx-quickstart on Fri Jan 21 17:26:34 2022.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

移植 blogos 到 ARM v8!
========================================

.. toctree::
   :maxdepth: 2
   :caption: Contents:

   exp1/index
   exp2/index


欢迎来到 rCore-Tutorial-Book 第三版！

.. note::

   :doc:`/log` 

   项目/文档于 2022-01-02 更新，情况如下：

   - 第一章更新完成，Rust 版本升级至 ``nightly-2022-01-01`` ， ``asm`` 和 ``global_asm`` 特性已稳定，相关的宏可在 ``core::arch`` 中找到。更新了作者和版权信息，版本暂定 ``3.6.0-alpha.1`` 。

项目简介
---------------------

这本教程旨在一步一步展示如何 **从零开始** 用 **Rust** 语言写一个基于 **RISC-V** 架构的 **类 Unix 内核** 。值得注意的是，本项目不仅支持模拟器环境（如 Qemu/terminus 等），还支持在真实硬件平台 Kendryte K210 上运行。

.. Indices and tables
.. ==================

.. * :ref:`genindex`
.. * :ref:`modindex`
.. * :ref:`search`
