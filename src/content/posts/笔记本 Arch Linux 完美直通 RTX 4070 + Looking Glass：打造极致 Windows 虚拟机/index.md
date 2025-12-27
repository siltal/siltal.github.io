---
title: 笔记本 Arch Linux 完美直通 RTX 4070 + Looking Glass：打造极致 Windows 虚拟机
published: 2025-12-28
description: ''
image: ''
tags: ['QEMU','KVM','libvirt']
category: 'Linux'
draft: false 
lang: ''
---


## 前言

作为一名 Arch Linux 用户（KDE Plasma + Wayland），虽然 Linux 已经能满足绝大多数开发和日常需求，但偶尔的游戏娱乐或特定的 Windows 软件依然必不可少。双系统切换繁琐，且打断工作流。

为了在 Linux 下获得原生的 Windows 3D 性能，我决定在搭载 **i9-13900HX + RTX 4070** 的笔记本上折腾 **GPU 直通 (GPU Passthrough)**。

本文记录了从环境准备、底层隔离、VBIOS 处理，到解决最棘手的 **NVIDIA Code 43（笔记本电池检测问题）**，最后通过 **Looking Glass** 实现零延迟画面的完整全过程。

---

## 1. 硬件与软件环境

* **宿主机 OS**: Arch Linux (Kernel 6.x-zen)
* **CPU**: Intel Core i9-13900HX (32 vCores)
* **GPU 1**: Intel UHD Graphics (宿主机使用)
* **GPU 2**: NVIDIA GeForce RTX 4070 Laptop (直通给虚拟机)
* **虚拟化方案**: KVM + QEMU + Libvirt (virt-manager)
* **关键技术**: VFIO, Looking Glass (B7), IVSHMEM
* **系统特性** systemd-boot、UKI、btrfs
---

## 2. 宿主机准备：隔离显卡

笔记本直通的核心是不让宿主机（Linux）接触独显。

### 2.1 启用 IOMMU

编辑 Bootloader 配置，在内核参数中添加：

```ini title="/etc/kernel/cmdline" ins="intel_iommu=on iommu=pt"
root=PARTUUID=...  intel_iommu=on iommu=pt vfio-pci.ids=10de:2820,10de:22bd
```

### 2.2 屏蔽 NVIDIA 驱动并绑定 VFIO

这是为了防止 Linux 加载 `nvidia` 驱动占用显卡。

确保 `MODULES` 列表如下（注意顺序，不要包含 nvidia 相关模块）：

:::tip
在较新的 Linux 内核（6.2 版本以后）中，vfio_virqfd 已经直接编译进了 vfio 模块内核核心中，不再作为一个独立的模块存在。所以 mkinitcpio 找不到它。
:::

```ini title="/etc/mkinitcpio.conf" ins="vfio_pci vfio vfio_iommu_type1" "vfio_virqfd"
MODULES=(vfio_pci vfio vfio_iommu_type1 vfio_virqfd...)

```
获取显卡 ID（运行 `lspci -nn | grep -E 'NVIDIA|AMD'` 查看），例如 `10de:2820` (GPU) 和 `10de:22be` (Audio)。

```bash /\d{2}:\d{2}.\d/ /[0-9a-f]{4}:[0-9a-f]{4}/
[siltal@rainbow ~]$ lspci -nn | grep -E 'NVIDIA|AMD'
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD106M [GeForce RTX 4070 Max-Q / Mobile] [10de:2820] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation AD106M High Definition Audio Controller [10de:22bd] (rev a1)

```

```ini title="/etc/kernel/cmdline" ins="vfio-pci.ids="
root=PARTUUID=... intel_iommu=on iommu=pt vfio-pci.ids=10de:2820,10de:22bd
```

重新生成 initramfs 并重启：

```bash
sudo mkinitcpio -P
reboot
```

重启后检查 `lspci -nnk -s 01:00.0`，确认 `Kernel driver in use` 为 `vfio-pci`。

