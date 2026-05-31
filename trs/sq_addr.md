## `halSqCqAllocate` 端到端 SQ 地址追溯

### 第 1 层：用户态 API 入口

**`ascend_hal_base.h:4553`** → **`trs_interface.c:240`**

```c
DLLEXPORT drvError_t halSqCqAllocate(uint32_t devId, struct halSqCqInputInfo *in, struct halSqCqOutputInfo *out);
```

内部调用 `trs_sqcq_alloc()` 或 `trs_sqcq_urma_alloc()`，取决于连接类型（PCIE/HCCS/UB）。

---

### 第 2 层：用户态准备阶段 — `trs_local_sqcq_alloc()`

**`trs_sqcq.c:971`**

```
trs_sqcq_alloc()
  └─ trs_local_sqcq_alloc(dev_id, in, out)
```

此函数执行 3 个关键步骤：

#### 步骤 2a：SQ 队列内存分配（条件性）

```c
// trs_sqcq.c:979
if ((trs_is_support_sq_mem_ops(dev_id, in)) && (trs_get_sqcq_mem_ops()->mem_alloc != NULL)) {
    in->flag |= TSDRV_FLAG_SPECIFIED_SQ_MEM;
    ret = trs_get_sqcq_mem_ops()->mem_alloc(dev_id, (uint64_t *)&sq_que_va, ...);
}
```

`mem_alloc` 是用户态的内存分配回调（如 SVM 分配），返回用户态虚拟地址 `sq_que_va`。此地址会通过 `uio_info` 传递给内核。

#### 步骤 2b：MMAP 设备内存

```c
// trs_sqcq.c:201
int trs_sq_mmap(uint32_t dev_id, struct halSqCqInputInfo *in, struct trs_sq_map *sq_map)
```

通过 `trs_dev_mmap()` 映射设备内存（通过 `/dev/xxx` 设备的 mmap）到用户空间，包括：
- **non_reg_addr**: SQ queue 内存区域 + 控制寄存器区域（head/tail/db）
- **reg_addr**: 额外的寄存器区域

`sq_map->que.addr` = SQ 队列的用户态虚拟地址（如果没有指定 SQ mem）。

#### 步骤 2c：填充分配信息

```c
// trs_sqcq.c:252
static void trs_fill_sq_alloc_info(in, uio_info, sq_map, sq_que_va)
{
    para->uio_info = uio_info;
    uio_info->sq_que_addr = sq_que_va ?: sq_map->que.addr;
    // 设置各控制寄存器的地址
}
```

`uio_info->sq_que_addr` 就是 **SQ 队列的用户态虚拟地址**，后续通过 ioctl 传给内核。

---

### 第 3 层：IOCTL 进入内核

```c
// trs_sqcq.c:1006
ret = trs_dev_ioctl(fd, TRS_SQCQ_ALLOC, in);
```

**`trs_dev_drv.c:590`**: `trs_dev_io_ctrl()` → `ioctl(fd, TRS_SQCQ_ALLOC, in)`

ioctl 命令号定义在 `trs_ioctl.h:164`:

```c
#define TRS_SQCQ_ALLOC  _IOWR('X', 15, struct halSqCqInputInfo)
```

---

### 第 4 层：内核 IOCTL 处理 — `ioctl_trs_sqcq_alloc()`

**`trs_fops.c:220`**

```c
static int ioctl_trs_sqcq_alloc(struct trs_proc_ctx *proc_ctx, unsigned int cmd, unsigned long arg)
{
    // 1. copy_from_user: 从用户态拷贝 halSqCqInputInfo
    // 2. copy_from_user: 从用户态拷贝 trs_uio_info (包含 sq_que_addr)
    // 3. 根据 type 分发到对应的分配函数
    ret = trs_sqcq_alloc_handles[para.type](proc_ctx, ts_inst, &para);
    // 4. copy_to_user: 将结果写回用户态
}
```

**分发表**（`trs_fops.c:203`）：

