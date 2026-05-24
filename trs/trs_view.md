# TRS 模块 4+1 视图设计分析

## 一、逻辑视图 (Logical View) — 模块功能分解与分层

```mermaid
graph TB
    subgraph "用户态 (User Space)"
        USR_API["对外API层<br/>halResourceConfig / halSqCqAllocate<br/>halStreamTaskFill / halSqTaskSend"]
        USR_CORE["核心管理层<br/>trs_res (资源ID管理)<br/>trs_sqcq (SQ/CQ管理)<br/>trs_cb_event (回调事件)"]
        USR_DEV["设备接口层<br/>trs_dev_drv (open/close/ioctl/mmap)"]
        USR_REMOTE["远程通信层<br/>trs_master_urma (URMA)<br/>trs_master_event (事件同步)<br/>trs_master_async (异步DMA)"]
        USR_SHR["共享管理层<br/>trs_shr_id_user (共享ID)<br/>trs_shr_id_fd (共享FD)"]
    end

    subgraph "内核态 (Kernel Space)"
        KERN_BASE["基础服务层 (trsbase)<br/>trs_chan (通道抽象)<br/>id_allocator (ID分配器)"]
        
        subgraph "主模块 (trs)"
            KERN_CORE["核心层 (trs_core)<br/>trs_fops (文件操作)<br/>trs_proc (进程管理)<br/>trs_res_mng (资源管理)<br/>trs_sqcq_ctx (SQ/CQ上下文)"]
            KERN_SQCQ["SQ/CQ实现层<br/>trs_hw_sqcq (硬件SQ/CQ)<br/>trs_sw_sqcq (软件SQ/CQ)<br/>trs_shm_sqcq (共享内存)<br/>trs_shr_sqcq (共享模式)<br/>trs_cb_sqcq (回调模式)<br/>trs_gdb_sqcq (全局门铃)<br/>trs_logic_cq (逻辑CQ)"]
            KERN_SHR["共享ID层 (shr_id)<br/>trs_shr_id_node (节点管理)<br/>trs_shr_id_spod (SPOD共享)<br/>trs_shr_id_event (事件更新)"]
            KERN_LBA["负载均衡层 (lba)<br/>comm (通用适配)<br/>near (近端通信)<br/>sia (单实例Agent)<br/>sriov_sec (安全增强)"]
            KERN_STARS["软硬协同层 (trs_stars)<br/>soc (SoC适配)<br/>comm (通用通信)"]
        end
        
        KERN_ADAPT["适配接口层<br/>trs_core_adapt_ops (核心适配)<br/>trs_device_agent_ops (设备Agent)<br/>trs_sqcq_agent_ops (SQ/CQ Agent)"]
    end

    subgraph "NPU Device (TSCPU)"
        DEV_TS["Task Scheduler<br/>任务调度引擎<br/>队列管理<br/>中断处理"]
    end

    USR_API --> USR_CORE
    USR_CORE --> USR_DEV
    USR_CORE --> USR_REMOTE
    USR_CORE --> USR_SHR
    USR_DEV -.->|"ioctl / mmap"| KERN_CORE
    USR_REMOTE -.->|"URMA协议"| KERN_LBA
    KERN_CORE --> KERN_SQCQ
    KERN_CORE --> KERN_SHR
    KERN_CORE --> KERN_LBA
    KERN_CORE --> KERN_STARS
    KERN_SQCQ --> KERN_BASE
    KERN_SHR --> KERN_BASE
    KERN_LBA --> KERN_BASE
    KERN_STARS --> KERN_BASE
    KERN_ADAPT -.-> KERN_CORE
    KERN_BASE -.->|"PCIe/Mailbox/Interrupt"| DEV_TS
```

---

## 二、进程视图 (Process View) — 运行时交互与并发

