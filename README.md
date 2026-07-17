# NVIDIA Open Kernel Module — eBPF Thermal Monitor

用 eBPF kprobe/kretprobe 钩取 NVIDIA Open GPU Kernel Module 的 **VRAM 温度**和 **Hotspot 温度**。

---

## ⚠️ 高度实验性 / HIGHLY EXPERIMENTAL

> **本项目处于早期探索阶段，不保证稳定性、兼容性或正确性。**
>
> - 依赖 NVIDIA 内部未公开的结构体和符号，驱动版本升级可能导致完全失效
> - eBPF probe 挂载在 `nv-kernel.o_binary` 模块上，行为未经官方验证
> - kretprobe 读内参指针存在竞态风险，高负载下可能丢失数据或读到脏值
> - **生产环境慎用**，建议仅在开发/调试场景使用

## 原理

nvidia-smi 读取温度的链路：

```
ioctl(/dev/nvidiactl) → nv_linux_ioctl() [kbuild] → rm_ioctl() [o_binary]
→ ctrl dispatch (0x2080xxxx) → subdeviceCtrlCmdThermalSystemExecuteV2_IMPL()
```

在 `subdeviceCtrlCmdThermalSystemExecuteV2_IMPL` 出口挂 kretprobe，遍历返回的 `instructionList[]`，找到 `opcode=0x1500 (GET_STATUS_SENSOR_READING)` 的条目，直接读其 `value`（°C）。一个 probe 同时覆盖 VRAM + Hotspot。

## 前置条件

### 1. 编译 open-gpu-kernel-modules 时启用 debug info

应用 `patches/add-debug-info.patch` 到源码树：

```bash
cd /path/to/open-gpu-kernel-modules
git apply ../nvidia-open-kmod-ebpf-temp/patches/add-debug-info.patch
make -j$(nproc)
sudo make install
```

> `-g` 零运行时开销，仅增加模块体积约 20\~40MB。生成的 DWARF debug info 让 eBPF 工具能通过字段名访问参数结构体，无需手动维护 offset。

### 2. 安装 bpftrace

```bash
# Fedora
sudo dnf install bpftrace

# Ubuntu/Debian
sudo apt install bpftrace
```

## 使用

```bash
# 运行（需要 CAP_BPF 或 root）
sudo bpftrace trace-thermal.bpftrace

# 同时跑 nvidia-smi 触发温度查询：
nvidia-smi -q -d TEMPERATURE
```

### 输出示例

```
[GPU 0] VRAM:     45 °C   (sensorIndex=1)
[GPU 0] Hotspot:  58 °C   (sensorIndex=?)
[GPU 0] GPU Core: 42 °C   (sensorIndex=0)
```

> **注意**：`sensorIndex` → 温度类型的映射由物理 RM（GSP 固件）黑盒决定。首次运行后，对比 `nvidia-smi -q -d TEMPERATURE` 的输出确认 Hotspot 对应的 sensorIndex。

## 技术细节

### kprobe + kretprobe 组合拳

kretprobe 触发时 CPU 寄存器已被返回值覆盖，无法直接读入参指针。方案：
- **kprobe（入口）**：把 `pParams` 指针存入 BPF map（key = tid）
- **kretprobe（出口）**：从 map 捞回指针 → 遍历 instructionList → 清理

### 参数结构体布局

```c
// ctrl2080thermal.h (NVIDIA internal)
struct NV2080_CTRL_THERMAL_SYSTEM_EXECUTE_V2_PARAMS {
    NvU32 reserved[5];                     // offset 0x00
    NvU32 instructionListSize;             // offset 0x14
    struct INSTRUCTION instructionList[0x20]; // offset 0x18, each 36 bytes
};

struct INSTRUCTION {
    NvS32 result;           // offset 0x00
    NvU32 executed;         // offset 0x04
    NvU32 opcode;           // offset 0x08 (0x1500 = GET_STATUS_SENSOR_READING)
    union {
        struct getStatusSensorReading {
            NvU32 sensorIndex;  // offset 0x0C
            NvS32 value;        // offset 0x10 (°C, signed!)
            // ...
        };
    } operands;             // offset 0x0C within INSTRUCTION
};
```

## 文件说明

| 文件 | 用途 |
|---|---|
| `patches/add-debug-info.patch` | Makefile patch，添加 `-g` debug info 编译标志 |
| `trace-thermal.bpftrace` | bpftrace 脚本，kprobe+kretprobe 采集温度 |