```c
static int (*const trs_sqcq_alloc_handles[...]) = {
    [DRV_NORMAL_TYPE]  = trs_hw_sqcq_alloc,     // ← HW 类型走这里
    [DRV_CALLBACK_TYPE] = trs_cb_sqcq_alloc,
    [DRV_LOGIC_TYPE]   = trs_logic_cq_alloc,
    [DRV_SHM_TYPE]     = trs_shm_sqcq_alloc,
    [DRV_CTRL_TYPE]    = trs_sw_sqcq_alloc,
    [DRV_GDB_TYPE]     = trs_gdb_sqcq_alloc,
};
```

---

### 第 5 层：内核通道创建 — `trs_hw_sqcq_alloc_pair()`

**`trs_hw_sqcq.c:556`**

```
trs_hw_sqcq_alloc()  →  trs_hw_sqcq_alloc_pair()
```

#### 步骤 5a：分配通道（Channel）

```c
ret = trs_hw_alloc_chan(proc_ctx, &ts_inst->inst, para, &chan_id);
```

内部调用 `hal_kernel_trs_chan_create()` → 通道层创建 SQ/CQ。

#### 步骤 5b：通道创建时分配 SQ 内存

**`chan_init.c:238`** — `trs_chan_sq_mem_init()`

```c
// 1. 查询 head/tail/db 的物理地址
ts_inst->ops.sqcq_query(..., QUERY_CMD_SQ_HEAD_PADDR, &sq->head_addr);
ts_inst->ops.sqcq_query(..., QUERY_CMD_SQ_TAIL_PADDR, &sq->tail_addr);
ts_inst->ops.sqcq_query(..., QUERY_CMD_SQ_DB_PADDR,   &sq->db_addr);

// 2. 分配 SQ 队列内存（关键！）
sq->sq_addr = ts_inst->ops.sq_mem_alloc(inst, &chan->types, &sq->para, &sq->mem_attr);
//    ↑ 返回内核虚拟地址，sq->mem_attr.phy_addr 被设为物理地址
```

`sq_mem_alloc` 的实现位于 LBA 适配层（如 `stars_v1/tscpu/stars_v2`），负责分配物理内存并返回内核虚拟地址，同时设置 `mem_attr.phy_addr`。

如果是 SRIOV VM VF 场景，不分配本地内存：

```c
if ((ts_inst->location == UDA_REMOTE) && ...) {
    sq->mem_attr.phy_addr = TRS_INVALID_PHYS_ADDR;
    return 0;
}
```

#### 步骤 5c：获取 SQ 信息

```c
// trs_hw_sqcq.c:609
(void)trs_chan_get_sq_info(inst, chan_id, &sq_info);
```

最终调用 `_trs_chan_get_sq_info()`（`chan_init.c:68`）：

```c
info->sq_phy_addr = chan->sq.mem_attr.phy_addr;  // ← 物理地址来源
```

---

### 第 6 层：SQ 重映射与地址覆盖 — `trs_sq_remap()`

**`trs_sqcq_map.c:294`** → **`trs_hw_sqcq.c:614`**

```c
(void)trs_sq_remap(proc_ctx, ts_inst, para, &ts_inst->sq_ctx[sq_info.sqid], &sq_info);
```

在 `soft_que` 模式下（有 trigger irq + normal type），会：

```c
// 1. 分配新的 SQ 队列内存
trs_sq_mem_alloc(sq_ctx, que_mem_len);
    └─ sq_ctx->que_mem.kva = ka_alloc_pages_exact_ex(...)  // 内核虚拟地址

// 2. 用新内存的物理地址覆盖 sq_phy_addr
trs_refill_sq_info(sq_ctx, sq_info);
    └─ sq_info->sq_phy_addr = ka_mm_virt_to_phys(sq_ctx->que_mem.kva)
```

非 soft_que 模式则直接使用通道返回的 `sq_info->sq_phy_addr` 进行 remap。

---

### 第 7 层：Mailbox 通知 TS 端

**`trs_hw_sqcq.c:711`** — 通过配对模式或非配对模式的通知路径

```c
static void trs_fill_notice_ts_msg(..., struct trs_chan_sq_info *sq_info, ...)
{
    msg->sq_addr = sq_info->sq_phy_addr;  // ← 最终物理地址填入 mailbox
}
```

此 `msg` 通过 `trs_mbox_send()` 发送给 TS 子系统（设备侧），TS 使用此物理地址访问 SQ 队列。

---

## 全景流程图