```mermaid
sequenceDiagram
    participant APP as 用户进程 A
    participant APP2 as 用户进程 B (共享模式)
    participant U_TRS as 用户态 TRS 库
    participant K_TRS as 内核态 TRS 驱动
    participant TSCPU as NPU 设备 TSCPU

    rect rgb(200, 230, 255)
        Note over APP,TSCPU: === 初始化阶段 ===
        APP->>U_TRS: trs_file_open()
        U_TRS->>K_TRS: open("/dev/tsdrv") + ioctl(OPEN)
        K_TRS->>K_TRS: trs_proc_ctx 创建 (pid/mm绑定)
        U_TRS->>K_TRS: ioctl(STL_BIND)
        K_TRS->>TSCPU: mailbox: 通知TS绑定
        TSCPU-->>K_TRS: 返回绑定结果
        K_TRS-->>U_TRS: 返回FD/mmap地址
    end

    rect rgb(230, 255, 200)
        Note over APP,TSCPU: === 资源分配阶段 ===
        APP->>U_TRS: halResourceIdAlloc(stream_id)
        U_TRS->>K_TRS: ioctl(TRS_RES_ID_ALLOC)
        K_TRS->>K_TRS: id_pool_alloc() / trs_res_mng 分配
        K_TRS-->>U_TRS: 返回 stream_id
        
        APP->>U_TRS: halSqCqAllocate(sq_id, cq_id)
        U_TRS->>K_TRS: ioctl(TRS_SQCQ_GET)
        K_TRS->>K_TRS: trs_sq_ctx 创建 + chan 注册
        K_TRS->>TSCPU: mailbox: SQ/CQ配置信息
        TSCPU-->>K_TRS: 确认
        K_TRS-->>U_TRS: 返回 mmap 地址
    end

    rect rgb(255, 230, 200)
        Note over APP,TSCPU: === 任务提交阶段 ===
        APP->>U_TRS: halStreamTaskFill(task_info)
        U_TRS->>U_TRS: 填充 SQE 到 SQ 内存
        
        APP->>U_TRS: halSqTaskSend()
        alt UIO模式 (High Performance)
            U_TRS->>U_TRS: 写 Doorbell 寄存器 (mmap)
            U_TRS-->>TSCPU: 直接通知(无内核介入)
        else KIO模式 (High Security)
            U_TRS->>K_TRS: ioctl(TRS_SQ_CMD_SEND)
            K_TRS->>K_TRS: trs_chan_send() → mailbox
            K_TRS-->>TSCPU: 通过 mailbox 发送
        end
        TSCPU->>TSCPU: 调度执行任务
    end

    rect rgb(230, 200, 255)
        Note over APP,TSCPU: === 完成通知阶段 ===
        TSCPU->>TSCPU: 写 CQE 到 CQ
        alt 中断模式
            TSCPU->>K_TRS: 触发中断
            K_TRS->>K_TRS: 中断处理→CQ轮询→唤醒等待进程
            K_TRS-->>APP: 信号/wakeup
        else 轮询模式
            APP->>U_TRS: halSqCqQuery()
            U_TRS->>U_TRS: 读取 CQ 内存检查 CQE
        end
    end

    rect rgb(255, 200, 200)
        Note over APP,APP2: === 进程间共享 (shr_id) ===
        APP->>K_TRS: ioctl(SHR_ID_CONFIG)
        K_TRS->>K_TRS: trs_shr_id_node 创建共享节点
        APP2->>K_TRS: ioctl(SHR_ID_ATTACH)
        K_TRS->>K_TRS: 共享节点引用+1, 映射共享内存
        K_TRS-->>APP2: 返回共享 SQ/CQ 地址
        APP2->>U_TRS: 通过共享 SQ 提交任务
    end

    rect rgb(200, 255, 255)
        Note over APP,TSCPU: === 远程设备 (URMA) ===
        APP->>U_TRS: trs_sq_task_send_urma()
        U_TRS->>U_TRS: 封装 URMA 请求
        U_TRS->>K_TRS: ioctl(URMA 操作)
        K_TRS->>K_TRS: 跨节点地址转换
        K_TRS->>TSCPU: 通过 URMA 发送到远端
    end
```

**并发模型要点：**
- 每个用户进程有独立的 `trs_proc_ctx`，通过 `pid` 隔离
- 每个 (devid, tsid) 对对应一个 `trs_core_ts_inst`，持有 SQ/CQ/Stream 上下文
- `trs_sq_ctx` 使用 `ka_mutex_t` 保护 SQ 尾指针并发操作
- 中断处理使用 workqueue 延迟执行，避免在中断上下文中长时间运行
- 进程退出时通过 `trs_proc_ctx` 的 release 回调自动回收资源

---

## 三、开发视图 (Development View) — 源码模块组织与依赖