---

## 3. 虚拟机基础配置

本部分使用libvirt-xml配置虚拟机

```xml

<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">
  <name>win10-gpu</name>
  <uuid>bc2d4448-2855-4cef-b569-99abbd8fadf6</uuid>
  <memory unit="KiB">25165824</memory>
  <currentMemory unit="KiB">25165824</currentMemory>
  <vcpu placement="static">16</vcpu>
  <iothreads>1</iothreads>
  <cputune>
  <!-- 根据CPU性能核与能效核分配绑定 -->
    <vcpupin vcpu="0" cpuset="0"/>
    <vcpupin vcpu="1" cpuset="1"/>
    <vcpupin vcpu="2" cpuset="2"/>
    <vcpupin vcpu="3" cpuset="3"/>
    <vcpupin vcpu="4" cpuset="4"/>
    <vcpupin vcpu="5" cpuset="5"/>
    <vcpupin vcpu="6" cpuset="6"/>
    <vcpupin vcpu="7" cpuset="7"/>
    <vcpupin vcpu="8" cpuset="8"/>
    <vcpupin vcpu="9" cpuset="9"/>
    <vcpupin vcpu="10" cpuset="10"/>
    <vcpupin vcpu="11" cpuset="11"/>
    <vcpupin vcpu="12" cpuset="12"/>
    <vcpupin vcpu="13" cpuset="13"/>
    <vcpupin vcpu="14" cpuset="14"/>
    <vcpupin vcpu="15" cpuset="15"/>
    <emulatorpin cpuset="16-31"/>
    <iothreadpin iothread="1" cpuset="16-31"/>
  </cputune>
  <os firmware="efi">
    <type arch="x86_64" machine="pc-q35-7.2">hvm</type>
    <firmware>
      <feature enabled="no" name="enrolled-keys"/>
      <feature enabled="no" name="secure-boot"/>
    </firmware>
    <!-- UEFI启动固件 -->
    <loader readonly="yes" type="pflash" format="raw">/usr/share/edk2/x64/OVMF_CODE.4m.fd</loader>
    <nvram template="/usr/share/edk2/x64/OVMF_VARS.4m.fd" templateFormat="raw" format="raw">/var/lib/libvirt/qemu/nvram/win10_VARS.fd</nvram>
  </os>
  <features>
    <acpi/>
    <apic/>
    <hyperv mode="custom">
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vpindex state="on"/>
      <synic state="on"/>
      <stimer state="on"/>
      <reset state="on"/>
      <vendor_id state="on" value="GenuineIntel"/>
    </hyperv>
    <kvm>
      <hidden state="on"/>
    </kvm>
    <ioapic driver="kvm"/>
  </features>
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" clusters="1" cores="8" threads="2"/>
  </cpu>
  <clock offset="localtime">
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
    <timer name="hypervclock" present="yes"/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type="file" device="disk">
      <driver name="qemu" type="qcow2" cache="none" io="native" discard="unmap"/>
      <!-- 磁盘映射 -->
      <source file="/var/lib/libvirt/images/win10.qcow2"/>
      <target dev="vda" bus="virtio"/>
      <boot order="1"/>
      <address type="pci" domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
    </disk>
    <disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <!-- 安装盘 -->
      <source file="/var/lib/libvirt/images/Win10_22H2_Chinese_Simplified_x64v1.iso"/>
      <target dev="sdb" bus="sata"/>
      <readonly/>
      <boot order="2"/>
      <address type="drive" controller="0" bus="0" target="0" unit="1"/>
    </disk>
    <disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <!-- 优化工具 -->
      <source file="/var/lib/libvirt/images/virtio-win.iso"/>
      <target dev="sdc" bus="sata"/>
      <readonly/>
      <address type="drive" controller="0" bus="0" target="0" unit="2"/>
    </disk>
    <controller type="usb" index="0" model="qemu-xhci">
      <address type="pci" domain="0x0000" bus="0x02" slot="0x00" function="0x0"/>
    </controller>
    <controller type="sata" index="0">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1f" function="0x2"/>
    </controller>
    <controller type="pci" index="0" model="pcie-root"/>
    <controller type="pci" index="1" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="1" port="0x10"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="2" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="2" port="0x11"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x1"/>
    </controller>
    <controller type="pci" index="3" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="3" port="0x12"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x2"/>
    </controller>
    <controller type="pci" index="4" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="4" port="0x13"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x3"/>
    </controller>
    <controller type="pci" index="5" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="5" port="0x14"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x4"/>
    </controller>
    <controller type="pci" index="6" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="6" port="0x15"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x5"/>
    </controller>
    <controller type="pci" index="7" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="7" port="0x16"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x6"/>
    </controller>
    <controller type="pci" index="8" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="8" port="0x17"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x7"/>
    </controller>
    <controller type="pci" index="9" model="pcie-to-pci-bridge">
      <model name="pcie-pci-bridge"/>
      <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
    </controller>
    <interface type="network">
      <mac address="52:54:00:e9:22:99"/>
      <source network="default"/>
      <model type="virtio"/>
      <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
    </interface>
    <input type="mouse" bus="ps2"/>
    <input type="keyboard" bus="ps2"/>
    <graphics type="spice" autoport="yes">
      <listen type="address"/>
    </graphics>
    <audio id="1" type="spice"/>
    <video>
      <model type="qxl" ram="65536" vram="65536" vgamem="16384" heads="1" primary="yes"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
    </video>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
      </source>
      <!-- 从https://www.techpowerup.com/vgabios/ 下载对应显卡驱动，并删除字节直到55aa -->
      <rom file="/var/lib/libvirt/vbios/patched_4070.rom"/>
      <address type="pci" domain="0x0000" bus="0x05" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x1"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="usb" managed="yes">
      <source>
        <vendor id="0x5986"/>
        <product id="0x9102"/>
      </source>
      <address type="usb" bus="0" port="1"/>
    </hostdev>
    <watchdog model="itco" action="reset"/>
    <memballoon model="virtio">
      <address type="pci" domain="0x0000" bus="0x04" slot="0x00" function="0x0"/>
    </memballoon>
  </devices>
</domain>

```

