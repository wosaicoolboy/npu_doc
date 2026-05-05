## halMemAlloc 与 halMemcpy 端到端实现原理（V3 架构）

本节深入解析 SVM V3 架构下 `halMemAlloc` 申请 Host/Device 内存，以及 `halMemcpy` 完成 H2D 拷贝的完整内部实现链路与核心原理。

### 1. 整体架构分层

SVM V3 的实现分为**用户态层**和**内核态层**两大部分，通过 ioctl 系统调用和 UMC（User-Mode Communication）消息传递机制协作：

```
┌─────────────────────────────────────────────────────────────────┐
│  业务层 (aclrtMalloc / aclrtMemcpy)                              │
├─────────────────────────────────────────────────────────────────┤
│  HAL API 层                                                      │
│  halMemAlloc / halMemFree / halMemcpy                           │
│  (src/ascend_hal/svm/v3/api/master/)                             │
├─────────────────────────────────────────────────────────────────┤
│  用户态 SVM 核心层                                                │
│  ├── malloc_mng (句柄/红黑树管理)                                  │
│  ├── normal_malloc / cache_malloc (分配策略)                      │
│  ├── va_allocator (虚拟地址空间分配器)                              │
│  └── mpl_client (物理页填充命令发送)                                │
├─────────────────────────────────────────────────────────────────┤
│  用户态 Copy 适配层                                               │
│  ├── svm_memcpy (方向分发 H2H/H2D/D2H/D2D)                        │
│  └── pci_adapt / ub_adapt (连接方式适配)                           │
├──────────────────────── ioctl / UMC ────────────────────────────┤
│  内核态 SVM 层 (src/sdk_driver/svm/v3/)                           │
│  ├── MPL (Memory Populate Layer - 物理页分配 + 页表映射)           │
│  ├── PMM (Physical Memory Manager - 物理段追踪)                    │
│  ├── KSVMM (Kernel SVM Manager - 共享内存段管理)                   │
│  ├── PMA (P2P Memory Access - P2P页表管理)                        │
│  └── DMA Copy Control (DMA 硬件拷贝引擎调度)                       │
├─────────────────────────────────────────────────────────────────┤
│  硬件层                                                           │
│  ├── SMMU/IOMMU (设备侧页表)                                       │
│  ├── DMA Engine (数据搬运引擎)                                      │
│  └── NPU Device Memory (HBM/DDR)                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. halMemAlloc 端到端实现原理

#### 2.1 API 入口层

`halMemAlloc(void **pp, unsigned long long size, unsigned long long flag)` 定义在 `api/master/svm_alloc.c`，是外部调用的直接入口：

- **参数解析**：`flag` 编码了内存类型（`MEM_HOST_VAL`/`MEM_DEV_VAL`/`MEM_DVPP_VAL`/`MEM_HOST_UVA_VAL`）、页粒度（`MEM_PAGE_HUGE`/`MEM_PAGE_GIANT`）、物理连续性（`MEM_CONTIGUOUS_PHY`）、P2P 属性（`MEM_ADVISE_P2P`/`MEM_ADVISE_BAR`）、只读属性（`MEM_HOST_RW_DEV_RO`）等。
- **内部流程**：`halMemAlloc` → `halMemAllocInner` → `svm_master_init()` → `svm_mem_malloc()`。

关键入口代码（`svm_alloc.c`）：
```
halMemAlloc(pp, size, flag)
  → halMemAllocInner
    → svm_master_init()        // 初始化 SVM 主进程上下文
    → svm_use_pipeline()       // 获取读锁（多线程读并发、写独占）
    → svm_mem_malloc(pp, size, flag)
    → svm_unuse_pipeline()     // 释放读锁
