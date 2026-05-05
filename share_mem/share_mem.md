# SVM V3 内存共享功能设计与实现原理

## 整体介绍

SVM V3 的内存共享功能分为两大体系：**设备间内存共享（IPC）** 和 **Host-Device 间内存共享（Register）**，分别通过不同组件协同完成。

业务模块在 Host 侧启动两个应用进程：host_app 进程 0 和 host_app 进程 1，分别对应设备 dev0 和 dev1。在算子执行过程中，dev0 将已完成的计算结果写入其本地设备内存。若后续计算需由 dev1 执行，则 dev1 可通过共享内存机制直接访问 dev0 的运算结果，无需经过 Host 中转或显式的数据拷贝，从而显著提升算子流水线的执行效率。

---

## 一、设备间内存共享（IPC）端到端实现原理

设备间内存共享通过 `halShmemCreateHandle` / `halShmemOpenHandleByDevId` / `halShmemCloseHandle` / `halShmemDestroyHandle` 四个接口完成。

### 1.1 核心组件架构

```
┌──────────────────────────────────────────────────────────────────┐
│  用户态 Host 进程                                                  │
│                                                                    │
│  halShmemCreateHandle(name, va, size)                              │
│    └→ svm_ipc_create_handle(va, size, &key)                       │
│         ├→ svm_get_prop() → 获取内存属性                           │
│         ├→ svm_casm_create_key(&dst_va, &key)                      │
│         │    └→ ioctl → [Kernel] casm_key_id_alloc()               │
│         │         └→ IDR分配+Key生成 → 存储源信息(casm_src_node)     │
│         └→ svm_ipc_format_name(name, key)                          │
│              └→ 将key+源信息编码为64字节IPC名称                      │
│                                                                    │
│  halShmemSetPidHandle(name, pid[], num)                            │
│    └→ svm_casm_add_local_task(key, pid, num)                       │
│         └→ [Kernel] MWL白名单设置                                   │
│                                                                    │
│  halShmemOpenHandleByDevId(dev_id, name, &vptr)                    │
│    └→ svm_ipc_parse_name(name, &key, &src_va)                      │
│    └→ svm_ipc_open_handle(devid, key, &opened_va)                  │
│         ├→ svm_casm_get_src_va_ex(devid, key, &src_va, &access_va) │
│         │    └→ ioctl → [Kernel] casm_mem_pin()                    │
│         │         ├→ MWL权限检查 (svm_mwl_task_is_trusted)          │
│         │         ├→ smp_pin_mem() → PIN住源内存物理页              │
│         │         └→ casm_add_dst_node() → DST节点注册到红黑树      │
│         ├→ svm_ipc_malloc_opened_va(devid, size)                   │
│         │    └→ svm_malloc(VA_ONLY) → 仅分配虚拟地址                │
│         └→ svm_casm_mem_map(devid, opened_va, size, key)           │
│              └→ ioctl → [Kernel] casm_mem_map_handler()             │
│                   └→ 根据连接方式建立设备间页表映射                    │
│                                                                    │
│  halShmemCloseHandle(vptr)                                         │
│    └→ svm_ipc_close_handle(opened_va)                              │
│         ├→ svm_casm_mem_unmap(devid, opened_va, size)              │
│         └→ svm_ipc_free_opened_va(opened_va)                       │
│                                                                    │
│  halShmemDestroyHandle(name)                                       │
│    └→ svm_ipc_destroy_handle(key)                                  │
│         └→ ioctl → [Kernel] casm_key_id_free() → 释放Key+源信息     │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 IPC 名称编码机制

IPC 名称是一个 64 字节的字符串，内部以二进制格式存储 `svm_ipc_info` 结构体：

```
struct svm_ipc_info {        // 共40字节，编码进64字节名称空间
    u8  ver;                 // 版本号
    u8  cs_valid;            // 跨服务器有效标记
    u16 rsv;                 // 保留
    u16 server_id;           // 服务器ID（跨服务器共享）
    u16 udevid;              // 统一设备ID
    int owner_pid;           // 创建者PID
    int tgid;                // 设备侧进程TGID
    u64 va;                  // 源虚拟地址
    u64 size;                // 源内存大小
    u64 key;                 // CASM分配的唯一Key
};
```

为了避免字节 `0x00` 作为字符串终止符导致信息截断，采用 **替换位图编码**：将值为 0 的字节替换为 1，同时用额外 6 字节的 bitmap 记录哪些位置被替换。bitmap 的每个字节最高位固定为 1，用剩余 7 位标识 7 个数据字节（每 7 字节对应 1 字节 bitmap），确保不会出现连续的 0 再次被误判为字符串结尾。

> **设计考量**：IPC 名称之所以采用字符串形式而非二进制句柄，是为了兼容进程间通信机制（如 Unix Domain Socket、共享文件等），名称可以直接作为字符串传递而无需担心二进制数据在传输中被破坏。替换位图编码保证了二进制信息在字符串通道中的完整性。

### 1.3 CASM Key 管理（内核态）

CASM（Core Accelerator Share Memory）是内核态共享内存管理的核心组件，负责 Key 的分配、源信息管理和白名单控制。

#### Key 分配（`casm_key.c`）

- 使用内核 IDR（ID Radix tree）为每个设备分配唯一的循环递增 ID
- Key 由 `(udevid << 32) | id` 组成，支持跨服务器扩展（通过 `casm_key_ops` 可注册自定义 key 编码）
- 分配时创建 `casm_src_node`，记录源 VA、大小、创建者 PID/TGID、对齐信息等

**数据结构**：

```
casm_key_ctx (每个设备一个):
  ├── udevid          // 统一设备ID
  ├── dev_ctx         // 设备上下文指针
  ├── mutex           // Key操作互斥锁
  └── idr             // ID Radix Tree (ID → casm_src_node)

