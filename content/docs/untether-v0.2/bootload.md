+++
Description = ""
date = "2015-11-16T11:56:00+01:00"
title = "Bootload procedure"
parent = "/docs/untether-v0.2/overview/"
prev = "/docs/untether-v0.2/pcr/"
next = "/docs/untether-v0.2/parameter/"
showdisqus = true

+++

**Note: the content of this part is subject to changes due to the lack of specification.**

Here explains the procedure required to boot a RISC-V Linux.

#### System status after power-on

After a hard (power off/on) reset, the whole SoC, including all PCRs are reset to their intial values:

 * **Pipeline**: Flushed.
 * **L1 I$**: Invalidated, PC <= `0x00000000_00000200`.
 * **L1 D$**: Invalidated.
 * **L2**: Invalidate.
 * **PCRs**: reset and interrupt disabled.


|                          |  actual address spaces         | mapped address spaces       | Type  |
| ----------------------   | ------------------------------ | --------------------------- | ----- |
| Memory section 0         |  (`0x00000000 - 0x7FFFFFFF`)   | (`0x00000000 - 0x7FFFFFFF`) | Mem   |
| > *On-chip BRAM (64 KB)* |  (`0x00000000 - 0x0000FFFF`)   | (`0x00000000 - 0x0000FFFF`) | Mem   |
| > *DDR DRAM*             |  (`0x40000000 - 0x7FFFFFFF`)   | (`0x40000000 - 0x7FFFFFFF`) | Mem   |
| I/O section 0            |  (`0x80000000 - 0x8FFFFFFF`)   | (`0x80000000 - 0x8FFFFFFF`) | I/O   |
| > *UART & SD*            |  (`0x80000000 - 0x8001FFFF`)   | (`0x80000000 - 0x8001FFFF`) | I/O   |

#### Copy BBL from SD to DDR RAM

The actual bootloader for RISC-V Linux is a revised Berkeley bootloader (BBL). Since the size of BBL is larger than 64 KB (the size of the on-chip boot RAM), it is stored on an SD card and needed to be copied to DDR RAM. An example program named 'boot' (`$TOP/fpga/board/$FPGA_BOARD/examples/boot.c`) is provided as the first stage bootloader. Please see [FPGA demo - Boot RISC-V Linux](../fpga-demo#boot) for more information.

Before copying BBL to DDR, it is neccessary to map DDR to I/O space in order to bypass L1/2 caches. Otherwise, some parts of BBL may remain in caches and become lost when the first stage bootloader 'boot' resets the SoC to tranmit control to BBL. The mapping used for BBL copying should be looked like:

|                          |  actual address spaces         | mapped address spaces       | Type  |
| ----------------------   | ------------------------------ | --------------------------- | ----- |
| Memory section 0         |  (`0x00000000 - 0x3FFFFFFF`)   | (`0x00000000 - 0x3FFFFFFF`) | Mem   |
| > *On-chip BRAM (64 KB)* |  (`0x00000000 - 0x0000FFFF`)   | (`0x00000000 - 0x0000FFFF`) | Mem   |
| I/O section 0            |  (`0x80000000 - 0x0FFFFFFF`)   | (`0x80000000 - 0x0FFFFFFF`) | I/O   |
| > *UART & SD*            |  (`0x80000000 - 0x8001FFFF`)   | (`0x80000000 - 0x8001FFFF`) | I/O   |
| I/O section 1            |  (`0x40000000 - 0x7FFFFFFF`)   | (`0x40000000 - 0x7FFFFFFF`) | I/O   |
| > *DDR DRAM*             |  (`0x40000000 - 0x7FFFFFFF`)   | (`0x40000000 - 0x7FFFFFFF`) | I/O   |

The BBL is then copied from SD to DDR DRAM starting from address `0x40000000`.

#### Soft reset

After copying BBL to DDR, the first stage bootloader maps the DDR RAM to boot memory, consequently the on-chip BRAM is hidden.

|                          |  actual address spaces         | mapped address spaces       | Type  |
| ----------------------   | ------------------------------ | --------------------------- | ----- |
| Memory section 0         |  (`0x40000000 - 0x7FFFFFFF`)   | (`0x00000000 - 0x3FFFFFFF`) | Mem   |
| > *DDR DRAM*             |  (`0x40000000 - 0x7FFFFFFF`)   | (`0x00000000 - 0x3FFFFFFF`) | Mem   |
| I/O section 0            |  (`0x80000000 - 0x0FFFFFFF`)   | (`0x80000000 - 0x0FFFFFFF`) | I/O   |
| > *UART & SD*            |  (`0x80000000 - 0x8001FFFF`)   | (`0x80000000 - 0x8001FFFF`) | I/O   |
| Other                    |                                |                             |       |
| > *On-chip BRAM (64 KB)* |  (`0x00000000 - 0x0000FFFF`)   | Hidden                      | N/A   |

Once the mapping is done, the first stage bootloader issues a soft reset:

 * **Pipeline**: Flushed.
 * **L1 I$**: Invalidated, PC <= `0x00000000_00000200`.
 * **L1 D$**: Invalidated.
 * **L2**: Invalidate.
 * **PCRs**: *Remain unchanged*.

#### BBL

BBL runs after the soft reset as DDR is now mapped to the boot address `0x00000200`. The major function of BBL is to initialize all peripherals, set up the page table and virtual memory, load the Linux kernel from SD to virtual memory, and finally boot the kernel.

During the kernel execution, BBL runs underlyingly as a hypervisor, serving all peripheral requests from Linux using the actaul FPGA hardware.