```mermaid
graph LR
    subgraph "用户态模块 (ascend_hal/trs)"
        direction TB
        U1["core/trs_interface.c<br/>对外API实现"]
        U2["core/trs_dev_drv.c<br/>设备打开/ioctl/mmap"]
        U3["core/trs_res.c<br/>资源ID管理"]
        U4["core/trs_sqcq.c<br/>SQ/CQ操作"]
        U5["core/trs_cb_event.c<br/>回调事件"]
        U6["remote/master/<br/>远程事件/DMA/缓存"]
        U7["core/urma/master/<br/>URMA主控"]
        U8["shr_id/<br/>共享ID用户态"]
        U9["inc/trs_user_pub_def.h<br/>公共定义"]
        U10["dc/trs_stl.h<br/>STL接口"]
    end

    subgraph "基础模块 (trsbase)"
        direction TB
        B1["chan/chan_init.c<br/>通道初始化"]
        B2["chan/chan_rxtx.c<br/>通道收发"]
        B3["chan/chan_abnormal.c<br/>通道异常"]
        B4["chan/chan_proc_fs.c<br/>通道procfs"]
        B5["chan/chan_trace.c<br/>通道trace"]
        B6["chan/chan_ts_inst.c<br/>TS实例绑定"]
        B7["id_allocator/trs_id.c<br/>ID分配器"]
        B8["inc/trs_chan.h<br/>通道头文件"]
        B9["inc/trs_id.h<br/>ID头文件"]
    end

    subgraph "主模块 (trs)"
        direction TB
        M1["trs_init.c<br/>模块入口/子模块编排"]
        M2["trs_core/trs_fops.c<br/>file_operations实现"]
        M3["trs_core/trs_proc.c<br/>进程上下文管理"]
        M4["trs_core/trs_res_mng.c<br/>资源管理器"]
        M5["trs_core/trs_ts_inst.c<br/>TS实例管理"]
        M6["trs_core/trs_sqcq_ctx.h<br/>SQ/CQ上下文定义"]
        M7["trs_core/trs_hw_sqcq.c<br/>硬件SQ/CQ"]
        M8["trs_core/trs_sw_sqcq.c<br/>软件SQ/CQ"]
        M9["trs_core/trs_shm_sqcq.c<br/>共享内存SQ/CQ"]
        M10["trs_core/trs_shr_sqcq.c<br/>共享模式SQ/CQ"]
        M11["trs_core/trs_cb_sqcq.c<br/>回调模式SQ/CQ"]
        M12["trs_core/trs_gdb_sqcq.c<br/>全局门铃SQ/CQ"]
        M13["trs_core/trs_logic_cq.c<br/>逻辑CQ"]
        M14["trs_core/trs_sqcq_map.c<br/>SQ/CQ内存映射"]
        M15["trs_core/trs_proc_fs.c<br/>procfs调试"]
        M16["trs_core/trs_shr_proc.c<br/>共享进程管理"]
        M17["shr_id/trs_shr_id.c<br/>共享ID核心"]
        M18["shr_id/trs_shr_id_node.c<br/>共享节点"]
        M19["shr_id/trs_shr_id_fops.c<br/>共享文件操作"]
        M20["lba/near/sia/<br/>SIA Agent适配"]
        M21["lba/near/sriov_sec/<br/>SR-IOV安全增强"]
        M22["trs_stars/adapt/soc/<br/>STARS SoC适配"]
        M23["trs_stars/comm/<br/>STARS通信"]
    end

    subgraph "头文件 (inc)"
        direction TB
        H1["trs_pub_def.h<br/>公共类型定义"]
        H2["trs_core.h<br/>核心接口/适配Ops"]
        H3["trs_adapt.h<br/>Agent适配接口"]
        H4["trs_pm_adapt.h<br/>电源管理适配"]
        H5["trs_device_agent.h<br/>设备Agent接口"]
        H6["trs_mia_adapt.h<br/>MIA适配"]
        H7["trs_sqe_update.h<br/>SQE更新"]
        H8["trs_cdqm.h<br/>CDQM管理"]
        H9["trs_stars_comm.h<br/>STARS通信"]
        H10["trs_event.h<br/>事件处理"]
        H11["trs_tsmng.h/trs_ts_hb.h<br/>TS管理/心跳"]
        H12["trs_chip_def_comm.h<br/>芯片定义"]
    end

    U1 --> U2 & U3 & U4
    U3 & U4 --> U6 & U7 & U8
    U1 --> U9 & U10

    M1 --> M2 & M4 & M5 & M14 & M16 & M20 & M22
    M2 --> M3 & M4 & M5
    M4 --> M6
    M5 --> M7 & M8 & M9 & M10 & M11 & M12 & M13
    M2 & M4 & M5 --> H1 & H2 & H3 & H4 & H5 & H6 & H7 & H8 & H9 & H10 & H11 & H12
    M2 --> B1 & B2 & B3
    M3 --> B7
    
    B1 & B2 & B3 & B4 & B5 --> B8 & B9
    subgraph "构建设置"
        C1["CMakeLists.txt<br/>add_subdirectory(trs)<br/>add_subdirectory(trsbase)"]
    end
    C1 --> B1 & M1
```