casm_src_node:
  ├── src_va          // 源虚拟地址信息 (svm_global_va)
  ├── owner_pid       // 创建者PID
  ├── owner_tgid      // 创建者设备侧TGID
  └── aligned_va      // 按共享粒度对齐后的VA
```

#### 源信息管理（`casm_src.c`）

- 维护 `casm_src_ctx` 结构，包含创建者 PID、对齐 VA、引用计数
- 支持跨服务器源信息同步（`casm_cs_set_src_info` / `casm_cs_clr_src_info`）

#### DST 节点管理（`casm_dst.c`）

`casm_dst_node` 记录打开的共享内存映射：

```
casm_dst_node:
  ├── range_node      // 红黑树节点 (以 DST VA 为键)
  ├── key             // 关联的 CASM Key
  ├── src_va          // 源内存全局信息 (udevid, tgid, va, size)
  └── ex_info         // 扩展信息
       ├── is_local      // 是否本地共享
       ├── owner_tgid    // 源Owner TGID
       └── updated_va    // UB更新后的VA (UB直连场景)
```

**内存 PIN 流程**（`casm_mem_pin`）：
1. `casm_src_info_get` → 获取源内存信息并进行 MWL 权限检查
2. `smp_pin_mem` → 将源内存物理页 PIN 住，防止换出或释放
3. `casm_add_dst_node` → 将 DST 节点插入 `casm_dst_ctx` 的红黑树

**内存 UNPIN 流程**（`casm_mem_unpin`）：
1. 从红黑树查找并删除 DST 节点
2. `smp_unpin_mem` → 解除物理页 PIN
3. `casm_src_info_put` → 释放源信息引用

#### 跨服务器共享（`casm_cs` / `casm_adapt/cross_server`）

- 当 `cs_valid == 1` 时，表示共享内存来自不同服务器
- Open 时先调用 `casm_cs_set_src_info` 将远程源信息注册到本地 CASM
- Close 时调用 `casm_cs_clr_src_info` 清除远程源信息
- 跨服务器场景通过 `casm_key_ops` 扩展 Key 编码，将远程服务器 ID 编码进 Key 的高位

### 1.4 MWL 内存白名单机制（内核态）

MWL（Memory Whitelist）负责共享内存的进程权限控制，防止未授权进程访问共享内存：

- **白名单设置**：`halShmemSetPidHandle` → `casm_add_local_task(key, pid[])` → 内核态 MWL 将指定 PID 添加到 key 对应的可访问白名单
- **权限检查**：每次 Open 时调用 `svm_mwl_task_is_trusted()` 验证当前进程是否有权限访问该 key
- **Pod PID 支持**：`halShmemSetPodPid` 支持按 SDID（Super Device ID）分组设置白名单，适用于多服务器集群场景

> **安全设计**：MWL 在内核态执行权限检查，用户态无法绕过。即使恶意进程获取到了 IPC name 字符串，如果其 PID 不在白名单中，Open 操作也会被拒绝。

### 1.5 共享内存映射建立（SMM）

SMM（Share Memory Mapping）负责在 DST 设备和 SRC 设备之间建立实际的页表映射：

- **Host 侧**（SRC 在 Host）：`svm_smm_mmap()` 直接 ioctl 调用内核建立映射
- **Device 侧**（SRC 在 Device）：`svm_smm_remote_map()` 通过 UMC H2D 消息通知设备侧 CP1 进程，在设备内核完成映射
- **映射建立后**：DST VA 可以通过设备 SMMU 直接访问 SRC 物理内存，CPU/设备均可透明读写

```
svm_smm_client_map(dst_info, src_info, flag)
  │
  ├─ dst在Host → svm_smm_mmap(host_devid, &va, size, flag, src_info)
  │               └→ ioctl(SMM_MMAP_CMD)
  │                    └→ [Kernel] 建立Host侧页表映射
  │
  └─ dst在Device → svm_smm_remote_map(dst_info, src_info, flag)
                    └→ svm_apbi_query(devid, CP1)
                    └→ svm_umc_h2d_send(SMM_MMAP_EVENT, &msg)
                         └→ [Device Kernel] 建立设备侧SMMU映射