```
用户态 (ascend_hal library)
─────────────────────────────────────────────────────────
halSqCqAllocate(devId, in, out)
  │
  ├─ [可选] trs_get_sqcq_mem_ops()->mem_alloc()
  │   └─ SVM 分配 → sq_que_va (用户态虚拟地址)
  │
  ├─ trs_sq_mmap() → mmap 设备内存
  │   └─ sq_map->que.addr (用户态虚拟地址)
  │
  ├─ trs_fill_sq_alloc_info()
  │   └─ uio_info->sq_que_addr = sq_que_va ?: sq_map->que.addr
  │
  └─ ioctl(fd, TRS_SQCQ_ALLOC, in)  ──────────────┐
                                                    │
内核态 (trsdrv kernel module)                       │
─────────────────────────────────────────────────────┘
ioctl_trs_sqcq_alloc()
  │ copy_from_user: halSqCqInputInfo + trs_uio_info
  │
  └─ trs_hw_sqcq_alloc() → trs_hw_sqcq_alloc_pair()
      │
      ├─ trs_hw_alloc_chan()
      │   └─ hal_kernel_trs_chan_create()
      │       └─ trs_chan_sq_mem_init()
      │           ├─ ops.sqcq_query(): head/tail/db 物理地址
      │           └─ ops.sq_mem_alloc(): 分配 SQ 队列内存
      │               └─ sq->mem_attr.phy_addr = 物理地址
      │
      ├─ trs_chan_get_sq_info()
      │   └─ sq_info.sq_phy_addr = chan->sq.mem_attr.phy_addr
      │
      ├─ trs_sq_remap()  [可选: soft_que 模式]
      │   ├─ ka_alloc_pages_exact_ex() → que_mem.kva (新分配)
      │   └─ trs_refill_sq_info()
      │       └─ sq_info.sq_phy_addr = ka_mm_virt_to_phys(que_mem.kva)
      │
      ├─ trs_sqcq_reg_map()
      │
      └─ trs_fill_notice_ts_msg()
          └─ msg->sq_addr = sq_info.sq_phy_addr  →  mailbox 送 TS
```

---

### 关键要点

| 地址类型 | 位置 | 含义 |
|---------|------|------|
| `sq_que_va` | 用户态（ascend_hal） | 用户态虚拟地址（SVM 分配），通过 `uio_info` 传给内核 |
| `sq_map->que.addr` | 用户态（ascend_hal） | mmap 得到的用户态虚拟地址 |
| `sq->sq_addr` | 内核（chan_init） | 内核虚拟地址（`sq_mem_alloc` 返回） |
| `sq->mem_attr.phy_addr` | 内核（chan_init） | **SQ 队列物理地址**（核心来源） |
| `sq_info.sq_phy_addr` | 内核（trs_hw_sqcq） | 传递给 mailbox 的物理地址 |
| `que_mem.kva` | 内核（soft_que） | 重新分配的内核虚拟地址（用于 UIO 触发模式） |
| `msg->sq_addr` | 内核 mailbox | 最终发给 TS 的物理地址 |


------------------------------
# SQ地址关系梳理
好的，我来总结一下所有 SQ 地址的关联关系。整个代码库中涉及 **三类不同的物理地址** 和 **三类不同的虚拟地址**。

---

## 所有 SQ 地址总览

| 符号 | 所在层 | 地址类型 | 含义 |
|------|--------|---------|------|
| `sq_que_va` | 用户态 ascend_hal | 用户虚拟地址 | SVM 分配的队列内存 |
| `sq_map->que.addr` | 用户态 ascend_hal | 用户虚拟地址 | mmap 获得的队列内存 |
| `uio_info->sq_que_addr` | 用户态 ascend_hal | 用户虚拟地址 | 传给内核的用户队列地址 |
| `chan->sq.sq_addr` | 内核通道层 | 内核虚拟地址 | 通道分配的 SQ 队列内存 |
| `chan->sq.mem_attr.phy_addr` | 内核通道层 | **物理地址** | SQ 队列真正的物理地址 |
| `chan->sq.head_addr` | 内核通道层 | **物理地址** | SQ head 寄存器地址 |
| `chan->sq.tail_addr` | 内核通道层 | **物理地址** | SQ tail 寄存器地址 |
| `chan->sq.db_addr` | 内核通道层 | **物理地址** | SQ doorbell 寄存器地址 |
| `sq_info.sq_phy_addr` | 内核核心层 | **物理地址** | 从通道获取的 SQ 物理地址 |
| `sq_ctx->que_mem.kva` | 内核核心层 | 内核虚拟地址 | soft_que 模式新分配的内存 |
| `sq_ctx->que_mem.uva` | 内核核心层 | 用户虚拟地址 | remap 后的用户地址 |
| `mbox_data.sq_addr` | mailbox 层 | **物理地址** | 最终发给 TS 的物理地址 |
| `out->queueVAddr` | 输出给用户 | 用户虚拟地址 | 返回给用户的 SHM 地址 |

