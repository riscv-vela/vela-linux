# vela-linux

Linux kernel components of the [Vela-OS](https://github.com/riscv-vela/vela)
project, running on the [Q-Vela](https://github.com/riscv-vela/q-vela)
emulation platform. 

| Subsystem | Contents |
|-----------|----------|
| **TEE** (Keystone) | Keystone kernel driver — the host-side interface for creating and running enclaves. Enclave isolation is enforced with **RISC-V WorldGuard** instead of Keystone's original PMP-based scheme. |
| **RAS** (RERI) | Kernel support for the RISC-V Error Record Interface |


---

## Repository Structure

```
vela-linux/
├── linux/                     # Linux kernel with the Vela patches applied
├── linux-keystone-driver/     # Keystone kernel driver (/dev/keystone_enclave)
└── sdk/include/               # Shared Keystone headers the driver builds against
```

**Kernel version:** `e9a6fb0bcdd7609be6969112f3fbfcce3b1d4a7c` (6.18.0-rc5)

---

## Building

### Prerequisites

```bash
sudo apt install build-essential flex bison bc libssl-dev libelf-dev
```

A RISC-V cross toolchain is also required.

### 1. Build the kernel

```bash
cd linux
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j$(nproc) Image modules
cd ..
```

Enabling `CONFIG_CMA` and `CONFIG_DMA_CMA` is recommended: the Keystone driver
allocates enclave memory from the buddy allocator and falls back to CMA when a
contiguous region larger than `MAX_PAGE_ORDER` is requested.

### 2. Build the Keystone driver

The driver is built as an out-of-tree module against the kernel from step 1.

```bash
cd linux-keystone-driver
make
cd ..
```

`LINUXSRC` and `KEYSTONE_SDK_DIR` default to `../linux` and `../sdk`, so no
arguments are needed for an in-tree layout. The driver Makefile invokes the
kernel build with `CROSS_COMPILE=riscv64-linux-gnu-`; if your kernel was built
with a different prefix, edit it or pass `CROSS_COMPILE=` explicitly to keep
both consistent.

### Build Output

| File | Description |
|------|-------------|
| `linux/arch/riscv/boot/Image` | S-mode kernel image, passed to QEMU via `-kernel` |
| `linux-keystone-driver/keystone-driver.ko` | Keystone driver module; `insmod` creates `/dev/keystone_enclave` |

See the [q-vela](https://github.com/riscv-vela/q-vela) README for the launch
command, and [vela-apps](https://github.com/riscv-vela/vela-apps) for the
enclave applications that run against this driver.