**依赖方向**：用户态 → 内核字符设备 → trs_core → trsbase → 硬件

---

## 四、物理视图 (Physical View) — 部署与硬件映射

```mermaid
graph TB
    subgraph "Host Server (x86/ARM)"
        subgraph "进程空间"
            APP1["AI Framework 进程<br/>(AscendCL/CANN)"]
            APP2["AI Framework 进程<br/>(共享模式)"]
        end
        
        subgraph "用户态库"
            U_TRS["libascend_hal.so<br/>(trs 用户态库)"]
            U_PBL["libpbl_uda.so<br/>(PBL 用户态)"]
        end
        
        subgraph "内核空间"
            K_TRS["trs_drv.ko<br/>(TRS 内核驱动)"]
            K_TRS_BASE["trs_base.ko<br/>(TRS 基础驱动)"]
            K_DMC["dmc.ko<br/>(设备管理)"]
            K_DPA["dpa.ko<br/>(DPA 驱动)"]
            K_HDC["hdc.ko<br/>(Host-Device通信)"]
        end
        
        subgraph "PCIe 硬件"
            PCIE_EP["PCIe Endpoint<br/>(BAR0/BAR2/BAR4)"]
        end
    end

    subgraph "NPU Device"
        subgraph "HBM (高带宽内存)"
            SQ_MEM["SQ Ring Buffers<br/>(提交队列内存)"]
            CQ_MEM["CQ Ring Buffers<br/>(完成队列内存)"]
            SHM_REGION["共享内存区域"]
            STREAM_MEM["Stream 任务数据"]
        end
        
        subgraph "TSCPU Core"
            TS_ENGINE["Task Scheduler<br/>调度引擎"]
            MAILBOX["Mailbox<br/>控制消息通道"]
            CDQ["Command Doorbell Queue<br/>命令门铃队列"]
            CQ_POLL["CQ Monitor<br/>完成队列监控"]
        end
        
        subgraph "硬件加速器"
            AI_CORE["AI Core<br/>(矩阵计算)"]
            AI_CPU["AI CPU<br/>(向量计算/AICPU)"]
            DVPP["DVPP<br/>(图像/视频编解码)"]
        end
    end

    subgraph "其他节点 (RDMA网络)"
        REMOTE_DEV["远端 NPU 设备"]
    end

    APP1 --> U_TRS
    APP2 --> U_TRS
    U_TRS --> U_PBL
    U_PBL -->|"mmap BAR"| PCIE_EP
    U_TRS -->|"ioctl"| K_TRS
    K_TRS --> K_TRS_BASE
    K_TRS --> K_DMC
    K_TRS --> K_DPA
    K_TRS --> K_HDC
    K_HDC -->|"PCIe Transaction"| PCIE_EP
    
    PCIE_EP -->|"DMA"| SQ_MEM
    PCIE_EP -->|"DMA"| CQ_MEM
    PCIE_EP -->|"DMA"| SHM_REGION
    
    MAILBOX -->|"中断"| K_TRS
    CQ_POLL -->|"MSI-X 中断"| K_TRS
    TS_ENGINE --> AI_CORE & AI_CPU & DVPP
    TS_ENGINE --> CDQ
    
    K_TRS -.->|"URMA over RDMA"| REMOTE_DEV

    subgraph "地址映射关系"
        direction LR
        MAP1["用户虚拟地址 (UVA)<br/>mmap返回"] --> MAP2["内核虚拟地址 (KVA)<br/>ioremap"] --> MAP3["物理地址 (PA)<br/>PCIe BAR / HBM"]
    end

    subgraph "中断路径"
        direction LR
        INT1["设备 MSI-X 中断"] --> INT2["中断控制器 (IOMMU)"] --> INT3["trs_chan_ops_request_irq<br/>注册的中断处理"] --> INT4["workqueue<br/>延迟处理CQ"]
    end
```