3.1 CPU 核心绑定与拓扑 (`<cputune>` & `<cpu>`)
这是确保虚拟机性能、减少掉帧的关键配置。

`<vcpupin>` (来源)：在 Linux 终端运行 lscpu -e 或 lstopo 查看 CPU 拓扑。

i9-13900HX 提示：该 CPU 拥有性能核（P-Core）和能效核（E-Core）。

策略：我们将 vcpu 0-15（前 8 个 P-Core 及其超线程）分配给虚拟机进行重负载计算，而将剩下的 cpuset 16-31（E-Core）留给宿主机进程（emulator）和磁盘 IO 处理，防止宿主机抢占导致卡顿。

`<topology>`：这里的 cores="8" threads="2" 对应了你分配的 16 个 vCPU。

3.2 固件与 UEFI 启动 (`<os>`)
显卡直通必须使用 UEFI 模式，不能使用传统的 BIOS。

`<loader>` & `<nvram>` (来源)：这些文件由 edk2-ovmf 包提供。

注意：Arch Linux 下的路径通常为 /usr/share/edk2/x64/。

.4m 后缀：这是较新的 OVMF 格式，确保 CODE（固件本身）和 VARS（保存的 UEFI 设置）版本匹配。

3.3 硬件欺骗与 Hyper-V 增强 (`<features>`)
NVIDIA 驱动如果发现自己在虚拟机中，可能会报错。

`<kvm><hidden state='on'/></kvm>`：隐藏 KVM 的存在，防止驱动程序检测。