```

**Pipeline 机制**（`svm_pipeline.c`）：使用 `pthread_rwlock_t` 实现读写锁，`halMemAlloc`/`halMemFree`/`halMemcpy` 等常规操作以读锁模式进入（`svm_use_pipeline()`），允许多线程并发；设备热复位等需要独占内存管理结构的操作以写锁模式进入（`svm_occupy_pipeline()`），防止写锁饥饿问题。

#### 2.2 内存分配主流程（svm_mem_malloc）

`svm_mem_malloc` 的内部处理分为三个阶段：

**阶段一：参数解析与能力标记**

1. **parse_alloc_devid**：从 `flag` 低位提取 `devid`，区分 Host/Device 内存。
   - `MEM_DEV_VAL`/`MEM_DVPP_VAL` → 目标设备 devid，合法性检查（`< SVM_MAX_DEV_AGENT_NUM`）。
   - `MEM_HOST_VAL`/`MEM_HOST_UVA_VAL` → `devid = svm_get_host_devid()`。

2. **parse_alloc_svm_flag**：将外部 HAL flag 转换为内部 SVM flag：
   - 页粒度：`MEM_PAGE_HUGE` → `SVM_FLAG_ATTR_PA_HPAGE`，`MEM_PAGE_GIANT` → `SVM_FLAG_ATTR_PA_GPAGE`
   - 物理属性：`MEM_CONTIGUOUS_PHY` → `SVM_FLAG_ATTR_PA_CONTIGUOUS`，`MEM_ADVISE_P2P` → `SVM_FLAG_ATTR_PA_P2P`
   - 功能能力（Capability）：自动附加 `SVM_FLAG_CAP_SYNC_COPY`、`SVM_FLAG_CAP_REGISTER`、`SVM_FLAG_CAP_PREFETCH`、`SVM_FLAG_CAP_IPC_CREATE` 等能力标记。
   - 特殊处理：`MEM_DEV_CP_ONLY`（仅 CP 侧可见）→ `SVM_FLAG_DEV_CP_ONLY` + `SVM_FLAG_BY_PASS_CACHE`；`MEM_HOST_RW_DEV_RO` → `SVM_FLAG_ATTR_PG_RDONLY`。

**阶段二：分配策略路由（svm_malloc）**

`svm_malloc`（`assign/malloc_mng/malloc_mng.c`）是核心分配管理器，采用两路分配策略：

```
                                svm_malloc()
                                     │
                          malloc_para_check()
                                     │
                          _svm_malloc(start, &aligned_size, ...)
                                     │
               ┌─────────────────────┴─────────────────────┐
               │ go_malloc_cache()?                         │
               │ (OOM 时 shrink cache)                      │
          ┌────▼────┐                              ┌───────▼───────┐
          │ Cache    │                              │ Normal        │
          │ Malloc   │                              │ Malloc        │
          └────┬────┘                              └───────┬───────┘
               │                                           │
               │                  ┌────────────────────────┼────────────────────────┐
               │                  │              normal_va_alloc()                   │
               │                  │        (从预留虚拟地址范围分配VA)                  │
               │                  │                                                  │
               │                  │    VA_ONLY? ─── YES ──→ 跳过物理页分配             │
               │                  │         │                                        │
               │                  │         NO                                       │
               │                  │         │                                        │
               │                  │    normal_mem_populate()                         │
               │                  │    (分配物理页 + 建立页表映射)                      │
               │                  └────────────────────────┼────────────────────────┘
               │                                           │
               └───────────────────────┬───────────────────┘
                                       │
                               svm_prop_pack()
                               handle_init()
                               svm_mng_ops_post_malloc()
                               handle_insert()  // 插入红黑树