---

## 三种场景下的地址关联关系

### 场景①：普通模式（非 soft_que、非指定内存）

```
┌─ 用户态 ──────────────────────────────────────────────────────┐
│  sq_map->que.addr  ← trs_dev_mmap() 得到的用户虚拟地址        │
│  uio_info->sq_que_addr = sq_map->que.addr                     │
└───────────────────────────┬─────────────────────────────────────┘
                            │ ioctl(TRS_SQCQ_ALLOC)
                            ▼
┌─ 内核 ────────────────────────────────────────────────────────┐
│  [1] 通道创建 sq_mem_alloc():                                  │
│       chan->sq.sq_addr          = 内核虚拟地址 (kva)           │
│       chan->sq.mem_attr.phy_addr = 物理地址    (pa)  ◄── 源头  │
│                                                                │
│  [2] trs_chan_get_sq_info():                                   │
│       sq_info.sq_phy_addr = chan->sq.mem_attr.phy_addr         │
│                                                                │
│  [3] trs_sq_remap() [soft_que=0]:                              │
│       sq_info.sq_phy_addr 不变                                 │
│       trs_remap_fill_para(uio_info->sq_que_addr,              │
│                           sq_info->sq_phy_addr)                │
│       将物理地址 remap 到用户地址 sq_que_addr                   │
│       sq_ctx->que_mem.uva = map_para->va                       │
│                                                                │
│  [4] mailbox 通知 TS:                                          │
│       mbox_data.sq_addr = sq_info.sq_phy_addr (= chan 的 pa)   │
└────────────────────────────────────────────────────────────────┘
```

**关系**：`sq_info.sq_phy_addr` = `chan->sq.mem_attr.phy_addr` = 发给 TS 的物理地址。  
该物理地址同时被 remap 到用户态 UIO 地址 `sq_ctx->que_mem.uva`。

---

### 场景②：soft_que 模式（有 trigger irq + normal type）

```
┌─ 用户态 ──────────────────────────────────────────────────────┐
│  sq_map->que.addr  ← trs_dev_mmap()                           │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─ 内核 ────────────────────────────────────────────────────────┐
│  [1] 通道创建 sq_mem_alloc():                                  │
│       chan->sq.mem_attr.phy_addr = pa_1   (原始物理地址)       │
│                                                                │
│  [2] trs_chan_get_sq_info():                                   │
│       sq_info.sq_phy_addr = pa_1                               │
│                                                                │
│  [3] trs_sq_remap() [soft_que=1]:  ← 关键不同点                │
│        │                                                       │
│        ├─ trs_sq_mem_alloc(sq_ctx, len):                       │
│        │    sq_ctx->que_mem.kva = ka_alloc_pages_exact_ex()    │
│        │    └─ 分配新内存 → 内核虚拟地址 kva_new               │
│        │                                                       │
│        └─ trs_refill_sq_info(sq_ctx, sq_info):                 │
│             sq_info->sq_phy_addr = ka_mm_virt_to_phys(kva_new) │
│             = pa_2  ◄── 覆盖原始地址！                          │
│                                                                │
│  [4] remap: 用 pa_2 映射到用户地址                             │
│       sq_ctx->que_mem.uva = remap(pa_2 → 用户)                 │
│                                                                │
│  [5] mailbox:                                                  │
│       mbox_data.sq_addr = sq_info.sq_phy_addr = pa_2           │
│       而不是 pa_1！                                              │
└────────────────────────────────────────────────────────────────┘
```