`<vendor_id state='on' value='GenuineIntel'/>`：将 Hyper-V 的供应商 ID 改为 Intel 原厂标识，进一步欺骗驱动。

3.4 磁盘与 VirtIO 驱动 (`<disk>`)
为了获得接近原生的磁盘 IO 性能，必须使用 virtio 总线。

`<driver cache='none' io='native'/>`：这是性能最高的磁盘访问模式，直接绕过宿主机缓存进行物理读写。

virtio-win.iso (来源)：Windows 原生不带 VirtIO 驱动。你需要挂载这个 ISO，在安装 Windows 过程中手动加载驱动程序，否则安装程序找不到硬盘。

3.5 显卡与 VBIOS 注入 (`<hostdev>`)
这是直通的最核心步骤。

address domain="0x0000" bus="0x01" slot="0x00"... (来源)：

在 Linux 运行 lspci。你会看到类似 01:00.0 VGA compatible controller。

这里的 bus="01" slot="00" function="0" 必须与物理显卡的地址完全一致。

`<rom file='...'/>` (关键)：

为什么要 Patch？ 笔记本显卡的 VBIOS 通常被包含在系统 BIOS 中。直通时需要提供一份干净的、以 55 AA 字节开头的 ROM 文件。

操作：从 TechPowerUp 下载后，使用十六进制编辑器删除文件开头多余的数据，直到第一行显示 55 AA。

3.6 命名空间声明 (`<domain xmlns:qemu=...`)
作用：为了在 XML 底部使用 `<qemu:commandline>` 注入自定义参数（如“假电池”补丁），必须在根标签声明这个命名空间。

配置生效
将上述 XML 保存为 win10.xml，然后执行以下命令定义虚拟机：
```bash
virsh define win10.xml
```

---

## 4. 核心难点：解决 Code 43 与“假电池”注入

笔记本显卡直通最容易遇到的就是 **代码 43 (Code 43)** 错误。
对于移动版 NVIDIA 显卡，驱动程序会检测“电池”是否存在。虚拟机默认被视为台式机（无电池），导致驱动拒绝加载。

**这是本文的精华部分：我们需要注入一个虚假的 ACPI 电池表。**

### 4.1 编译 SSDT 电池表

安装编译器 `acpica` 或 `iasl`。创建文件 `battery.dsl`：

```c title="battery.dsl"
DefinitionBlock ("ssdt1.aml", "SSDT", 2, "RedHat", "Bat", 0x00000001)
{
    Scope (\_SB)
    {
        // 1. 添加 AC 适配器，否则电池状态可能无法正确触发更新
        Device (ADP1)
        {
            Name (_HID, "ACPI0003")
            Name (_PCL, Package() { \_SB })
            Method (_PSR, 0, NotSerialized)
            {
                Return (0x01) // 0x01 表示接通电源，0x00 表示断开
            }
            Method (_STA, 0, NotSerialized)
            {
                Return (0x0F)
            }
        }

        Device (BAT0)
        {
            Name (_HID, EisaId ("PNP0C0A"))
            Name (_UID, 0x00)
            Name (_PCL, Package() { \_SB } )

            // 0x1F = 11111b (存在、启用、UI可见、正常、有电池)
            Method (_STA, 0, NotSerialized)
            {
                Return (0x1F)
            }

            // 电池信息 (Battery Information)
            Method (_BIF, 0, NotSerialized)
            {
                Return (Package (0x0D)
                {
                    0x00,      // Power Unit: 0=mWh (毫瓦时)
                    0x2710,    // Design Capacity: 10000 mWh
                    0x2710,    // Last Full Charge Capacity: 10000 mWh
                    0x01,      // Battery Technology: 1=Rechargeable
                    0x2710,    // Design Voltage: 10000 mV (10V)
                    0x01F4,    // Design Capacity of Warning: 500 mWh
                    0x0064,    // Design Capacity of Low: 100 mWh
                    0x00,      // Capacity Granularity 1
                    0x00,      // Capacity Granularity 2
                    "BAT0",    // Model Number
                    "12345",   // Serial Number
                    "LION",    // Battery Type
                    "Virtual"  // OEM Information
                })
            }

            // 电池状态 (Battery Status)
            Method (_BST, 0, NotSerialized)
            {
                Return (Package (0x04)
                {
                    0x00,      // State: 0=正常(Idle), 1=放电, 2=充电
                    0x00,      // Present Rate: 0 (因为是满电且Idle)
                    0x2710,    // Remaining Capacity: 10000 mWh (满电)
                    0x2710     // Present Voltage: 10000 mV
                })
            }
        }
    }
}
```