```

**Cache Malloc** vs **Normal Malloc** 选择策略：
- **Cache Malloc**：当满足以下全部条件时优先使用：
  - `numa_id == SVM_MALLOC_NUMA_NO_NODE`（未指定 NUMA 节点）
  - 非旁路缓存模式（非 `SVM_FLAG_BY_PASS_CACHE`）
  - 非连续物理页（非 `SVM_FLAG_ATTR_PA_CONTIGUOUS`）
  - 非巨页（非 `SVM_FLAG_ATTR_PA_GPAGE`）
  - 非只读页（非 `SVM_FLAG_ATTR_PG_RDONLY`）
  - 底层缓存层支持该规格（`svm_cache_is_support`）
- **正常分配 Normal Malloc**：不满足 Cache 条件时走标准路径。

**OOM 时的 Cache 收缩机制**：当 Normal Malloc 返回 `DRV_ERROR_OUT_OF_MEMORY` 时，系统会调用 `shrink_cache()` 释放缓存中的空闲内存，然后重试一次 Normal Malloc。

**阶段三：句柄管理（红黑树 + 引用计数）**

`malloc_mng.c` 使用红黑树（rbtree）管理所有已分配的内存块句柄：

- **handle_t** 结构体包含：
  - `prop`：内存属性（devid, flag, start, size, aligned_size）
  - `ref`：原子引用计数
  - `task_bitmap`：任务位图（标记 CP 侧使用）
  - `priv`/`priv_ops`：私有数据与操作（用于协议特定扩展，如共享内存/IPC）
- **句柄生命周期**：
  - `handle_alloc()` → `handle_init()` → `handle_insert()`（分配并插入红黑树）
  - `svm_get_prop()` 通过 `handle_get()` 查询红黑树获取属性
  - `svm_free()` → `handle_erase()` → `handle_uninit()` → `handle_free()`（释放）

#### 2.3 Virtual Address 分配（va_allocator）

`normal_va_alloc()` 调用 `svm_alloc_va()` 从预分配的虚拟地址空间中分配一段连续 VA：

- **Host 内存**：从 Host VA 预留范围分配，通过 `mmap` 创建匿名映射区域（`MAP_ANONYMOUS | MAP_SHARED`），确保有 VMA 结构供后续页表操作。
- **Device 内存**：同样从预分配 VA 范围中分配，并创建 VMA。对于 `DEV_CP_ONLY` 内存，标记为 `SVM_VA_ALLOCATOR_FLAG_DEV_CP_ONLY`，不关联 Host 侧的 Master 进程。
- **对齐**：按页大小对齐。对于连续物理页（Contiguous），需要按 order 对齐（2 的幂次倍页大小）。
- **Master UVA**：`MEM_HOST_UVA_VAL` 类型内存使用 `SVM_VA_ALLOCATOR_FLAG_MASTER_UVA` 标记，VA 由上层 Runtime 指定。

#### 2.4 物理页分配与页表建立（内核态 MPL）

这是 `halMemAlloc` 最关键的内核态操作，由 `normal_mem_populate()` 触发。

##### 2.4.1 用户态 → 内核态通信路径

根据目标 devid 不同，走两条路径：

**路径 A：Host 侧内存（devid == svm_get_host_devid()）**
```
normal_mem_populate()
  → svm_mpl_client_populate()
    → svm_mpl_populate(devid, va, size, flag)   // 直接调用内核 ioctl
      → ioctl(fd, SVM_MPL_POPULATE_CMD, &para)
        → 内核态 mpl_ioctl_populate()
```

**路径 B：Device 侧内存（devid 指向 NPU 设备）**
```
normal_mem_populate()
  → svm_mpl_client_populate()
    → svm_mpl_populate_remote(devid, va, size, flag)
      → svm_apbi_query(devid, CP1)  // 查询 CP1 进程信息
      → svm_umc_h2d_send(&head, &msg)  // 通过 UMC 通道发送 H2D 消息
        → 设备侧 CP1 进程收到 SVM_MPL_POPULATE_EVENT
          → 调用 svm_mpl_populate() 在设备侧内核完成物理页分配
