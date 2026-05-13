## TRS (Task Resource Scheduler) 模块总结

### 一、模块概述

TRS 是 **Task Resource Scheduler**（任务资源调度器），是华为 Ascend NPU 驱动中负责**任务调度、资源管理、进程间同步**的核心模块。它管理硬件队列（SQCQ）、事件、通知、流（Stream）等资源的分配、注册、映射和生命周期。

代码分为两层：
- **trs** — 用户态层 (userspace HAL)
- **trsdrv** — 内核态层 (kernel driver)

---

### 二、代码目录结构

```
src/ascend_hal/trs/          (用户态 HAL)
├── core/                    核心功能
│   ├── command/msg/         与TS CPU通信的消息/命令
│   ├── urma/                URMA (RDMA) 异步接口
│   ├── trs_interface.c      对外接口 (halResourceConfig, halStreamTaskFill等)
│   ├── trs_sqcq.c           SQCQ 用户态管理
│   ├── trs_res.c            资源管理
│   ├── trs_cb_event.c       Callback 事件处理
│   └── trs_dev_drv.c        设备驱动上下文
├── dc/                      数据结构 (STL)
├── inc/                     公共头文件
├── remote/master/           远程主控端事件/SQ内存管理
└── shr_id/                  共享资源ID (fd管理, spod通信)

src/sdk_driver/trsdrv/       (内核态驱动)
├── trs/                     TRS 内核主模块
│   ├── trs_core/            核心实现
│   │   ├── trs_hw_sqcq.c    硬件SQCQ (直接操作硬件寄存器)
│   │   ├── trs_sw_sqcq.c    软件SQCQ (通过门铃/消息通知TS CPU)
│   │   ├── trs_shm_sqcq.c   SHM共享内存SQCQ
│   │   ├── trs_shr_sqcq.c   共享SQCQ上下文
│   │   ├── trs_cb_sqcq.c    Callback SQCQ
│   │   ├── trs_gdb_sqcq.c   GDB调试SQCQ
│   │   ├── trs_logic_cq.c   逻辑CQ (软件CQE队列)
│   │   ├── trs_sqcq_map.c   SQCQ地址映射
│   │   ├── trs_res_mng.c    资源管理
│   │   ├── trs_fops.c       文件操作接口
│   │   ├── trs_proc.c/proc_fs.c 进程管理和procfs
│   │   └── trs_ts_inst.c    TS实例管理
│   ├── shr_id/              共享资源ID管理
│   │   ├── trs_shr_id.c     共享ID核心
│   │   ├── trs_shr_id_node.c 共享ID节点
│   │   ├── trs_shr_id_fops.c 共享ID文件操作
│   │   └── trs_shr_id_spod* SPOD (跨Pod)共享ID
│   ├── lba/                 负载均衡
│   │   ├── comm/            公共
│   │   └── near/            近端调度
│   ├── dc/                  数据结构
│   ├── stars/               STARS场景 (软硬件协同)
│   └── inc/                 头文件
├── trsbase/                 TRS基础层
│   ├── chan/                通道 (Channel) 管理 —— 核心传输抽象
│   ├── id_allocator/        ID分配器
│   └── inc/                 头文件 (trs_chan.h等)
```

---

### 三、核心设计原理

#### 3.1 分层架构

```
用户态应用
    ↕ ioctl / mmap
┌─────────────────────────────────┐
│ ascend_hal/trs (用户态HAL)      │ ← 封装ioctl调用，提供 hal* 接口
│  - trs_sqcq.c                   │
│  - trs_interface.c              │
│  - trs_shr_id_fd.c              │
└──────────┬──────────────────────┘
           ↓ ioctl
┌─────────────────────────────────┐
│ sdk_driver/trsdrv (内核态驱动)  │ ← 实际资源管理、调度
│  - trs/trs_core/                │
│  - trsbase/chan/                │
│  - trs/shr_id/                  │
└──────────┬──────────────────────┘
           ↓ 硬件交互 (MMIO/门铃/中断)
┌─────────────────────────────────┐
│ TS CPU / NPU硬件                │ ← 执行实际任务
└─────────────────────────────────┘
```

#### 3.2 资源ID系统（ID Resource System）

TRS 管理多种资源ID，每种ID有独立分配空间：

| 资源类型 | 用途 |
|---------|------|
| `TRS_STREAM` | 任务流 ID |
| `TRS_EVENT` | 事件 ID（同步/通知） |
| `TRS_NOTIFY` | 通知 ID |
| `TRS_HW_SQ/CQ` | 硬件发送/完成队列 ID |
| `TRS_SW_SQ/CQ` | 软件(控制面)队列 ID |
| `TRS_LOGIC_CQ` | 逻辑(聚合)完成队列 ID |
| `TRS_CB_SQ/CQ` | Callback 队列 ID |
| `TRS_MAINT_SQ/CQ` | 维护队列 ID |
| `TRS_CDQM` | CDQ 管理 ID |
| `TRS_CNT_NOTIFY` | 计数通知 ID |

**分配策略**：ID 可以在单进程内分配（local），也可以在进程间共享（shared），还可以跨 Pod 共享（SPOD）。

#### 3.3 通道模型（Channel — 核心抽象）

**Channel** 是 TRS 最核心的抽象层，定义在 trs_chan.h 中：