**关键物理映射关系：**

| 资源 | 物理位置 | 映射方式 |
|------|----------|----------|
| SQ 内存 | Host DRAM 或 Device HBM | mmap(BAR) 或 dma_alloc_coherent |
| CQ 内存 | Host DRAM 或 Device HBM | mmap(BAR) 或 dma_alloc_coherent |
| Doorbell 寄存器 | PCIe BAR 空间 | mmap 直接映射到用户态 (UIO) |
| Mailbox | 设备内部寄存器 | 通过 PCIe MMIO 访问 |
| 控制通道 | 设备内部 SRAM/寄存器 | 通过 hdc.ko 驱动访问 |

---

## 五、场景视图 (Scenarios View) — 关键用例

### 场景 1：单进程任务提交与完成（核心流程）

```mermaid
sequenceDiagram
    participant App as 用户进程
    participant UTRS as trs用户态库
    participant KTRS as trs内核驱动
    participant TSCPU as TSCPU

    App->>UTRS: 1. halSqCqAllocate(type=HW_SQ)
    UTRS->>KTRS: ioctl(TRS_SQCQ_GET)
    KTRS->>KTRS: 分配 sq_ctx / cq_ctx
    KTRS->>KTRS: trs_chan_init() 注册通道
    KTRS-->>UTRS: mmap地址 / 门铃地址
    UTRS-->>App: sq_info（包含head/tail/db地址）

    App->>App: 2. 准备任务数据
    App->>UTRS: 3. halStreamTaskFill(stream_id, task_info)
    UTRS->>UTRS: 写SQE到SQ环缓冲区
    
    App->>UTRS: 4. halSqTaskSend()
    alt UIO模式
        UTRS->>UTRS: 写Doorbell(tail寄存器)
        UTRS-->>TSCPU: 直接触发
    else KIO模式
        UTRS->>KTRS: ioctl发送
        KTRS->>TSCPU: mailbox通知
    end

    TSCPU->>TSCPU: 5. 读取SQE, 调度到AI Core
    TSCPU->>TSCPU: 6. 任务执行完成
    TSCPU->>TSCPU: 7. 写CQE到CQ
    TSCPU->>KTRS: 8. MSI-X中断
    KTRS->>KTRS: 9. 中断处理→CQ轮询
    KTRS-->>App: 10. 唤醒(epoll/信号)

    App->>UTRS: 11. halSqCqQuery() 读取CQE
    App->>KTRS: 12. halSqCqFree()
    KTRS->>KTRS: 回收SQ/CQ资源
```

### 场景 2：进程间资源共享 (shr_id)

```mermaid
sequenceDiagram
    participant P1 as 进程A (创建者)
    participant P2 as 进程B (共享者)
    participant KTRS as 内核驱动
    participant SHR as shr_id模块

    P1->>KTRS: ioctl(SHR_ID_CONFIG, {res_type, res_id})
    KTRS->>SHR: trs_shr_id_node_create()
    SHR->>SHR: 创建共享节点, ref=1
    KTRS-->>P1: OK

    P2->>KTRS: ioctl(SHR_ID_ATTACH, {res_type, res_id})
    KTRS->>SHR: trs_shr_id_node_get()
    SHR->>SHR: ref++, 获取物理ID映射
    KTRS->>KTRS: 映射共享SQ内存到P2地址空间
    KTRS-->>P2: 共享SQ地址/门铃地址

    P1->>P1: 写SQE到共享SQ
    P1->>KTRS: ioctl(SQ_SEND, shr_sq_id)
    KTRS->>TSCPU: 通知调度
    
    P2->>P2: 写SQE到共享SQ（相同的SQ）
    P2->>KTRS: ioctl(SQ_SEND, shr_sq_id)

    P1->>KTRS: ioctl(SHR_ID_DECONFIG)
    KTRS->>SHR: ref--
    P2->>KTRS: ioctl(SHR_ID_DETACH)
    KTRS->>SHR: ref-- → 0, 删除节点
```