```

设备侧内存的物理页分配发生在 NPU 设备侧的 CP1（Control Processor 1）进程中，其 `tgid` 通过 `svm_apbi_query` 从 APBI（Accelerator Process Binding Info）表中查询。

##### 2.4.2 内核态 MPL Populate 详细流程

内核函数 `mpl_populate()`（`sdk_driver/svm/v3/mpl/mpl.c`）：

1. **权限检查**（`mpl_permission_check`）：确认调用进程已通过 APM 绑定到对应设备，且进程类型为 `PROCESS_CP1`。

2. **参数校验**（`mpl_comm_para_check`）：VA 和 size 必须按页大小对齐，页大小通过 `mpl_get_page_gran_by_flag()` 从 flag 中提取（Giant/Huge/Normal）。

3. **物理页分配**（`svm_alloc_pages`）：
   - 调用 GFP（General Frame Pool）分配 `page_num = size / page_size` 个物理页
   - 支持 Flag：
     - `SVM_GFP_FLAG_CONTINUOUS`：物理连续页（使用 compound page）
     - `SVM_GFP_FLAG_P2P`：通过 P2P（Peer-to-Peer）BAR 空间分配
     - `SVM_GFP_FLAG_FIXED_NUMA`：固定 NUMA 节点分配

4. **VMA 查找与验证**（`ka_mm_find_vma`）：在内核地址空间中找到用户态 mmap 创建的 VMA 结构，验证 VA 范围。

5. **PMM 段注册**（`pmm_add_seg`）：将 VA 范围注册到 PMM（Physical Memory Manager），标记为 `PMM_SEG_WITH_PA`（已分配物理地址）。

6. **页表建立**（`svm_remap_pages`）：核心页表映射操作，将物理页映射到用户 VA：
   - 设置 VMA 状态为 `VMA_STATUS_NORMAL_OP`
   - 调用 `remap_pfn_range` 或等价的页表项填充函数
   - 设置页表属性（`svm_pgtlb_attr_packet`）：缓存属性（`PG_NC` 非缓存）、访问权限（`PG_RDONLY` 只读）、页大小
   - 恢复 VMA 状态为 `VMA_STATUS_IDLE`

7. **内存清零**（`svm_clear_mem_by_uva`）：对于非 Host 侧且非只读的内存，通过 UVA（Unified Virtual Address）将新分配的物理页内容清零。

##### 2.4.3 设备侧 MMU 页表同步

在设备侧 NPU 上，SMMU（System MMU，或称 IOCU）需要与 Host 侧 CPU 页表保持同步。SVM V3 通过以下机制保障：

- **SVM_SET 命令**：在 populate 成功后，sys_cmd/ioctl_post_handler 会向设备侧发送 SVM_SET cmd，更新设备 SMMU 的页表项。
- **Page Fault 处理**：若设备访问未映射的 VA，触发页故障中断，由 `svm_pagefault.c` 中的处理函数解析 fault address，并通过 UMC 通道通知 Host 侧按需填充（demand paging）。

#### 2.5 halMemAlloc 完整调用链总结

```
halMemAlloc(pp, size, flag)
  │
  ├─ halMemAllocInner(pp, size, flag)
  │    ├─ svm_master_init()                               // 初始化 Master 进程
  │    ├─ svm_use_pipeline()                              // 获取读锁
  │    └─ svm_mem_malloc(pp, size, flag)
  │         ├─ svm_parse_alloc_devid()                     // 解析目标设备ID
  │         ├─ svm_parse_alloc_module_id()                 // 解析调用模块ID
  │         ├─ svm_parse_alloc_svm_flag()                  // 转换为内部SVM标记
  │         └─ svm_module_mem_malloc(devid, numa_id, svm_flag, &start, size, module_id)
  │              └─ svm_malloc(start, size, align, flag, location)
  │                   ├─ malloc_para_check()               // 参数校验
  │                   ├─ _svm_malloc()                     // 两路分配
  │                   │    ├─ go_malloc_cache? ─── malloc_cache()
  │                   │    │    (Cache层快速分配复用已缓存的页)
  │                   │    └─ else: malloc_normal()
  │                   │         ├─ normal_va_alloc()       // VA分配(mmap创建VMA)
  │                   │         │    └─ svm_alloc_va()     // VA分配器
  │                   │         └─ normal_mem_populate()   // 物理页分配+页表映射
  │                   │              ├─ [Host侧] svm_mpl_populate()
  │                   │              │    └─ ioctl(SVM_MPL_POPULATE_CMD)
  │                   │              │         └─ [Kernel] mpl_populate()
  │                   │              │              ├─ svm_alloc_pages()     // 分配物理页
  │                   │              │              ├─ pmm_add_seg()          // PMM注册
  │                   │              │              ├─ svm_remap_pages()      // 页表建立
  │                   │              │              └─ svm_clear_mem_by_uva() // 内存清零
  │                   │              └─ [Device侧] svm_mpl_populate_remote()
  │                   │                   └─ svm_umc_h2d_send() → CP1
  │                   │                        └─ [Kernel] mpl_populate()(同上)
  │                   ├─ handle_alloc()
  │                   ├─ svm_prop_pack()               // 打包属性结构
  │                   ├─ handle_init()                 // 初始化句柄
  │                   ├─ svm_mng_ops_post_malloc()     // 后置回调
  │                   └─ handle_insert()               // 插入红黑树
  └─ svm_unuse_pipeline()                              // 释放读锁