**关系**：`pa_1`（通道分配的原始物理地址）**被丢弃**，`pa_2`（新分配的 `ka_alloc_pages_exact_ex` 物理地址）取而代之，既用于 UIO remap，也通过 mailbox 发给 TS。

**为什么要覆盖？** soft_que 模式下需要 UIO trigger 能力，通道原始分配的 SQ 内存可能不支持 UIO trigger，所以重新分配一块内核内存，用新物理地址替换。

---

### 场景③：指定 SQ 内存（`TSDRV_FLAG_SPECIFIED_SQ_MEM`）

```
┌─ 用户态 ──────────────────────────────────────────────────────┐
│  sq_que_va = trs_get_sqcq_mem_ops()->mem_alloc()  ← SVM 分配  │
│  uio_info->sq_que_addr = sq_que_va  (用户虚拟地址)             │
│  注意：不 mmap 队列内存 (que_mmap_len = 0)                    │
└───────────────────────────┬─────────────────────────────────────┘
                            │ ioctl
                            ▼
┌─ 内核 ────────────────────────────────────────────────────────┐
│  [1] 通道创建:                                                 │
│       chan_para.sq_para.sq_que_uva = uio_info->sq_que_addr    │
│       chan_para.flag |= CHAN_FLAG_SPECIFIED_SQ_MEM_BIT        │
│       仍调用 sq_mem_alloc() → chan->sq.mem_attr.phy_addr = pa │
│                                                                │
│  [2] trs_chan_get_sq_info() → sq_info.sq_phy_addr = pa        │
│                                                                │
│  [3] trs_sq_remap():                                           │
│       if (SPECIFIED_SQ_MEM 被设置)                             │
│          跳过 sq_que_addr 的物理 remap!                        │
│          用户已自行管理内存映射，内核不再覆盖                   │
│                                                                │
│  [4] mailbox:                                                  │
│       mbox_data.sq_addr = sq_info.sq_phy_addr = pa            │
│       (仍是通道分配的物理地址，非用户 SVM 地址)                 │
└────────────────────────────────────────────────────────────────┘
```

**关系**：用户 `sq_que_va`（SVM 虚拟地址）和内核 `sq_info.sq_phy_addr`（通道分配的物理地址）**是两个不同的内存**。用户的 SVM 内存用于 enqueue/dequeue 操作（用户侧），而通道分配的物理地址用于 mailbox 通知 TS（实现跨设备同步）。

---

## 全景关系矩阵

| 地址 | 拥有者 | 关联关系 |
|------|-------|---------|
| `chan->sq.sq_addr` (kva) | SQ内存分配者 | 通过 `ka_mm_virt_to_phys()` → `mem_attr.phy_addr` |
| `chan->sq.mem_attr.phy_addr` (pa) | 通道层 | **所有物理地址的源头**（除非被 soft_que 覆盖）|
| `sq_info.sq_phy_addr` (pa) | 核心层 | = `mem_attr.phy_addr`（普通模式）或 = `ka_mm_virt_to_phys(que_mem.kva)`（soft_que）|
| `sq_ctx->que_mem.kva` (kva) | soft_que | → `ka_mm_virt_to_phys()` → 覆盖 `sq_phy_addr` |
| `sq_ctx->que_mem.uva` (uva) | 核心层 | = `remap(sq_phy_addr)` 映射到用户空间 |
| `mbox_data.sq_addr` (pa) | mailbox | = `sq_info.sq_phy_addr`，发给 TS |
| `sq_que_va` (用户va) | ascend_hal | 仅 `SPECIFIED_SQ_MEM` 时有意义，用户自管内存 |
| `out->queueVAddr` (用户va) | 返回给用户 | 仅 `DRV_SHM_TYPE` 有效，= `sq_map->que.addr` |

**一个核心设计思想**：正常路径下，**三份地址指向同一段物理内存**（通道分配的 SQ 队列内存），只是表现形式不同（内核虚拟、物理、用户虚拟地址），通过 `ka_mm_virt_to_phys()`、`remap()` 等在不同视角间转换。只有 soft_que 模式会"偷梁换柱"，用新分配的内存物理地址替换原始的。