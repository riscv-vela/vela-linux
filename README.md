# vela-linux

Linux kernel components of the [Vela-OS](https://github.com/riscv-vela/vela)
, running on the [Q-Vela](https://github.com/riscv-vela/q-vela)
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

---

## RAS — Kernel Support

RERI support is already applied in `linux/` — there is no patch step, and the
required kernel options are all in the riscv defconfig. The build follows
`## Building` above.

### Required kernel options

| Option | Where it comes from | After a plain `make defconfig` |
|--------|---------------------|-------------------------------|
| `CONFIG_ACPI_APEI` | `arch/riscv/configs/defconfig` | enabled |
| `CONFIG_ACPI_APEI_GHES` | `arch/riscv/configs/defconfig` | enabled |
| `CONFIG_RISCV_SBI_SSE` | `drivers/firmware/riscv/Kconfig` (`default y`) | enabled |
| `CONFIG_ACPI_APEI_SSE` | `drivers/acpi/apei/Kconfig` (`default y`) | enabled |
| **`CONFIG_RAS`** | `arch/riscv/configs/defconfig` | enabled |

`CONFIG_RAS` is a `menuconfig` symbol with no `default` clause and nothing
selects it — `ACPI_APEI_GHES` does not, and `MEMORY_FAILURE`/`PCIEAER` do not
apply on riscv. It is therefore listed explicitly in the riscv defconfig.

### Why `CONFIG_RAS` is required

A RERI error is delivered as a CPER **generic processor** section. In
`ghes_do_proc()` that section matches none of the specialised branches and falls
through to `log_non_standard_event()`, which emits the `ras:non_standard_event`
tracepoint — the interface rasdaemon subscribes to.

With `CONFIG_RAS=n` that function is an empty inline stub
(`include/linux/ras.h`) and `drivers/ras/ras.o` is never built, so the
tracepoint does not exist. The `[Hardware Error]` dmesg report still appears,
because it is produced by a separate `cper_print_*` path — which is why the gap
is easy to miss.

### Build

```bash
cd linux
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
grep '^CONFIG_RAS=' .config          # must print CONFIG_RAS=y

make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j$(nproc) Image
cd ..
```

The result can be checked before booting — with `CONFIG_RAS=n` these symbols do
not exist at all:

```bash
grep non_standard_event linux/System.map
```

### Kernel version

**v6.12 or newer.** The RISC-V ACPI RINTC model required to boot this platform
with `aia=aplic-imsic` landed in v6.12-rc1; v6.8 panics during boot. This tree
is 6.18.0-rc5.

### In the guest

RERI reporting goes through ACPI (HEST/GHES), so the guest must be booted with
the EDK2 UEFI firmware from
[vela-fws](https://github.com/riscv-vela/vela-fws) and
`-M virt,reri=true,aia=aplic-imsic`. A device-tree boot produces no reports.

`tracefs` is not always mounted automatically — a buildroot initramfs does not
mount it:

```sh
mount -t tracefs nodev /sys/kernel/tracing
ls /sys/kernel/tracing/events/ras/       # non_standard_event
```

See [q-vela `docs/ras.md`](https://github.com/riscv-vela/q-vela/blob/main/docs/ras.md)
for the launch command and the RERI injection procedure.