```

### 3. halMemcpy 端到端实现原理

#### 3.1 API 入口与方向判断

`drvMemcpy(DVdeviceptr dst, size_t dest_max, DVdeviceptr src, size_t byte_count)` 定义在 `api/master/svm_cpy.c`：

```
drvMemcpy(dst, dest_max, src, byte_count)
  → drvMemcpyInner(dst, dest_max, src, byte_count)
    ├─ 参数校验: dst != 0, src != 0, dest_max >= byte_count
    ├─ svm_copy_check_addr_prop_cap(dst, SVM_FLAG_CAP_SYNC_COPY) // 检查dst能力
    ├─ svm_copy_check_addr_prop_cap(src, SVM_FLAG_CAP_SYNC_COPY) // 检查src能力
    ├─ svm_set_copy_va_info(src) // 通过svm_get_prop查询红黑树获取src的devid/属性
    ├─ svm_set_copy_va_info(dst) // 通过svm_get_prop查询红黑树获取dst的devid/属性
    └─ svm_memcpy_sync(&src_info, &dst_info)
```

**`svm_set_copy_va_info()`** 的核心作用：通过 `svm_get_prop()` 在红黑树中查找 VA，确定地址属于 Host 还是 Device，获取对应的 `devid`。若不在 SVM 管理范围内，则视为 Host 地址（`devid = svm_get_host_devid()`）。

#### 3.2 拷贝方向分发

`svm_memcpy_sync()` 根据 src/dst 的 devid 判断传输方向，并调用对应的处理函数：

```
copy_dir_get_by_devid(src_devid, dst_devid):
  src==Host && dst==Host  → SVM_H2H_CPY
  src==Host && dst==Dev   → SVM_H2D_CPY  ← halMemcpy典型场景
  src==Dev  && dst==Host  → SVM_D2H_CPY
  src==Dev  && dst==Dev   → SVM_D2D_CPY

函数表 g_sync_copy[]:
  [SVM_H2H_CPY] = svm_mem_sync_copy_h2h
  [SVM_H2D_CPY] = svm_mem_sync_copy_h2d
  [SVM_D2H_CPY] = svm_mem_sync_copy_d2h
  [SVM_D2D_CPY] = svm_mem_sync_copy_d2d
```

#### 3.3 H2D 拷贝详细流程（svm_mem_sync_copy_h2d）

```
svm_mem_sync_copy_h2d(src_info, dst_info)
  └─ _svm_mem_sync_copy(SVM_H2D_CPY, src_info, dst_info)
       │
       ├─ host_va = src_info->va  // Host侧源地址
       ├─ 检查 host_va 是否在 SVM 管理范围内
       │    ├─ 在范围内 → 直接拷贝
       │    └─ 不在范围内（如 malloc 的普通 Host 内存）
       │         └─ svm_register_to_master(user_devid, &register_va, flag)
       │              // 将Host内存注册到Master，PIN住物理页，建立设备侧页表映射
       │              // flag = REGISTER_TO_MASTER_FLAG_PIN
       │              // 使NPU设备能通过SMMU访问Host物理内存
       │
       ├─ svm_sync_copy(src_info, dst_info)
       │    └─ [根据设备连接方式适配]
       │         ├─ PCIe模式: svm_pci_sync_copy()
       │         │    ├─ ioctl(SVM_ASYNC_COPY_SUBMIT, para)
       │         │    │    └─ [Kernel] DMA引擎配置 H2D 传输描述符
       │         │    └─ ioctl(SVM_ASYNC_COPY_WAIT, para)
       │         │         └─ [Kernel] 轮询/中断等待DMA完成
       │         └─ UB模式: svm_ub_sync_copy()
       │
       └─ [若注册了Host内存] svm_unregister_to_master() // 解除PIN
