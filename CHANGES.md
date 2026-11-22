# Changes on branch `fix/win7-on-master`

The starting point was the NT 4.0 DDK DEC LANCE driver as ported by Copilot: 16-bit LANCE mode (SWSTYLE 0), 24-bit DMA, PCI detection added, and EISA/ISA/TURBOchannel removed. Every fix below is its own commit, and the commit messages give the full reasoning. The design has not changed: 16-bit LANCE mode, SWSTYLE 0, 24-bit descriptor addresses. Since the cold-boot fix below, the DMA memory sits in one 16 MB-aligned window below 4 GB instead of below 16 MB.

This branch sits on top of `master` as of `c723dfb` (merge of #17), so it keeps the fixes already merged there:

- **Low contiguous DMA pool** (#15). The pool is allocated with `MmAllocateContiguousMemorySpecifyCache` below 16 MB, which fixed the Windows 7 "conflict in resources" error. The fixes below free it with `MmFreeContiguousMemory` to match.
- **Interrupt causes kept for the DPC** (#17). The ISR acknowledges CSR0 and saves the causes; the DPC collects them under `NdisMSynchronizeWithInterrupt`. Fix 5 below keeps this design and closes the remaining gaps.

**Windows 7 DHCP failure** ("unidentified network", 169.254.x.x address): fix 12 is the likely cause. After a packet-filter or multicast change, the driver never told NDIS to resume sending, so the DHCP request never went out. Windows 7 makes several of those changes as the interface comes up; Server 2003 does not.

**Windows 7 x64 cold boot, Code 10** (#53, #55): every DMA allocation was limited to the first 16 MB of RAM. Early in a cold boot on x64 that memory is mostly in use, so the allocation failed. A disable/enable or a warm reboot happened to find room, which is why those worked. The chip doesn't need memory below 16 MB. In 16-bit mode the PCnet-PCI chips take bits 31:24 of every bus-master address (init block, rings and buffers) from CSR2[15:8] (IADR[31:24]). So the driver now:

- allocates all of its DMA memory as one block of about 60 KB: init block, rings, receive buffers and transmit buffers. Before, it made 101 separate page-sized allocations, about 400 KB in all;
- places that block anywhere below 4 GB, without crossing a 16 MB boundary;
- writes the full upper 16 address bits to CSR2 (it used to write 0 in bits 15:8).

Map registers and bounce buffers (`NdisMAllocateSharedMemory`, as #55 suggests) can't help here. The chip reads and writes the init block, rings and buffers directly, so they have to be at addresses it can reach.

**Audit follow-ups** (after #60), one commit each, with the details in the commit messages:

- A reset set up while `LanceHalt` ran on another CPU made the DPC loop forever, hanging the disable.
- A reset started by the driver itself (memory error, transmitter stopped, filter or multicast change) never finished if its IDON interrupt was lost. The DPC now also checks CSR0 for IDON, and `LanceCheckForHang` runs the DPC every 2 seconds while a reset is running.
- The transmit and receive statistics OIDs reported the wrong counters, and `OID_GEN_VENDOR_ID` returned an uninitialized byte.
- `BytesWritten`, `BytesRead` and `BytesNeeded` were not always filled in.
- The driver failed to load on Windows XP before SP2 because it imported `KeFlushQueuedDpcs`. It now looks it up at run time.
- A `NetworkAddress` override with the group bit set was used as the station address. It is now ignored.

## Build results

Built with WDK 7.1.0 (7600.16385.1), using `setenv.bat … fre x86 WNET` and then `build -cZ`.

| | Errors | Warnings | Output |
|---|---|---|---|
| Baseline (`master`, before any change) | 0 | 0 code warnings | `lance.sys`, 14 848 bytes |
| Final (`fix/win7-on-master` HEAD) | 0 | 0 | `lancepci.sys`, 16 384 bytes, SHA-256 `8D9BD55E…4EC3192` |

Warnings are at `/W3 /WX`, and OACR reported nothing. No `.err` or `.wrn` file was produced for either build. The baseline log is in `dist\baseline\` and the final log in `dist\buildfre_wnet_x86.final.log`.

Note on the build environment: `build.exe` counts every PATH entry that does not exist as a "warning" in its summary, without printing any text in the log. On this machine that showed up as "5 Warnings" until the PATH was cleaned. These are not compiler warnings. If you see an unexplained warning count, look for nonexistent folders on PATH, including `C:\WinDDK\7600.16385.1\tools\sdv\bin`.

## Fixes

Each entry gives what was wrong, what changed, and how to verify it. The test numbers refer to the checklist in TESTING.md.

### Install and identity

1. **.gitignore was inactive** (`fa41507`). The file was named `gitignore.txt`, so build output could be committed. It is now `.gitignore` and ignores `obj*`, the build logs and `dist/`. *Verify:* `git status` after a build is clean.

2. **Binary name mismatch** (`56ff762`). `TARGETNAME` built `lance.sys`, while the INF expected a different file and service name. The names now match across `sources`, `lance.rc` (internal and original filename, version 1.0.0.0) and the INF. They were `lancepci.sys` and `LANCEPCI` at the time; the binary is now `pcntn5m.sys` and the service `PCNTN5M`. *Verify:* Driver Details shows `pcntn5m.sys`, and `sc query pcntn5m` works (tests 1–2).

3. **PCI matching too broad, chip unknown** (`5fe44e4`). The driver now accepts only `1022:2000` and `1022:2001`. After reset it reads CSR88/89 and stores the chip ID, part number and name in the adapter. It prints one `pcntn5m: <chip>, chip ID …` line at load. *Verify:* that line appears with the expected chip on both VM adapter types.

4. **INF was not a clean NT x86 INF** (`d11fec1`). The INF was rewritten (then as `lancepci.inf`, now `pcntn5m.inf`), with an `NTx86`-decorated models section listing only the two PCI IDs, and with `Characteristics 0x4`, `BusType 5` and `ndis5`/`ethernet` bindings. It has a single `AddService` and netevent.dll event-log registration. An `NTamd64` models section was added later, once the driver built clean for x64. *Verify:* Have Disk install on ReactOS and Server 2003 (TESTING.md, section 3).

### Runtime correctness

5. **ISR assumed RAP selected CSR0** (`f6e820e`). Master's ISR (#17) already acknowledges CSR0 and queues the DPC. On a shared PCI line it also runs for other devices' interrupts, including between the "select CSR1/CSR2" and data writes in `SetInitBlockAndInit`. It would then read CSR1 as CSR0 and write the acknowledge value into the init-block address. It also claimed every interrupt once the device was removed (CSR0 reads 0xffff). The ISR now:
   - saves RAP, selects CSR0, and restores RAP before returning;
   - returns "not mine" when CSR0.INTR is clear or the device reads all ones.

   The CSR1/CSR2/INIT sequence runs under `NdisMSynchronizeWithInterrupt`. The first initialization waits for IDON instead of a blind 500 ms stall. Since the cold-boot fix it waits up to 2 seconds and also polls CSR0 every 10 ms, in case the interrupt is late or lost. *Verify:* tests 2–7.

6. **CSR4 unmasked interrupts the driver never clears, and BCR18 was overwritten** (`d37763f`). Writing `CSR4 = 0x4000` cleared the TXSTRT, JAB, MFCO and RCVCCO masks. TXSTRT then latched on the first transmit and held INTR, causing an interrupt storm on a level-triggered line. The driver now writes `0x4115` (DMAPLUS plus the reset-default masks). `BCR18 = 0x0060` wiped ROMTMG, LINBC and the other fields; BCR18 is now read-modify-written to set BREADE|BWRITE on the 79C970A and later only. Both writes happen once, after reset. *Verify:* test 7 with no CPU spike; in the debugger, CSR4 reads 0x4115 (or with status bits set above it).

7. **No WIO/DWIO check** (`84912ee`). The register offsets assume 16-bit WIO mode. If anything left the chip in DWIO mode (which only a hardware reset clears), every access was undefined. After reset the driver now checks the RAP read-back and CSR0 == STOP. If either check fails, it logs an event, prints a DbgPrint naming the cause, and fails init cleanly. *Verify:* the load line appears and there is no "does not answer in 16-bit WIO mode" message.

8. **Init failure freed DMA memory under a running chip** (`ef6f7e3`). The failure path now stops the chip, deregisters the interrupt and cancels the timer, and only then frees memory. The deferred timer is now initialized before the interrupt is connected. The WIO-check failure path from fix 7 also frees the pool with `MmFreeContiguousMemory`, matching how master allocates it. *Verify:* hard to provoke in a VM. Code review only.

9. **NdisMDeregisterInterrupt called from the DPC** (`418e985`). When the receive ring wrapped with no end of packet, the driver deregistered the interrupt at DISPATCH_LEVEL. LanceHalt then deregistered it a second time, and the adapter stayed dead. This case now logs an event and does a normal MAC-issued reset. *Verify:* code review. A healthy chip does not hit this.

10. **NDIS version mismatch, no shutdown handler** (`7b35205`). The build defined NDIS 5.0 (plus the protocol-driver `NDIS50` define) while the target is NDIS 5.1. The driver is now `NDIS51_MINIPORT`, version 5.1, with:
    - `AdapterShutdownHandler`: stops the chip; touches only the chip, so it is safe at high IRQL;
    - `PnPEventNotifyHandler`: records surprise removal.

    The INF keeps `ndis5` (the binding interface for every 5.x driver). *Verify:* tests 10–11. `OID_GEN_DRIVER_VERSION` returns 0x0501.

11. **LanceHalt teardown order** (`dc18780`). Halt stopped the chip without synchronizing with the ISR, never cancelled the deferred timer, and touched hardware after a surprise removal. The new order is:
    1. Stop the chip (synchronized).
    2. Deregister the interrupt.
    3. Cancel the timer.
    4. Free the shared memory, adapter buffers and map registers.
    5. Deregister the I/O range and free the adapter.

    *Verify:* test 9 (5× disable/enable without a bugcheck).

12. **Send stalls after a reset, and lost set completions** (`0be405c`). This is the likely cause of the Windows 7 DHCP failure. Every packet-filter or multicast change reinitializes the chip, and sends offered during it get RESOURCES. Nothing then called `NdisMSendResourcesAvailable`, so NDIS's send queue stalled; Windows 7 makes several such changes while it brings the interface up, so its DHCP requests never went out. Separately, a MAC-issued reset during a pending set overwrote the single `ResetRequestType` field, so the set never completed. Two flags now track what each reset owes. On IDON the driver restarts the chip, completes the set, then the reset, then calls `NdisMSendResourcesAvailable`. *Verify:* on Windows 7, `ipconfig /renew` gets an address from DHCP; tests 6–9, and test 12 if a capture tool is available.

13. **Bad packets blocked the send queue** (`4ed4555`). Zero-length or >1514-byte sends returned RESOURCES, which NDIS retries forever. They now return FAILURE. *Verify:* code review.

14. **Multicast table overflow** (`cbf91f4`). The table holds 32 entries, but `MaxMulticastList` could be up to 255 and the OID handler did not check the count. Anything over 32 overwrote the adapter structure. The registry value is now clamped to 32, and longer lists fail with `NDIS_STATUS_MULTICAST_FULL`. *Verify:* code review. Setting `MaximumMulticastList = 100` in the registry should make `OID_802_3_MAXIMUM_LIST_SIZE` report 32.

15. **Wrong LCOL bit** (`13a1838`). The TMD3 late-collision mask was 0x0100 (a TDR bit). It is now 0x1000. *Verify:* code review against the TMD3 layout in the datasheet.

16. **Unchecked receive length** (`2b325a0`). MCNT < 4 or a frame shorter than 14 bytes made the indicated sizes wrap to about 4 GB, and frames over 1514 bytes were indicated as is. All three are now dropped and counted as receive errors. *Verify:* code review. Tests 4 and 6 must still pass at full frame size.

### Reviewed and found correct (no change)

- **24-bit DMA.** Descriptors hold address bits 23:0, with the high byte taken from `(addr >> 16) & 0xff`. Bits 31:24 come from CSR2[15:8], which holds the top byte of the 16 MB window that contains all of the DMA memory (see the cold-boot fix above).
- **Register offsets.** RDP 0x10, RAP 0x12, reset 0x14 and BDP 0x16 are correct for WIO mode.

## Untested

- **Everything except build.** These changes are only reviewed and built. TESTING.md is the plan. The cold-boot changes (#60) and the audit follow-ups have not been run yet.
- **Am79C970, 79C971 (PCnet-FAST), 79C972 (PCnet-FAST+), 79C975 and 79C978 (PCnet-Home).** VirtualBox emulates only the 79C970A and 79C973, so these need real cards. The 79C971/972/975 should behave like the 79C973 in 16-bit mode. The 79C978 is matched on `1022:2001`, but its HomePNA PHY selection is not handled. The original 79C970 is identified by part 0x2420/0x2430, but the 0x2430 mapping is from memory and should be checked against a real chip.
- **Windows Server 2003 x86, Windows XP x86, Windows 7 x86 and x64.** These are supported targets, since all of them load NDIS 5.1 miniports. TESTING.md section 7 has an XP and Windows 7 run of the same checklist.
- **SMP.** The ISR/RAP synchronization is written for SMP but has only been reviewed.

## Out of scope

- **Windows 8, 10 and 11.** These versions no longer load NDIS 5.x miniports. Supporting them means a separate driver: an NDIS 6 miniport, built for x64, and attestation-signed through Microsoft's Hardware Dev Center. That would be a new driver, not a fix to this one.
- **ReactOS upstreaming.** A ReactOS PR is most likely blocked by the license. This code comes from the NT 4.0 DDK sample, and ReactOS needs GPL-compatible code. ReactOS's FastFAT came from Microsoft's MS-PL GitHub samples, which have a different license from this code. ReactOS also already has its own `pcnet` driver.

## Performance

The driver still copies every transmitted packet into a bounce buffer in the shared DMA block, but each buffer pool now has 64 entries to match the transmit ring and reduce send-queue stalls. The receive ring uses 64 full-frame buffers, including room for the four-byte FCS, so each packet needs only one descriptor and the full packet is available as lookahead. The interrupt DPC drains all available receive descriptors instead of stopping after half the ring and waiting for a timer. This avoids timer-dependent receive stalls under sustained downloads. The shared DMA block is about 211 KiB and remains within the chip's 16 MB DMA window. Throughput should be checked with the sustained-transfer tests in TESTING.md; in VirtualBox, the **Intel PRO/1000 MT** adapter type may still be faster and has built-in drivers on XP through Windows 11.

## Known issues not fixed

These are out of scope or need a decision first:

- **Link state is polled, not interrupt-driven.** `LanceCheckForHang` reads BCR4 about every 2 seconds and reports `NDIS_STATUS_MEDIA_CONNECT`/`DISCONNECT`, so an unplugged cable shows up within about 2 seconds. It reports a disconnect only after BCR4 has shown the link up once, so a chip or emulator without a working link bit stays "connected" as before. It skips the original 79C970.
- **No hang detection.** `LanceCheckForHang` always returns FALSE, so a wedged chip is only reset when NDIS sees a send or request time out. While a reset is running it does run the DPC, so a reset whose IDON interrupt was lost still finishes.
- **64-bit.** The build is clean for `fre x64 WIN7`, and the 8-byte write into `IoPortBase` is fixed. `IoPortBase` is still a `ULONG`, which holds an x86/x64 I/O port number. x64 Windows only loads drivers that are signed (or test-signed with test signing on).
- **Filter and multicast changes drop queued sends.** Each change stops and reinitializes the chip, which discards frames still waiting on the transmit ring, although NDIS was already told they were sent. Protocols retry, but at interface startup this can cost a DHCP or ARP retry of a few seconds. Fixing it means waiting for the transmit ring to drain before stopping the chip.
- **Windows XP before SP2.** `KeFlushQueuedDpcs` doesn't exist there, so `LanceHalt` waits 20 ms instead, which is not a guarantee on a multi-processor system.
- **NDIS-issued reset during a pending set.** The pending set is not completed, on the assumption that NDIS aborts it, as the original NT4 driver assumed. This is worth confirming on ReactOS NDIS.
- **Interrupt DPC vs. timer.** The deferred timer callback calls the same DPC routine. This relies on NDIS serializing timer callbacks with the interrupt DPC for a serialized miniport, which NDIS does on Windows. ReactOS should be confirmed.
- **The driver is unsigned.** Server 2003, XP and Windows 7 x86 all warn at install, but all three allow it.