### 场景 3：远程设备任务调度 (URMA)

```mermaid
sequenceDiagram
    participant Local as 本地节点
    participant URMA as URMA模块
    participant Remote as 远端NPU

    Local->>Local: halSqCqAllocate(flag=REMOTE_ID)
    Local->>URMA: trs_sqcq_urma_alloc()
    URMA->>Remote: RDMA: 请求远端分配SQ/CQ
    Remote-->>URMA: 返回远端SQ/CQ ID
    URMA-->>Local: 返回远端资源信息

    Local->>Local: trs_get_urma_tseg_info_by_va()
    Local->>URMA: 查询远端内存映射表
    URMA-->>Local: 返回TSEG信息

    Local->>Local: 准备任务数据
    Local->>URMA: trs_sq_task_send_urma()
    URMA->>Remote: RDMA: 写SQE到远端SQ
    URMA->>Remote: RDMA: Doorbell通知
    Remote->>Remote: 调度执行
    Remote-->>Local: RDMA: 写CQE到本地CQ
    Local->>Local: 轮询CQ获取完成状态
```

### 场景 4：异常处理与进程退出

```mermaid
flowchart TD
    A["进程退出 / fd关闭"] --> B{检查进程状态}
    B -->|"正常退出"| C["trs_handle_proc_release()"]
    B -->|"异常退出"| D["trs_abnormal_proc()"]
    
    C --> E["锁定 proc_ctx"]
    E --> F["遍历所有 TS Instance"]
    F --> G["释放所有资源ID<br/>trs_res_id_recycle()"]
    G --> H["解映射SQ/CQ内存<br/>trs_sqcq_reg_unmap()"]
    H --> I["关闭通道<br/>trs_chan_uninit()"]
    I --> J["解绑SMMU<br/>proc_unbind_smmu()"]
    J --> K["通知TSCPU释放<br/>trs_core_notice_ts()"]
    K --> L["从 proc_list 移除"]
    
    D --> M["TS异常检测<br/>(心跳超时)"]
    M --> N["tsmng_abnormal_proc_func()"]
    N --> O["设置 TS 状态为 ABNORMAL"]
    O --> P["回收所有该TS上的资源"]
    P --> Q["通知所有关联进程"]
    
    L --> R["kref_safe_put() 释放"]
    Q --> R
```

---

## 六、4+1 视图总结矩阵

| 视图 | 核心关注点 | TRS 模块对应 |
|------|-----------|-------------|
| **逻辑视图** | 系统功能分解、模块职责、接口 | 用户态 API 层 → 内核核心层 → 通道/ID池 → 设备 Agent，层次分明 |
| **进程视图** | 运行时交互、并发、同步、通信 | 独立进程 ctx、mutex/spinlock 保护、UIO/KIO 双模式、中断/workqueue 异步 |
| **开发视图** | 源码组织、模块依赖、构建关系 | `trs` + `trsbase` 双模块、`inc/` 统一头文件、`lba/shr_id/stars` 子目录解耦 |
| **物理视图** | 硬件部署、地址映射、中断路径 | Host DRAM/Device HBM 双端内存、PCIe BAR mmap、MSI-X 中断、URMA RDMA 跨节点 |
| **场景视图** | 关键用例：任务提交、资源共享、远程调度、异常恢复 | 覆盖正常任务生命周期 + 多进程协作 + 跨节点扩展 + 容错恢复 |

Completed: *绘制4+1视图* (2/2)

以上通过 **4+1 视图模型** 完整展示了 TRS 模块的设计：

1. **逻辑视图** — 分层功能分解：用户态 API → 内核核心 → SQ/CQ实现 → 通道/ID池 → 设备端
2. **进程视图** — 运行时并发交互：进程隔离、UIO/KIO 双模式、中断/workqueue 异步、共享/远程场景时序
3. **开发视图** — 源码组织结构与依赖关系：`trs` + `trsbase` 双模块、清晰的头文件/实现分离
4. **物理视图** — 部署与硬件映射：Host/Device 内存模型、PCIe BAR mmap、中断路径、RDMA 跨节点
5. **场景视图** — 关键用例：任务提交完成、进程间共享、远程调度、异常恢复