```

**Host 内存注册机制**（关键优化）：当 src 是普通的 `malloc`/`new` 分配的 Host 内存（不在 SVM 管理的 VA 范围内），NPU 设备无法直接通过 SMMU 访问。`svm_register_to_master()` 的作用是：
1. 将这片 Host VA 对应的物理页 PIN 住（防止被换出）
2. 在设备侧 SMMU 中建立页表映射
3. 返回后设备即可通过 DMA 直接读写这片 Host 内存

#### 3.4 内核态 DMA 拷贝执行（dma_copy.c）

内核态 `svm_dma_sync_cpy()` 负责实际调用硬件 DMA 引擎完成数据传输：

- **传输策略**：
  - `< 1KB`：PCIe MSG 模式（小数据量使用消息传递），等待方式为 QUERY（轮询）
  - `1KB ~ 256KB`：Traffic 模式 + QUERY 等待
  - `> 256KB`：Traffic 模式 + INTR 等待（中断通知）

- **DMA 队列拥塞控制**：
  - 当同时使用 QUERY 等待的线程数达到 8 时，自动切换为 INTR 模式
  - 使用 `ka_atomic_t copy_num` 原子计数跟踪并发线程数

- **同步拷贝**：通过 `hal_kernel_devdrv_dma_sync_link_copy_plus()` 提交 DMA 描述符并等待完成。当 DMA 队列满（`-ENOSPC`）时，最多重试 5000 次，每次间隔 100~200μs。

- **异步拷贝**：`svm_dma_async_cpy()` 使用 `hal_kernel_devdrv_dma_async_link_copy_plus()` 提交异步传输，完成后通过 `finish_notify` 回调通知。队列满时降级为同步拷贝。

#### 3.5 共享内存的拷贝路径解析

当 src 或 dst 是跨进程共享内存（IPC 或 VMM）时，需要解析真实物理地址所在进程：

```
svm_copy_try_resolve_shared_info(src_info, dst_info)
  ├─ svm_ipc_query_src_info()  // 查询IPC共享内存的原始信息
  └─ vmm_query_ipc_src_info()  // 查询VMM共享内存的原始信息
       │
       └─ svm_copy_real_info_pack()
            ├─ uda_get_devid_by_udevid_ex() // 将统一设备ID转为真实设备ID
            └─ halQueryMasterPidByDeviceSlave() // 查询Master进程tgid
```

对于 PCIe 连接的设备，共享内存解析后若 src 和 dst 在同一设备，直接走本地拷贝 `svm_memcpy_local_client()`，避免不必要的 H2D 传输。

#### 3.6 halMemcpy(H2D) 完整调用链总结

```
drvMemcpy(dst=DeviceVA, dest_max, src=HostVA, byte_count)
  │
  ├─ drvMemcpyInner(dst, dest_max, src, byte_count)
  │    ├─ svm_copy_check_addr_prop_cap(dst, SVM_FLAG_CAP_SYNC_COPY)
  │    │    └─ svm_get_prop() → 确认Device内存具有拷贝能力
  │    ├─ svm_copy_check_addr_prop_cap(src, SVM_FLAG_CAP_SYNC_COPY)
  │    │    └─ svm_get_prop() → 确认内存具有拷贝能力
  │    ├─ svm_set_copy_va_info(src) → info.devid = svm_get_host_devid()
  │    ├─ svm_set_copy_va_info(dst) → info.devid = prop.devid (NPU设备ID)
  │    └─ svm_memcpy_sync(&src_info, &dst_info)
  │         ├─ copy_dir_get_by_devid() → SVM_H2D_CPY
  │         ├─ [PCIe共享解析] svm_copy_try_resolve_shared_info()
  │         └─ svm_mem_sync_copy(src_info, dst_info)
  │              └─ g_sync_copy[SVM_H2D_CPY]
  │                   └─ svm_mem_sync_copy_h2d(src_info, dst_info)
  │                        └─ _svm_mem_sync_copy(SVM_H2D_CPY, src, dst)
  │                             ├─ [HostVA不在SVM范围]
  │                             │    └─ svm_register_to_master()
  │                             │         ├─ PIN Host物理页
  │                             │         └─ 建立设备SMMU映射
  │                             ├─ svm_sync_copy(src, dst)
  │                             │    └─ g_copy_ops[devid]->sync_copy()
  │                             │         └─ svm_pci_sync_copy()
  │                             │              ├─ ioctl(SVM_ASYNC_COPY_SUBMIT)
  │                             │              │    └─ [Kernel] DMA_Ctrl
  │                             │              │         ├─ 查询src/dst物理地址
  │                             │              │         │    (通过Host页表/SMMU页表)
  │                             │              │         ├─ 构造DMA描述符
  │                             │              │         │    (src物理地址→dst物理地址)
  │                             │              │         └─ DMA引擎启动传输
  │                             │              └─ ioctl(SVM_ASYNC_COPY_WAIT)
  │                             │                   └─ [Kernel] 等待DMA完成
  │                             │                        ├─ QUERY: 轮询DMA状态寄存器
  │                             │                        └─ INTR: 等待DMA完成中断
  │                             └─ [若注册了Host内存]
  │                                  └─ svm_unregister_to_master() // 解除PIN
  └─ return DRV_ERROR_NONE