```

### 1.6 共享内存拷贝时的路径解析

当对共享内存执行 `halMemcpy` 时，需要解析真实的源信息：

```
svm_copy_try_resolve_shared_info()
  ├→ svm_ipc_query_src_info(opened_va)
  │    └→ svm_get_prop() → 获取handle
  │    └→ svm_get_priv(handle) → 读取svm_ipc_opened_va_priv
  │         └→ 返回svm_global_va{server_id, udevid, tgid, va, size}
  └→ svm_copy_real_info_pack()
       ├→ uda_get_devid_by_udevid_ex() → 统一设备ID转真实设备ID
       └→ halQueryMasterPidByDeviceSlave() → 查询Master进程tgid
```

对于 PCIe 连接的设备，共享内存解析后若 src 和 dst 在同一设备，直接走本地拷贝 `svm_memcpy_local_client()`，避免不必要的 H2D 传输。这是共享内存场景下性能优化的关键路径。

---

## 二、Host-Device 间内存共享（Register）端到端实现原理

Host-Device 内存共享通过 `halHostRegister` / `halHostUnregister` 实现对端内存的透明访问。

### 2.1 核心流程

```
halHostRegister(src_ptr, size, flag, devid, &dst_ptr)
  └→ [flag = DEV_SVM_MAP_HOST]  设备内存映射到Host
       └→ svm_register_inner(src_ptr, size, devid, flag)
            ├→ svm_alloc_va()                    // 分配Host侧VA
            ├→ svmm_inst_create()               // 创建SVMM实例(设备无关)
            ├→ svmm_add_seg()                    // 添加段记录
            ├→ svm_smm_client_map(dst, src, flag) // 建立页表映射
            │    ├→ [src在Host] ioctl(SMM_MMAP_CMD)
            │    └→ [src在Device] umc_h2d_send(SMM_MMAP_EVENT)
            └→ dst_ptr = 映射后的VA

  └→ [flag = HOST_SVM_MAP_DEV]  Host内存映射到设备
       └→ 同上，但方向相反，在设备侧分配VA并建立映射

halHostUnregister(src_ptr, devid)
  └→ svm_register_inner_unregister()
       └→ svmm_del_seg() + svm_smm_client_unmap() + svm_free_va()