编译并移动到 QEMU 可读目录（**注意权限！**）：

```bash
iasl battery.dsl
sudo mv ssdt1.aml /var/lib/libvirt/images/
sudo chmod 777 /var/lib/libvirt/images/ssdt1.aml

```

### 4.2 修改 XML 注入参数

使用 `virsh edit win10-gpu` 编辑配置

**添加底部参数**：在 `</domain>` 上方添加：
```xml
  <qemu:commandline>
    <qemu:arg value='-acpitable'/>
    <qemu:arg value='file=/var/lib/libvirt/images/ssdt1.aml'/>
  </qemu:commandline>
```



**重启虚拟机后，Windows 右下角出现电池图标，且 RTX 4070 设备状态恢复正常！**
```bash
sudo virsh destroy win10-gpu
sudo virsh start win10-gpu
```
---

## 5. 配置 Looking Glass (实现画面回传)

笔记本独显通常没有直连内屏，如果不接外接显示器，显卡处于“无头”模式。我们需要 Looking Glass 来将画面从显存拷贝到宿主机窗口。

### 5.1 宿主机设置

创建共享内存文件：

```bash title="/etc/tmpfiles.d/10-looking-glass.conf" del="your_username"
f /dev/shm/looking-glass 0660 your_username kvm -
```

在 XML 中添加 IVSHMEM 设备：

```xml
<shmem name='looking-glass'>
  <model type='ivshmem-plain'/>
  <size unit='M'>64</size>
</shmem>

```

### 5.2 虚拟机内部设置 (Windows)

1. **安装驱动**：下载并安装 Looking Glass Host (B6/B7)，确保系统设备里安装了 `IVSHMEM Device` 驱动。
2. **欺骗显示器**：安装 **Virtual Display Driver (VDD)** 或使用 HDMI 诱骗器，创建一个虚拟的 1080P/2K/4K 屏幕。
3. **接管渲染**：在显示设置里，将虚拟屏幕设为**主显示器**，并**禁用**掉默认的 QXL/virt-manager 显卡。

### 5.3 启动客户端

在 Arch Linux 终端运行：

```bash
looking-glass-client

```

此时，你应该能看到丝般顺滑的 Windows 桌面了。

---

## 6. 优化体验：解决鼠标回弹

在使用 Looking Glass 玩游戏时，如果鼠标疯狂回弹或无法控制视角，是因为 QEMU 默认的 `Tablet` 模式与 Looking Glass 的捕获模式冲突。

**解决方法**：
在 XML 配置中，**删除**所有的 `<input type='tablet'... />` 设备，只保留 `<input type='mouse'... />`。
进入游戏前，按 `Scroll Lock` 键锁定鼠标，即可获得完美的游戏体验。

---

## 总结

经过上述配置，我成功在 Arch Linux 上运行了一个全性能的 Windows 10 虚拟机：

* **显卡**: RTX 4070 满血运行，驱动无报错。
* **显示**: 通过 Looking Glass 达到 2K @ 165Hz，几乎零延迟。
* **输入**: 键鼠无缝切换。

这套方案完美解决了 Linux 用户“既要...又要...”的需求，虽然配置过程充满坎坷（特别是电池注入的细节），但最终的体验绝对物超所值。