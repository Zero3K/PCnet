# Changes on branch `fix/win7-on-master`

The starting point was the NT 4.0 DDK DEC LANCE driver as ported by Copilot: 16-bit LANCE mode (SWSTYLE 0), 24-bit DMA, PCI detection added, and EISA/ISA/TURBOchannel removed. Every fix below is its own commit, and the commit messages give the full reasoning. The design has not changed: 16-bit LANCE mode, SWSTYLE 0, 24-bit DMA through a DMA pool below 16 MB.

This branch sits on top of `master` as of `c723dfb` (merge of #17), so it keeps the fixes already merged there:

- **Low contiguous DMA pool** (#15). The pool is allocated with `MmAllocateContiguousMemorySpecifyCache` below 16 MB, which fixed the Windows 7 "conflict in resources" error. The fixes below free it with `MmFreeContiguousMemory` to match.
- **Interrupt causes kept for the DPC** (#17). The ISR acknowledges CSR0 and saves the causes; the DPC collects them under `NdisMSynchronizeWithInterrupt`. Fix 5 below keeps this design and closes the remaining gaps.

**Windows 7 DHCP failure** ("unidentified network", 169.254.x.x address): fix 12 is the likely cause. After a packet-filter or multicast change, the driver never told NDIS to resume sending, so the DHCP request never went out. Windows 7 makes several of those changes as the interface comes up; Server 2003 does not.

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

2. **Binary name mismatch** (`56ff762`). `TARGETNAME` built `lance.sys`, while the INF expected a different file and service name. The binary is now `lancepci.sys` and the service is `LANCEPCI`, consistently across `sources`, `lance.rc` (internal and original filename, version 1.0.0.0) and the INF. *Verify:* Driver Details shows `lancepci.sys`, and `sc query lancepci` works (tests 1–2).

3. **PCI matching too broad, chip unknown** (`5fe44e4`). The driver now accepts only `1022:2000` and `1022:2001`. After reset it reads CSR88/89 and stores the chip ID, part number and name in the adapter. It prints one `lancepci: <chip>, chip ID …` line at load. *Verify:* that line appears with the expected chip on both VM adapter types.

4. **INF was not a clean NT INF** (`d11fec1`). The INF has `NTx86` and `NTamd64` models for the supported PCI IDs, `Characteristics 0x4`, `BusType 5` and `ndis5`/`ethernet` bindings. It has a single `AddService` and netevent.dll event-log registration. *Verify:* Have Disk install on ReactOS and Server 2003 (TESTING.md, section 3).

### Runtime correctness

5. **ISR assumed RAP selected CSR0** (`f6e820e`). Master's ISR (#17) already acknowledges CSR0 and queues the DPC. On a shared PCI line it also runs for other devices' interrupts, including between the "select CSR1/CSR2" and data writes in `SetInitBlockAndInit`. It would then read CSR1 as CSR0 and write the acknowledge value into the init-block address. It also claimed every interrupt once the device was removed (CSR0 reads 0xffff). The ISR now:
   - saves RAP, selects CSR0, and restores RAP before returning;
   - returns "not mine" when CSR0.INTR is clear or the device reads all ones.

   The CSR1/CSR2/INIT sequence runs under `NdisMSynchronizeWithInterrupt`. The first initialization waits for IDON (up to 500 ms) instead of a blind 500 ms stall. *Verify:* tests 2–7.

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

14. **Multicast table overflow** (`cbf91f4`). The table holds 32 entries, but `MaxMulticastList` could be up to 255 and the OID handler did not check the count. Anything over 32 overwrote the adapter structure. The registry value is now clamped to 32, and longer lists fail with `NDIS_STATUS_MULTICAST_FULL`. *Verify:* code review. Setting `MaxMulticastList = 100` in the registry should make `OID_802_3_MAXIMUM_LIST_SIZE` report 32.

15. **Wrong LCOL bit** (`13a1838`). The TMD3 late-collision mask was 0x0100 (a TDR bit). It is now 0x1000. *Verify:* code review against the TMD3 layout in the datasheet.

16. **Unchecked receive length** (`2b325a0`). MCNT < 4 or a frame shorter than 14 bytes made the indicated sizes wrap to about 4 GB, and frames over 1514 bytes were indicated as is. All three are now dropped and counted as receive errors. *Verify:* code review. Tests 4 and 6 must still pass at full frame size.

17. **Mapped I/O range truncated on x64.** `NdisMRegisterIoPortRange` returns a pointer-sized mapping, but the adapter stored it in a `ULONG` and truncated derived register addresses. The mapped range is now stored as `PVOID`, with derived port addresses stored as `ULONG_PTR`. *Verify:* Windows 7 x64 tests 3, 9 and 10 (TESTING.md, sections 5 and 8).

### Reviewed and found correct (no change)

- **24-bit DMA.** The driver allocates its contiguous DMA pool with a highest acceptable address of 0x00FFFFFF, so it lies entirely below 16 MB. Every descriptor and buffer address is taken from that pool. The high byte comes from `(addr >> 16) & 0xff`, and CSR2[15:8] (the upper address byte in 16-bit mode) is written as 0.
- **Register offsets.** RDP 0x10, RAP 0x12, reset 0x14 and BDP 0x16 are correct for WIO mode.

## Untested

- **Everything except build.** No change on this branch has been run yet. TESTING.md is the plan. (Master itself has been reported working on Server 2003 and getting no DHCP address on Windows 7.)
- **Am79C970, 79C971 (PCnet-FAST), 79C972 (PCnet-FAST+), 79C975 and 79C978 (PCnet-Home).** VirtualBox emulates only the 79C970A and 79C973, so these need real cards. The 79C971/972/975 should behave like the 79C973 in 16-bit mode. The 79C978 is matched on `1022:2001`, but its HomePNA PHY selection is not handled. The original 79C970 is identified by part 0x2420/0x2430, but the 0x2430 mapping is from memory and should be checked against a real chip.
- **Windows Server 2003 x86, Windows XP x86, Windows 7 x86.** These are supported targets, since all three load NDIS 5.1 miniports. TESTING.md section 7 has an XP and Windows 7 run of the same checklist. None of them has been run yet.
- **SMP.** The ISR/RAP synchronization is written for SMP but has only been reviewed.
- **Windows 7 x64.** The mapped I/O port address storage is pointer-width, but this target has not been tested. It requires an x64 build and a signed or test-signed package.

## Out of scope

- **Windows 8, 10 and 11.** These versions no longer load NDIS 5.x miniports. Supporting them means a separate driver: an NDIS 6 miniport, built for x64, and attestation-signed through Microsoft's Hardware Dev Center. That would be a new driver, not a fix to this one.
- **ReactOS upstreaming.** A ReactOS PR is most likely blocked by the license. This code comes from the NT 4.0 DDK sample, and ReactOS needs GPL-compatible code. ReactOS's FastFAT came from Microsoft's MS-PL GitHub samples, which have a different license from this code. ReactOS also already has its own `pcnet` driver.

## Performance

The driver still copies every transmitted packet into a bounce buffer in the shared DMA block, but its pool now has 16 large buffers instead of 4 to reduce send-queue stalls. The receive ring uses 64 512-byte buffers, keeping its total data capacity at 32 KB while reducing a full-size frame from 6 descriptors to 3. Receive processing now handles up to half the ring (32 packets) before resynchronizing with the ISR, reducing synchronization overhead during bursts while keeping each pass bounded. These changes reduce buffer exhaustion and descriptor-processing overhead without exceeding the 64 KB DMA pool. Throughput should be checked with the sustained-transfer tests in TESTING.md; in VirtualBox, the **Intel PRO/1000 MT** adapter type may still be faster and has built-in drivers on XP through Windows 11.

## Known issues not fixed

These are out of scope or need a decision first:

- **No link-state reporting.** There is no `OID_GEN_MEDIA_CONNECT_STATUS` and no `NDIS_STATUS_MEDIA_CONNECT`/`DISCONNECT` indication, so the OS always shows the cable as connected. Fixing this needs a CheckForHang handler that polls the link state (BCR4 LNKST on 79C970A and later). That is new functionality.
- **No CheckForHang handler.** A wedged chip is never detected or reset.
- **`NdisQueryBuffer`** is used in the send and transfer paths. It maps through `MmMapLockedPages` and bugchecks if the mapping fails under low memory. `NdisQueryBufferSafe` is the NDIS 5.1 replacement.
- **Filter and multicast sets fail while a MAC-issued reset is in progress** (they return FAILURE instead of pending). This is rare, but a protocol may report the failure.
- **NDIS-issued reset during a pending set.** The pending set is not completed, on the assumption that NDIS aborts it, as the original NT4 driver assumed. This is worth confirming on ReactOS NDIS.
- **Interrupt DPC vs. timer.** The deferred timer callback calls the same DPC routine. This relies on NDIS serializing timer callbacks with the interrupt DPC for a serialized miniport, which NDIS does on Windows. ReactOS should be confirmed.
- **The driver is unsigned.** Server 2003, XP and Windows 7 x86 all warn at install, but all three allow it.