```

### 2.2 SVMM（Shared Virtual Memory Manager）

SVMM 是用户态共享内存段管理器，每个 register 操作会创建一个 SVMM 实例：

```
struct svmm_inst {
    enum svmm_overlap_type overlap_type;    // 重叠模式
    u64 svmma_start;                        // 共享虚拟地址起始
    u64 svmma_size;                         // 共享虚拟地址大小
    u64 svm_flag;                           // SVM标记
    u64 seg_num;                            // 段计数
    pthread_rwlock_t pipeline_rwlock;        // Pipeline读写锁
    pthread_rwlock_t rwlock;                 // 段操作读写锁
    union {
        struct rbtree_root root;            // 单设备红黑树
        struct rbtree_root dev_root[SVM_MAX_DEV_NUM]; // 多设备红黑树
    };
};
```

支持两种重叠模式：
- **SVMM_OVERLAP**：允许多个段交叠（如 register 不同大小的同一区域到不同设备）
- **SVMM_NON_OVERLAP**：段区间不重叠，适用于精确的一对一映射场景

### 2.3 自动 Host Register 机制

SVM V3 实现了 **自动 Host Register** 优化（`svm_host_register.c`），这是实现高性能 H2D/D2H 拷贝的关键：

**触发时机**：当设备 Agent 打开时（`halMemAgentOpen`），调用 `svm_host_register_enable(devid)`。

**自动注册流程**：
```
svm_host_register_enable(devid)
  ├→ g_support_register[devid] = true
  ├→ svm_for_each_valid_handle(svm_try_host_register_existed_mem)
  │    └→ 遍历所有已分配的SVM内存handle
  │         ├→ [Normal内存] 通过svm_normal_ops.post_malloc自动注册
  │         ├→ [Cache内存]  通过svm_cache_for_each_range遍历注册
  │         └→ [VMM内存]    通过vmm_svmm_for_each_seg遍历注册
  └→ svm_register_to_master(user_devid, &register_va, REGISTER_FLAG_ACCESS_WRITE)
       └→ [Kernel] PIN住物理页 + 建立设备SMMU映射
```

**生命周期管理**：
- **新分配自动注册**：通过 `svm_normal_ops` 回调链，`post_malloc` 钩子在新内存分配后自动注册到所有已 enable 的设备
- **内存释放自动取消**：通过 `pre_free` 钩子自动取消注册
- **设备关闭自动清理**：`svm_host_register_disable(devid)` 遍历取消所有注册

**优点**：避免了每次 `halMemcpy` 时临时 PIN/UNPIN Host 物理页的开销（典型的 H2D 拷贝路径中，临时 PIN/UNPIN 可能占据 30%~50% 的延迟），大幅提升 H2D/D2H 拷贝性能。

**PCIe 场景特殊处理**：PCIe 连接的设备在 disable 时不主动 unregister，因为地址可能仍在使用中，依赖内核回收机制清理。

---

## 三、并发控制

共享内存操作涉及多进程、多线程并发，SVM V3 通过多层锁机制保障正确性：

| 层级 | 锁类型 | 保护对象 | 使用接口 |
|------|--------|----------|----------|
| **用户态 Pipeline** | `pthread_rwlock_t` | 全局 SVM 操作串行化 | `svm_use_pipeline()` / `svm_unuse_pipeline()` |
| **CASM Key 锁** | `ka_mutex_t` | IDR 分配/释放 | `ka_base_idr_lock()` / `ka_base_idr_unlock()` |
| **CASM DST 锁** | `ka_task_read/write_lock_bh` | DST 红黑树 | `casm_dst_ctx.lock` |
| **Register 链锁** | `pthread_mutex_t` | register_node 链表 | `register_mutex` |
| **SVMM 段锁** | `pthread_rwlock_t` | 段红黑树 | `svmm_inst.rwlock` |
| **SMP PIN/UNPIN** | SMP 协议锁 | 源内存物理页状态 | `smp_pin_mem()` / `smp_unpin_mem()` |

**SMP PIN/UNPIN 机制**：通过 SMP（Shared Memory Protocol）在源设备侧执行内存 PIN，确保共享内存在使用期间不被释放或换出。PIN 操作在 Open 时执行，UNPIN 在 Close 时执行，保证了 DST 侧访问的稳定性和一致性。

---

## 四、完整调用时序

以下时序图展示了两进程设备间内存共享的完整交互过程：

```
进程0 (dev0 Owner)                    进程1 (dev1 User)                      Kernel (dev0侧)
─────────────────                     ────────────────                      ─────────────
halMemAlloc → DeviceVA_0
halShmemCreateHandle(va0, name)
  ├→ casm_create_key(va0, &key)
  │    └── ioctl ─────────────────────────────────────────────────→ casm_key_id_alloc()
  │                                                                     ├→ IDR分配ID
  │                                                                     ├→ 创建casm_src_node
  │                                                                     └→ 关联key→源信息
  └→ ipc_format_name(name, key)
       └→ 编码key+源信息到name字符串

halShmemSetPidHandle(name, [pid1])
  └── ioctl ─────────────────────────────────────────────────────→ MWL: key → [pid1]