```

### 4. 核心数据结构与设计模式

| 组件 | 数据结构 | 核心设计 |
|------|----------|----------|
| **句柄管理** (malloc_mng) | `handle_t` + 红黑树 `rbtree_root` | 按 VA 范围索引，支持快速查找、插入、删除；原子引用计数保护并发访问 |
| **VA 分配器** (va_allocator) | VA 范围位图/区间树 | 从预分配的虚拟地址池中按对齐要求分配连续 VA |
| **MPL 物理页管理** | `ka_page_t*[]` + `svm_pgtlb_attr` | 内核态物理页数组 + 页表属性包，支持 Normal/Huge/Giant 三种粒度 |
| **PMM 物理段追踪** | `pmm_seg` 链表 | 跟踪每个 VA 段的物理页状态（WITH_PA/WITHOUT_PA） |
| **KSVMM 共享内存** | `ksvmm_seg` + `range_rbtree` | 管理跨进程共享的内存段，记录源信息 + 引用计数 |
| **PMA P2P 页表** | `pma_mem_node` + `p2p_page_table` | 管理 P2P BAR 空间的物理页表，支持 kref 引用计数 |
| **Pipeline 并发控制** | `pthread_rwlock_t` | 读写锁：常规操作读锁并发，设备复位写锁独占 |
| **设备间通信** | UMC H2D/D2H 消息 | 用户态 Host → Device CP1 通信，传递 populate/depopulate/memcpy 命令 |

### 5. 关键设计要点

1. **虚拟地址与物理地址分离**：VA 分配（mmap VMA 创建）和物理页分配（populate + remap）是两个独立步骤，支持 VMM 的灵活映射场景，也便于后续实现 demand paging。

2. **Host/Device 双路径设计**：Host 侧内存操作通过本地 ioctl 直接进入内核；Device 侧内存操作通过 UMC 消息转发到设备侧 CP1 进程执行。这保证了设备侧内存管理的隔离性和正确性。

3. **多粒度页支持**：Normal Page（如 4KB/64KB）、Huge Page（如 2MB）、Giant Page（如 1GB）三级页粒度，根据 flag 和物理段对齐情况自动选择最优粒度。

4. **Pipeline 读写锁**：`svm_use_pipeline()`/`svm_unuse_pipeline()` 确保常规 alloc/free/copy 操作并发执行，同时保证设备热复位等独占操作不被饿死。

5. **Cache 与 Normal 双路分配**：Cache 层缓存已释放的物理页，下次分配时直接复用，减少内核态页分配开销。Cache 不可用时退化为 Normal 路径。

6. **DMA 传输自适应**：根据数据大小自动选择最优传输模式（MSG/Traffic）和等待方式（QUERY/INTR），兼顾小数据低延迟和大数据高吞吐。

7. **Host 内存动态注册**：对于非 SVM 管理的 Host 内存，通过 `svm_register_to_master()` 动态 PIN 住物理页并建立 SMMU 映射，使设备能透明访问任意 Host 内存。

---