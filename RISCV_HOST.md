
# Tenstorrent PCIe Accelerators with RISC-V Host

## Components
### create-ethernet-map
* Part of [Luwen](https://github.com/tenstorrent/luwen); it's written in Rust
* It is a tool for producing a YAML document that describes the particulars of a chip topology
* Not super important for a single-PCIe card configuration (i.e. there is no topology), although:
  * N300 card contains a "remote" chip (only reachable via Ethernet; not visible to the PCIe driver)
  * The YAML document contains "harvesting" information required by the user driver
* Should just work -- see example below
```
ubuntu@p550 ~/git/luwen $ git rev-parse HEAD
5cf4cc6545b7d9d92f167417faf39822ed88dbf7

ubuntu@p550 ~/git/luwen $ cargo --version
cargo 1.82.0 (8f40fc59f 2024-08-21)

ubuntu@p550 ~/git/luwen $ rustc --version
rustc 1.82.0 (f6e511eec 2024-10-15)

ubuntu@p550 ~/git/luwen $ cargo build --release --bin luwen-cem
    <snip: omitted for brevity>
    Finished `release` profile [optimized] target(s) in 2m 50s

ubuntu@p550 ~/git/luwen $ file target/release/luwen-cem
target/release/luwen-cem: ELF 64-bit LSB pie executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-riscv64-lp64d.so.1, BuildID[sha1]=95ba1413d70ab11c2a70a8b843ba36b354a7a486, for GNU/Linux 4.15.0, not stripped

ubuntu@p550 ~/git/luwen $ target/release/luwen-cem output.yaml
  Detecting chips (found 1)

ubuntu@p550 ~/git/luwen $ cat output.yaml
arch: {
   0: Wormhole,
}

chips: {
   0: [0,0,0,0],
}

ethernet_connections: [
]

chips_with_mmio: [
   0: 0,
]

# harvest_mask is the bit indicating which tensix row is harvested. So bit 0 = first tensix row; bit 1 = second tensix row etc...
harvesting: {
   0: {noc_translation: true, harvest_mask: 8},
}

# This value will be null if the boardtype is unknown, should never happen in practice but to be defensive it would be useful to throw an error on this case.
boardtype: {
   0: n150,
}
```

### [tt-smi](https://github.com/tenstorrent/tt-smi)
* Installing into venv should just work
```
python3 -m venv ~/venv
. ~/venv/bin/activate
pip install git+https://github.com/tenstorrent/tt-smi
```

### [tt-kmd](https://github.com/tenstorrent/tt-kmd)
* Kernel driver
* Should just work

### [tt-umd](https://github.com/tenstorrent/tt-umd)
* Usermode driver
* Requires touches around `create-ethernet-map` use under RISC-V, but otherwise should just work
* [This branch](https://github.com/tenstorrent/tt-umd/tree/joelsmith/riscv64-build) has a `create-ethernet-map binary` for RISC-V

### [tt-metal](https://github.com/tenstorrent/tt-metal/)
* `main` contains X86-specific SIMD code and will not build under RISC-V
* [This branch](https://github.com/tenstorrent/tt-metal/tree/joelsmith/riscv64-build) has scalar implementations and compiles under RISC-V (caveat emptor: there may be bugs)
* At runtime, Metal invokes custom GCC binaries; you will need to build RISC-V versions.  Check out [sfpi](https://github.com/tenstorrent/sfpi) and follow steps 1-4.  Then adjust `$TT_METAL_HOME/tt_metal/third_party/sfpi/compiler` directory.  I renamed it and created a symlink, see below:
```
ubuntu@p550 ~/git/tt-metal/tt_metal/third_party/sfpi $ ll
total 44
drwxr-xr-x  5 ubuntu ubuntu  4096 Oct 27 13:56 ./
drwxr-xr-x 12 ubuntu ubuntu  4096 Oct 26 18:44 ../
-rw-r--r--  1 ubuntu ubuntu    48 Oct 26 18:47 .git
-rw-r--r--  1 ubuntu ubuntu  3882 Oct 26 18:48 .gitattributes
-rw-r--r--  1 ubuntu ubuntu 11346 Oct 26 18:48 LICENSE
-rw-r--r--  1 ubuntu ubuntu    12 Oct 26 18:48 README.md
lrwxrwxrwx  1 ubuntu ubuntu    30 Oct 27 13:56 compiler -> /home/ubuntu/git/sfpi/compiler/
drwxr-xr-x  9 ubuntu ubuntu  4096 Oct 26 18:48 compiler.x86/
drwxr-xr-x  5 ubuntu ubuntu  4096 Oct 26 18:48 include/
drwxr-xr-x  2 ubuntu ubuntu  4096 Oct 26 18:48 tools/
```

## Systems

### SiFive Premier P550
* Fastest (?) commercially-available RISC-V development board as of 11/2024, but no RVV 1.0
* Has PCIe
* Has IOMMU
* Successfully built all Tenstorrent code (tt-kmd, tt-umd, tt-metal, luwen), but Python dependencies for tt-metal not fully worked out under Ubuntu 24.04
* Card is visible, driver loads, `tt-smi -r0` works
```
ubuntu@p550 ~/git/luwen $ lscpu
Architecture:          riscv64
  Byte Order:          Little Endian
CPU(s):                4
  On-line CPU(s) list: 0-3
NUMA:
  NUMA node(s):        1
  NUMA node0 CPU(s):   0-3
ubuntu@p550 ~/git/luwen $ uname -a
Linux p550 6.6.21-5-premier #1 SMP PREEMPT_DYNAMIC Fri Oct 11 05:09:42 UTC 2024 riscv64 riscv64 riscv64 GNU/Linux
ubuntu@p550 ~/git/luwen $ sudo lspci -v -s 01:00.0
01:00.0 Processing accelerators: Tenstorrent Inc Wormhole (rev 01)
	Subsystem: Tenstorrent Inc n150
	Flags: bus master, fast devsel, latency 0, IRQ 89, NUMA node 0, IOMMU group 5
	Memory at 8000000000 (64-bit, prefetchable) [size=512M]
	Memory at 41100000 (32-bit, non-prefetchable) [size=1M]
	Memory at 8020000000 (64-bit, prefetchable) [size=32M]
	Capabilities: [40] Power Management version 3
	Capabilities: [50] MSI: Enable+ Count=1/32 Maskable+ 64bit+
	Capabilities: [70] Express Endpoint, MSI 00
	Capabilities: [100] Advanced Error Reporting
	Capabilities: [148] Secondary PCI Express
	Capabilities: [178] Physical Layer 16.0 GT/s <?>
	Capabilities: [1a8] Lane Margining at the Receiver <?>
	Capabilities: [200] Vendor Specific Information: ID=0002 Rev=4 Len=100 <?>
	Capabilities: [300] Vendor Specific Information: ID=0001 Rev=1 Len=038 <?>
	Capabilities: [338] Data Link Feature <?>
	Kernel driver in use: tenstorrent
```
* DMA is broken; [this test fails](https://github.com/tenstorrent/tt-umd/blob/1590454f4030da9556c8748bae966aabd3883752/tests/wormhole/test_silicon_driver_wh.cpp#L710).  It [fails on Grayskull](https://github.com/tenstorrent/tt-umd/blob/bd00f4ed18dc07b563ab4ced403c93dc6da832bb/tests/grayskull/test_silicon_driver.cpp#L455), too.  The card reads all ones.  Unclear if this is a host problem or a device problem.
* Debugging DMA is ongoing work:
  * Disabling system IOMMU `iommu.passthrough=1` results in unstable system
  * Wrote a small VFIO user driver for one of [these guys](https://shop.lambdaconcept.com/home/50-screamer-pcie-squirrel.html), idea was to test DMA that way -- but `vfio-pci` can't probe the card.  Turns out `vfio-pci` doesn't work with any other PCIe card I've tried either
  * DMA must work in some form, since a 10GbE NIC functions normally with the board

### SiFive Unmatched
* Has PCIe slot
* SoC has poor performance
* Abandoned due to performance and stability issues around PCIe device enumeration

### VisionFive 2
* M2 to PCIe slot adapter allows the board to a full size PCIe card
* SoC has poor performance
* Abandoned work due to PCIe stability problems

### Qemu

* Successfully ran a simple "canned" workload on a [deprecated](https://github.com/tenstorrent/tt-budabackend) software stack
* I did not save any of this work but have some notes for achieving the PCIe passthrough part, reproduced below

## Qemu PCIe Passthrough
* PCIe passthrough from X86 host to RISC-V guest requires [patched](https://lf-rise.atlassian.net/wiki/spaces/HOME/pages/8585526/SE_01_005+-+QEMU+PCIe+passthru+on+x86+hosts) Qemu
```
git clone https://github.com/patchew-project/qemu.git
cd qemu; git checkout patchew/20230731015317.1026996-1-fei2.wu@intel.com
```
* Enable system IOMMU
* Helpful: https://www.kernel.org/doc/Documentation/vfio.txt
* Bind PCIe device(s) to vfio-pci driver.  Example:
```
echo "0000:c1:00.0" > /sys/bus/pci/drivers/tenstorrent/unbind
echo "vfio-pci" > /sys/bus/pci/devices/0000\:c1\:00.0/driver_override
echo "0000:c1:00.0" > /sys/bus/pci/drivers/vfio-pci/bind
```
* Use `-device vfio-pci,host=<PCIe bus address as seen by host>` when launching `qemu-system-riscv64`
* Quick recipe for getting started _without passthrough_ (caveat emptor: this is old):
```
sudo apt-get update
sudo apt-get install opensbi qemu-system-misc u-boot-qemu wget xz-utils

# Download and extract a pre-built RISC-V Ubuntu image:
wget https://cdimage.ubuntu.com/releases/22.04.2/release/ubuntu-22.04.3-preinstalled-server-riscv64+unmatched.img.xz
xz -d ubuntu-22.04.3-preinstalled-server-riscv64+unmatched.img.xz

# Optional: expand the image
qemu-img resize -f raw ubuntu-22.04.3-preinstalled-server-riscv64+unmatched.img +5G

# Launch QEMU
qemu-system-riscv64 \
  -machine virt \
  -nographic \
  -m 2048 \
  -smp 4 \
  -bios /usr/lib/riscv64-linux-gnu/opensbi/generic/fw_jump.bin \
  -kernel /usr/lib/u-boot/qemu-riscv64_smode/uboot.elf \
  -device virtio-net-device,netdev=eth0 \
  -netdev user,id=eth0,hostfwd=tcp::10022-:22 \
  -device virtio-rng-pci \
  -drive file=ubuntu-22.04.3-preinstalled-server-riscv64+unmatched.img,format=raw,if=virtio
```