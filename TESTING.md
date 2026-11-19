# Testing lancepci.sys

How to test the driver in a ReactOS x86 VM under VirtualBox. Use the same steps for Windows Server 2003 x86; only the debugger boot switch differs.

Test both emulated chips, one after the other:

| Pass | VirtualBox adapter type | Chip | Expected chip ID line |
|---|---|---|---|
| 1 | PCnet-FAST III (Am79C973) | 79C973 | `Am79C973 (PCnet-FAST III)` |
| 2 | PCnet-PCI II (Am79C970A) | 79C970A | `Am79C970A (PCnet-PCI II)` |

## 1. What to install

From `dist\` in the repository (built by `build -cZ`, gitignored):

- `lancepci.sys`: the driver, file version 1.0.0.0
- `lancepci.inf`: the INF, CRLF line endings

Copy both into the same folder in the guest. The simplest way is to put them on a small ISO image (any ISO tool works, e.g. `oscdimg -n dist lancepci.iso`) and attach it as a CD in the VM settings. A VirtualBox shared folder also works if Guest Additions are installed.

## 2. VM setup

1. Create a ReactOS VM (type *Other/Unknown* or *Windows 2003*, 32-bit), at least 256 MB RAM, IDE disk, I/O APIC off. Install a current ReactOS 0.4.x release or nightly.
2. **Settings → Network → Adapter 1**: enabled, *Attached to: NAT*, **Advanced → Adapter Type: PCnet-FAST III (Am79C973)**. Leave *Cable Connected* ticked.
3. **Settings → Serial Ports → Port 1** (for the debugger, see section 4):
   - Enable Serial Port, *Port Number: COM1*
   - *Port Mode: Host Pipe*
   - Leave *Connect to existing pipe/socket* **unticked** (VirtualBox creates the pipe)
   - *Path/Address:* `\\.\pipe\reactos`

For pass 2, power the VM off and change the adapter type to **PCnet-PCI II (Am79C970A)**. Nothing else changes.

## 3. Installing the driver (and making sure it is ours)

ReactOS ships its own `pcnet.sys` for `PCI\VEN_1022&DEV_2000`. Windows Server 2003 has AMD's own driver for the same ID. The in-box driver is installed during setup and generally ranks at least as well as ours, so **the driver must be picked explicitly with Have Disk**:

1. Open Device Manager: *Start → Run → `devmgmt.msc`*.
2. *Network adapters* → the AMD PCnet adapter → *Properties → Driver → Update Driver…*
3. Choose *Install from a list or specific location* → *Don't search. I will choose the driver to install* → **Have Disk…** → browse to the folder with `lancepci.inf`.
4. Pick **AMD PCnet-PCI/FAST Ethernet Adapter (lancepci)** and finish. The driver is unsigned, so accept the warning on Server 2003.
5. Reboot if asked.

Confirm the adapter is running on **lancepci.sys**, not the in-box driver:

- Device Manager → adapter → *Properties → Driver → Driver Details* must list `…\system32\drivers\lancepci.sys`. The adapter name ends in "(lancepci)".
- *Properties → General* shows "This device is working properly".
- At a command prompt, `sc query lancepci` shows `STATE: 4 RUNNING`.
- In the debug output (section 4) at load time:

  ```
  lancepci: Am79C973 (PCnet-FAST III), chip ID 0x........, PCI 1022:2000, I/O 0x...., IRQ ..
  ```

  If instead you see `lancepci: I/O 0x.... does not answer in 16-bit WIO mode after reset`, something left the chip in 32-bit DWIO mode. That usually means another PCnet driver ran earlier in this boot. Power the VM off and on (a full power cycle, not a restart) and try again.

## 4. Kernel debugging over the serial pipe

### Windows Server 2003 guest, or a ReactOS build with the KD (WinDbg) protocol

Add `/DEBUG /DEBUGPORT=COM1 /BAUDRATE=115200` to the boot entry: `boot.ini` on Server 2003, `freeldr.ini` on ReactOS.

Start the VM first, so the pipe exists. Then, on the host:

```
C:\WinDDK\7600.16385.1\Debuggers\windbg.exe -k com:pipe,port=\\.\pipe\reactos,resets=0,reconnect
```

Useful commands once connected:

- `ed nt!Kd_DEFAULT_Mask 0xf`: Server 2003 filters DbgPrint output by default; this shows the `lancepci:` lines.
- `!ndiskd.miniports`, then `!ndiskd.miniport <addr>`: on Server 2003 only. ReactOS's NDIS does not support the ndiskd extension.
- `lm m lancepci`: confirms the module is loaded and shows its base address.
- `bp lancepci!LanceIsr` / `bp lancepci!LanceHandleInterrupt`: if no traffic flows, these show whether interrupts arrive at all. Clear them afterwards; they fire on every packet.

Symbols: `objfre_wnet_x86\i386\lancepci.pdb`. Use `.sympath+ C:\tasks\PCnet\objfre_wnet_x86\i386` and then `.reload /f lancepci.sys`.

### ReactOS with its built-in KDBG (the default for release and nightly builds)

Standard ReactOS builds use KDBG rather than the WinDbg protocol, so WinDbg will not connect. ReactOS still writes all DbgPrint output (including the `lancepci:` lines) to COM1 when booted with the *ReactOS (Debug)* boot menu entry.

To read it, either:
- set the serial port mode to **Raw File**, which writes everything to a log file on the host, or
- keep the host pipe and attach a terminal program that can open named pipes.

On a crash, KDBG drops to its own prompt on the serial port; `bt` shows the stack.

## 5. Test checklist

Run every item on both adapter types. Write down the result and anything odd in the debug output.

| # | Test | How | Pass when |
|---|---|---|---|
| 1 | Driver loads | Boot after install | Chip ID line in debug output; Device Manager shows no error; `sc query lancepci` reports RUNNING |
| 2 | Link up | `ipconfig /all` | Adapter listed with the VM's MAC address (VirtualBox shows it under Adapter 1 → Advanced) |
| 3 | DHCP | `ipconfig /release`, then `ipconfig /renew` | Gets 10.0.2.15 from the NAT DHCP server |
| 4 | Ping | `ping 10.0.2.2`, then `ping -n 100 -l 1472 10.0.2.2` | 0% loss. The 1472-byte payload gives a full 1514-byte frame |
| 5 | DNS / outside | `ping www.reactos.org` (if ICMP passes the host's NAT), or open a web page | Name resolves, replies or page load |
| 6 | Large transfer | Download a file of 100 MB or more (browser, `wget`, or SMB from the host), or copy a large file to/from a host share | Completes; the hash matches the source; no stalls longer than a few seconds |
| 7 | Sustained load | Run test 6 while `ping -t 10.0.2.2` runs | No "Request timed out" bursts, no hang, no bugcheck |
| 8 | Cable unplug | VM *Settings → Network → Adapter 1 → Cable Connected* off (or the network icon in the VM status bar → *Connect Network Adapter*) for 30 s, then back on | Pings fail while unplugged and resume by themselves after replug. No bugcheck, adapter stays enabled. (The driver does not report media-disconnect status; see CHANGES.md.) |
| 9 | Disable / enable | Device Manager → adapter → *Disable*, then *Enable*. Repeat 5 times | Each enable reloads the driver (chip ID line again), and DHCP and ping work again. No bugcheck on disable, which exercises LanceHalt |
| 10 | Reboot | Restart from the Start menu, 3 times | Clean shutdown (LanceShutdown stops the chip), and after the next boot tests 1–4 pass |
| 11 | Cold boot | Power off, then on | Same as 10 |
| 12 | Promiscuous / multicast (optional) | Run a packet capture tool, if one is available for the guest | Capture starts and stops without breaking connectivity. That exercises the filter-change reset path |

If a test fails, keep the debug output and note the test number. A `!analyze -v` (WinDbg) or `bt` (KDBG) from any bugcheck is the most useful thing to send back.

## 6. Not covered by the VM

VirtualBox only emulates the 79C970A and 79C973. The 79C970, 79C971 (PCnet-FAST), 79C972 (PCnet-FAST+), 79C975 and 79C978 need real cards. CHANGES.md lists what is untested.