(name传递给进程1)
                                       halShmemOpenHandleByDevId(dev1, name, &va1)
                                         ├→ ipc_parse_name(name, &key, &src_va)
                                         ├→ casm_get_src_va_ex(dev1, key)
                                         │    └── ioctl ──────────→ casm_mem_pin()
                                         │                              ├→ MWL检查(pid1∈白名单?)
                                         │                              ├→ smp_pin_mem(va0) //PIN住源页
                                         │                              └→ casm_add_dst_node()
                                         ├→ ipc_malloc_opened_va(dev1, size)
                                         │    └→ svm_malloc(VA_ONLY) → DeviceVA_1
                                         └→ casm_mem_map(dev1, va1, size, key)
                                              └── ioctl ──────────→ casm_mem_map_handler()
                                                                       └→ 建立dev1→dev0 SMMU页表映射

// dev1 现在可以通过 va1 直接读写 dev0 的内存
// halMemcpy 时自动解析共享关系，走最优路径

halShmemCloseHandle(va1)
  └→ casm_mem_unmap(dev1, va1, size)
       └── ioctl ────────────────────────────────────────────────→ casm_mem_unpin()
                                                                       ├→ casm_del_dst_node()
                                                                       └→ smp_unpin_mem(va0)

halShmemDestroyHandle(name)
  └→ casm_destroy_key(key)
       └── ioctl ─────────────────────────────────────────────────→ casm_key_id_free()
                                                                       └→ 释放src_node
```

---

## 五、关键设计要点

### 5.1 Name 编码机制
64 字节的名称空间以二进制格式存储源内存元信息（version, server_id, udevid, tgid, va, size, key），使用替换 bitmap 编码避免 `0x00` 截断。IPC 名称成为"自描述的共享内存句柄"，可直接通过字符串通道传递。

### 5.2 CASM Key 透明性
Key 由 `(udevid << 32) | id` 组成，对上层业务透明。跨服务器场景通过 `casm_key_ops` 扩展，支持将远程服务器 ID 编码进 Key 的高位，实现跨服务器共享内存的无缝访问。

### 5.3 MWL 白名单安全
内核态 MWL 强制检查 Open 进程的 PID 是否在创建者设置的白名单中。白名单支持本地 PID 和跨服务器 Pod PID，提供灵活的多级访问控制。安全检查在内核态执行，用户态无法绕过。

### 5.4 SMP PIN 机制
打开共享内存时通过 SMP 协议将源内存物理页 PIN 住，防止原进程释放内存或操作系统换出，保证 DST 侧访问的稳定性和一致性。PIN 状态在 Open/Close 之间持续有效。

### 5.5 自动 Host Register
设备 Agent 打开时自动将所有已有和未来的 Host 内存注册到设备 SMMU，避免每次拷贝的临时 PIN 开销。这是实现高性能 H2D/D2H 拷贝的关键优化，通过 `svm_normal_ops` 回调链实现全生命周期自动管理。

### 5.6 SVMM 多设备管理
每个 register 操作创建独立的 SVMM 实例，每个设备维护独立的映射段列表。支持同一个源内存向多个设备共享，以及同一设备上的多段映射管理（重叠/非重叠模式）。

### 5.7 连接方式适配
根据设备连接方式（PCIe/UB/HCCS）选择不同的 mem_map 实现路径：
- **PCIe**：通过标准 SMMU 页表映射
- **UB（Ultra Bus）**：支持 UB-updated VA 直接映射，减少页表转换开销
- **HCCS**：通过高速缓存一致性总线共享

---

## 六、核心数据结构速查

| 组件 | 源文件 | 核心结构 | 功能 |
|------|--------|----------|------|
| **IPC Name** | `svm_ipc.c` | `svm_ipc_info` (40B) | 编码为64B名称字符串 |
| **CASM Key** | `casm_key.c` | `casm_key_ctx` / `casm_src_node` | IDR分配Key + 源信息存储 |
| **CASM DST** | `casm_dst.c` | `casm_dst_node` + range_rbtree | DST映射节点管理 + PIN/UNPIN |
| **CASM CS** | `casm_adapt/cross_server/` | 跨服务器扩展 | 远程源信息同步 |
| **MWL** | `mwl_core.c` | `mwl_ctx` | 内存白名单权限控制 |
| **SMM** | `smm_client.c` | `svm_dst_va` / `svm_global_va` | 共享内存页表映射 |
| **SVMM** | `svmm_inst.c` | `svmm_inst` + rbtree | 用户态共享内存段管理 |
| **Host Register** | `svm_host_register.c` | `svm_register_node` | 自动Host→设备内存注册 |
| **SMP** | `smp/` | SMP协议 | 跨进程内存PIN/UNPIN |
