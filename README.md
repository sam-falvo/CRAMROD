# CRAMROD

Intended to be used in conjunction with the MyLA core,
this core aims to help me diagnose/debug problems with accessing the Digilent Nexys-2 "Cellular" RAM chip.
CRAMROD is an acronym, standing for Cellular RAM Research-Oriented Debugging core.

**What follows is preliminary.**

## Registers

The following registers are specified:

|Register Select|Name|R/W|Purpose|
|:----:|:--:|:-:|:------|
|0|RCPL|R/W|Supplies bits 15-1 of the PSRAM byte address to read or write.  Since the RAM chip supplies a halfword wide bus, bit 0 is ignored on write, and always reads back as 0.|
|1|RCPH|R/W|Supplies bits 23-16 of the PSRAM halfword address to read or write.  Bits 14-8 are ignored on write, and always reads back as 0.  Bit 15, if set on write, commences a read operation from PSRAM, thus loading the `RCDI` register.  Bit 15 always reads back as 0.|
|2|RCDI|R/O|Records the most recently read data from PSRAM.|
|2|RCDO|W/O|Records the data to write out to PSRAM.  This commences a write transaction with the PSRAM chip, but does not block the master.|
|3|RCST|R/O|Status register telling if an operation is still in progress.|
|3|RCBCR|W/O|Commences an asynchronous write to the PSRAM chip to program a new Bus Configuration Register setting.|

## RCPL

| 15:1 | 0 |
|:------:|:-:|
|Bits 15-1 of the byte address to read or write.| 0 |

## RCPH

| 15 | 14:8 | 7:0 |
|:--:|:----:|:---:|
| StartRead | 0 | Bits 23-16 of the byte address to read or write. |

## RCDI

| 15:0 |
|:----:|
|The most recently read value from the PSRAM chip.|

Note that the value of this register becomes stable only after a read operation has finished.
The following code illustrates how to do this:

    ; From C:  int x = read_from_psram(int address, CRAMROD *mmio);

    read_from_psram:
        ; Write out bits 15-1 of the address.
        sh      a0, RCPL(a1)

        ; Write out bits 23-16 of the address, plus high bit set to start read.
        srli    a0, a0, 16
        andi    a0, a0, 255
        addi    a2, x0, 1
        slli    a2, a2, 15
        or      a0, a0, a2
        sh      a0, RCPH(a1)

        ; Wait for read operation to complete.
    _rfp0:
        lh      a0, RCST(a1)
        blt     a0, x0, _rfp0

        ; Return value read
        lh      a0, RCDI(a1)
        jalr    x0, 0(ra)

## RCDO

| 15:0 |
|:----:|
|The value to write out to PSRAM at the currently selected address.|

The act of writing to this register starts the write operation.
The bus master does *not* block until it completes.
The following code illustrates how to write to PSRAM:

    ; From C:  write_to_psram(int address, int data, CRAMROD *mmio);

    write_to_psram:
        ; Write out bits 15-1 of the address.
        sh      a0, RCPL(a2)

        ; Write out bits 23-16 of the address.  Ensure StartRead is 0.
        srli    a0, a0, 16
        andi    a0, a0, 255
        sh      a0, RCPH(a2)

        ; Write the data and start the write operation.
        sh      a1, RCDO(a2)

        ; Return now since this happens asynchronously.
        jalr    x0, 0(ra)

    ; Wait for a write to complete.

    wait_for_psram:
        lh      a0, RCST(a1)
        blt     a0, x0, wait_for_psram
        jalr    x0, 0(ra)

## RCST

| 15 | 14:0 |
|:--:|:----:|
|busy| 0 |

The meaning of the bits follows.

|Bit|Name|Purpose|
|:-:|:--:|:------|
|15|busy|**1** if the PSRAM controller is still running a read or write operation in the background.  **0** when it's finished.|

## RCBCR

This information was taken from preliminary data sheets on the Cellular RAM 1.5 product line for Micron.

| 15 | 14 | 13:11 | 10 | 9 | 8 | 7:6 | 5:4 | 3 | 2:0 |
|:--:|:--:|:-----:|:--:|:-:|:-:|:---:|:---:|:-:|:---:|
| mode | initialLatency | latencyCounter | waitPolarity | 0 | waitConfig | 0 | driveStrength | burstWrap | burstLength |

The meaning of the bits follows.

|Bit|Name|Default|Purpose|
|:-:|:--:|:-----:|:------|
|15|mode|1|**0** selects Synchronous Burst Access Mode; **1** selects asynchronous access mode.|
|14|initialLatency|0|**0** selects Variable Initial Latency; **1** selects Fixed Initial Latency.|
|13:11|latencyCounter|3|**2** through **6** selects the number of clocks to wait before proceeding with the actual data transfer.  This value depends on how fast you clock the PSRAM chip.  If assigned a value *n*, then the latency is set to *n*+1 clocks.  **0**, **1**, and **7** are reserved and must not be used.|
|10|waitPolarity|1|**0** configures the WAIT signal from the PSRAM chip to be active-LOW, in effect turning this signal into a READY signal instead.  **1** configures the WAIT signal to be active-HIGH.|
|8|waitConfig|1|**0** configures WAIT to be driven *during* the delay.  **1** configures it to be driven *one clock ahead* of the delay.|
|5:4|driveStrength|1|This setting configures how much current the PSRAM chip drives its wires with.  **0** configures full drive strength, **1** configures one half drive strength, **2** one quarter, and **3** is reserved and should not be used.|
|3|burstWrap|1|**0** configures a burst access to wrap within the burst length.  This ensures it never crosses a page boundary.  **1** configures a burst to proceed linearly from its start address.  Bursting close to a page boundary can cause that boundary to be crossed, leading to slower read times.  Be sure to monitor the WAIT pin when using this mode.|
|2:0|burstLength|7|Configures how many halfwords to transfer in a burst.  Valid values include **1** for 4 halfwords, **2** for 8 halfwords, **3** for 16 halfwords, **4** for 32 halfwords, and **7** for continuous burst.  All other values are reserved and should not be used.|