```
Channel Types (二维矩阵):
┌──────────────┬──────────┬─────────┬──────┬────────┐
│ 主类型\子类型 │ rts/esched│  logic  │ shm  │   cb   │
├──────────────┼──────────┼─────────┼──────┼────────┤
│ CHAN_TYPE_HW  │  硬件SQ  │    -    │  -   │   -    │
│ CHAN_TYPE_SW  │    -     │ 逻辑CQ  │ SHM  │   -    │
│ CHAN_TYPE_MNT │    -     │   -     │  -   │   -    │
│ CHAN_TYPE_CB  │    -     │   -     │  -   │ callback│
└──────────────┴──────────┴─────────┴──────┴────────┘
```

每种 Channel 包含：
- **SQ (Send Queue)** — 发送队列，放入 SQE (Send Queue Element)
- **CQ (Completion Queue)** — 完成队列，取出 CQE (Completion Queue Element)
- **Doorbell (门铃)** — 通知硬件有新任务
- **Head/Tail 指针** — 环形队列的读写指针

#### 3.4 三种 SQCQ 实现模式

| 模式 | 文件 | 原理 | 适用场景 |
|------|------|------|---------|
| **HW SQCQ** | trs_hw_sqcq.c | 直接操作硬件 MMIO 寄存器读写 head/tail，通过门铃触发 | 高性能、低延迟任务提交 |
| **SW SQCQ** | trs_sw_sqcq.c | 通过内核消息通知 TS CPU，由 TS CPU 代理执行 | 控制面操作、管理命令 |
| **SHM SQCQ** | trs_shm_sqcq.c | 通过共享内存环形队列通信，用于板内多核场景 | 同芯片内的核间通信 |

**`trs_hw_sq_send_task` 流程示例：**
```
1. 获取 SQ head/tail 指针 (MMIO)
2. while(head != tail):
   a. 从 SQ 中取出 SQE
   b. 调用 chan_send 发送
   c. 更新 tail 指针
3. 写门铃寄存器通知硬件
```

#### 3.5 共享资源 ID（Shr ID）机制

这是实现**多进程间共享资源**的核心机制：

```
进程A (创建者)                  进程B (共享者)
    │                              │
    │ ioctl(CREATE_NODE)           │
    ├─→ 内核分配 shr_id_node       │
    │    绑定 notify/event ID      │
    │    写入 proc_ctx 链表        │
    │                              │
    │ 通过 UNIX socket / fd        │
    │ 传递 shr_id 信息 ────────────→│
    │                              │ ioctl(OPEN_NODE)
    │                              ├─→ 查找已有的 shr_id_node
    │                              │    增加 refcount
    │                              │    映射到本进程地址空间
    │                              │
    │ 双方均可操作共享资源          │
    │ (notify/event 等)             │
```

支持两种共享范围：
- **Local (同设备内)** — 通过 `shr_id_proc_ctx` 管理的 proc 链表
- **SPOD (跨Pod)** — 通过 `trs_shr_id_spod.c` 实现跨设备消息传递

#### 3.6 进程间资源归属检查

TRS 实现了严格的资源归属检查系统：

```
trs_res_is_belong_to_proc(inst, pid, res_type, res_id)
    ↓
  查找 proc_ctx → 遍历 create_list / open_list
    ↓
  支持多级 PID 检查 (server_id → chip_id → die_id)
```

当进程退出时，`shr_id_proc_destroy` 会自动清理该进程创建/打开的所有资源节点。

#### 3.7 中断与事件处理

```
硬件完成
    ↓ 中断
trs_cb_event.c  (用户态)
    ↓
trs_event.h / trs_uk_msg.h (内核态)
    ↓
  CQE 解析 → sq_head 更新 → 唤醒等待线程
```

支持两种 SQ 发送模式：
- **High Security (UIK 模式)** — 加密方式提交，安全性高
- **High Performance (UIO 模式)** — 直接映射硬件寄存器，性能高

---

### 四、初始化流程

```
trs_init.c
  ├─→ trs_core_init_module()    核心初始化
  ├─→ init_trs_stars()          STARS 场景初始化
  ├─→ id_pool_setup_init()      ID 池初始化
  ├─→ tsmng_init()              TS 管理初始化
  ├─→ shr_id_init_module()      共享资源ID初始化
  ├─→ init_trs_adapt()/agent()  适配层初始化
  └─→ init_trs_sec_eh()         安全事件处理初始化
```

### 五、设计模式总结

| 设计模式 | 体现 |
|---------|------|
| **分层抽象** | Channel 层屏蔽硬件差异，SQCQ 层提供统一接口 |
| **适配器模式** | `trs_core_adapt_ops` 函数指针表适配不同硬件平台 |
| **观察者模式** | CQE 事件通知 → LogicCQ 分发 → 回调处理 |
| **引用计数** | shr_id 共享资源通过 `kref_safe` 管理生命周期 |
| **模板方法** | SQCQ 的 alloc/send/recv/free 流程固定，具体实现可替换 |
| **工厂模式** | ID 分配器根据类型创建不同资源节点 |

### 六、总结

TRS 是一个**面向 Ascend NPU 的任务调度与资源管理框架**，通过 Channel 抽象层统一了硬件队列、软件队列和共享内存三种通信方式，通过 Shr ID 机制实现了跨进程资源共享，通过多级 ID 类型管理了 Stream/Event/Notify/SQCQ 等全部调度资源，是连接用户态应用和 NPU 硬件执行单元的关键桥梁。