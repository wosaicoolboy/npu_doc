# `halResAddrMap` 端到端实现与原理分析

## 一、函数声明

定义在 ascend_hal_base.h:

```c
DLLEXPORT drvError_t halResAddrMap(
    unsigned int devId,
    struct res_addr_info *res_info,
    unsigned long *va,
    unsigned int *len);
```

**用途**: 将设备的 SoC 资源（如 notify 寄存器、cnt_notify 寄存器等）的物理地址映射到当前进程的用户态虚拟地址空间，使进程可以直接访问硬件资源寄存器。

---

## 二、端到端调用链路全景

```
┌──────────────────────────────────────────────────────────────────────┐
│ 调用者: trs_res_map_reg_init() (trs_res.c)                          │
│   ┌─ 构造 res_addr_info                                              │
│   │    .res_type = RES_ADDR_TYPE_STARS_NOTIFY_RECORD                 │
│   │    .res_id   = notify_id                                         │
│   │    .target_proc_type = PROCESS_CP1                               │
│   └─ 调用 halResAddrMap(dev_id, &res_info, &va, &len)               │
├──────────────────────────────────────────────────────────────────────┤
│ 第1层: 用户态 HAL (弱符号默认实现)                                   │
│   ascend_hal/trs/core/trs_res.c                                      │
│   → 弱符号 __attribute__((weak)) 返回 DRV_ERROR_NONE (空操作)       │
├──────────────────────────────────────────────────────────────────────┤
│ 第2层: 用户态 APM 实现 (覆盖弱符号)                                  │
│   ascend_hal/dpa/apm/apm_user_res_map.c                              │
│   → 转换参数 → ioctl → 解析结果                                      │
├──────────────────────────────────────────────────────────────────────┤
│ 第3层: 内核态 ioctl 入口                                              │
│   sdk_driver/dpa/apm/res_map/apm_res_map.c                           │
│   → apm_fops_res_map() → apm_res_addr_map()                         │
├──────────────────────────────────────────────────────────────────────┤
│ 第4层: 资源类型特定操作                                              │
│   sdk_driver/dpa/apm/res_map/apm_res_map_ops.c                       │
│   → 通过 res_ops[res_type] 分发到具体实现                            │
├──────────────────────────────────────────────────────────────────────┤
│ 第5层: 跨设备消息通信                                                │
│   sdk_driver/dpa/apm/res_map/proxy_adapt/host/apm_res_map_host.c     │
│   → 发送 APM_MSG_TYPE_MAP 到设备侧 TS CPU 处理                      │
├──────────────────────────────────────────────────────────────────────┤
│ 第6层: 设备侧处理 (EP)                                               │
│   → 通过 devdrv_common_msg_send 发送到设备                           │
│   → 设备侧获取资源物理地址 → ioremap → 返回 VA                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 三、关键数据结构

### 3.1 输入参数 `res_addr_info` (在 `hal_pkg/trs_pkg.h`)

```c
struct res_addr_info {
    unsigned int id;                    // 一般为 tsId
    processType_t target_proc_type;     // 目标进程类型 (PROCESS_CP1 等)
    enum res_addr_type res_type;        // 资源地址类型
    unsigned int res_id;                // 资源ID (如 notify_id)
    unsigned int flag;                  // 标志位
    unsigned int rudevid;               // 远端统一设备ID
    unsigned int rsv[2];
};
```

### 3.2 资源地址类型枚举

```c
enum res_addr_type {
    RES_ADDR_TYPE_STARS_NOTIFY_RECORD,      // STARS notify 记录寄存器
    RES_ADDR_TYPE_STARS_CNT_NOTIFY_RECORD,  // STARS cnt_notify 记录寄存器
    RES_ADDR_TYPE_STARS_RTSQ,               // STARS RTSQ
    RES_ADDR_TYPE_CCU_CKE,                  // CCU CKE
    RES_ADDR_TYPE_CCU_XN,                   // CCU XN
    RES_ADDR_TYPE_STARS_CNT_NOTIFY_BIT_WR,  // CNT_NOTIFY bit写
    RES_ADDR_TYPE_STARS_CNT_NOTIFY_BIT_ADD, // CNT_NOTIFY bit加
    RES_ADDR_TYPE_STARS_CNT_NOTIFY_BIT_CLR, // CNT_NOTIFY bit清
    RES_ADDR_TYPE_STARS_TOPIC_CQE,          // STARS Topic CQE
    RES_ADDR_TYPE_HCCP_URMA_JETTY,          // HCCP URMA Jetty
    RES_ADDR_TYPE_HCCP_URMA_JFC,            // HCCP URMA JFC
    RES_ADDR_TYPE_HCCP_URMA_DB,             // HCCP URMA DB
    RES_ADDR_TYPE_NDA_URMA_DB,              // NDA URMA DB
    RES_ADDR_TYPE_MAX
};
```

### 3.3 内核态扩展参数 `res_map_info_in` (在 ascend_hal_define.h)

```c
struct res_map_info_in {
    unsigned int id;
    processType_t target_proc_type;
    enum res_addr_type res_type;
    unsigned int res_id;
    unsigned int flag;
    unsigned int priv_len;
    void *priv;                     // 私有数据 (如 trs_res_map_priv)
    unsigned int rsv[8];
};
```

### 3.4 私有数据 (TRS 注册时使用)

```c
struct trs_res_map_priv {
    unsigned int flag;              // 含 TSDRV_FLAG_REMOTE_ID 等
    unsigned int local_devid;
    unsigned int remote_devid;
    unsigned int rsv[8];
};
```

### 3.5 内核态映射 ops 接口

```c
struct apm_res_map_ops {
    ka_module_t *owner;
    bool (*res_is_belong_to_proc)(...);     // 检查资源归属
    int (*get_res_addr)(...);               // 获取资源物理地址
    int (*get_res_addr_array)(...);         // 多页场景获取物理地址
    void (*put_res_addr_array)(...);        // 释放物理地址引用
    int (*update_res_info)(...);            // 更新资源信息
};
```

---

## 四、分步实现详解

### Step 1: TRS 发起调用 (`trs_res.c:trs_res_map_reg_init`)

```c
int trs_res_map_reg_init(uint32_t dev_id, uint32_t ts_id, uint32_t res_id,
                          drvIdType_t type, struct res_id_usr_info *id_info)
{
    struct res_addr_info res_info = {0};
    unsigned long notify_va;
    uint32_t notify_len;

    // 1. 填充资源信息
    res_info.id = ts_id;
    res_info.res_id = res_id;
    res_info.target_proc_type = PROCESS_CP1;   // CP1 进程
    res_info.res_type = (type == DRV_NOTIFY_ID) ?
        RES_ADDR_TYPE_STARS_NOTIFY_RECORD :     // notify → notify record
        RES_ADDR_TYPE_STARS_CNT_NOTIFY_RECORD;  // cnt_notify → cnt_notify record

    // 2. 调用 halResAddrMap
    ret = halResAddrMap(dev_id, &res_info, &notify_va, &notify_len);

    // 3. 记录映射结果
    id_info->res_addr = (uint64_t)notify_va;    // 映射后的用户态 VA
    id_info->res_len = notify_len;

    // 4. 注册资源地址 (mmap 到用户态)
    ret = trs_register_reg(dev_id, id_info->res_addr, id_info->res_len);
}
```

### Step 2: 用户态 APM 实现 (apm_user_res_map.c)

覆盖弱符号，将 `res_addr_info` 转换为 `res_map_info_in` 并通过 ioctl 下发：

```c
drvError_t halResAddrMap(unsigned int devId, struct res_addr_info *res_info,
                          unsigned long *va, unsigned int *len)
{
    struct res_map_info_in res_info_in = {0};
    struct res_map_info_out res_info_out = {0};

    // 1. 参数校验
    if ((res_info == NULL) || (va == NULL) || (len == NULL))
        return DRV_ERROR_PARA_ERROR;

    // 2. 转换参数 (V1 → V2 格式)
    apm_user_res_info_in_convert(res_info, &res_info_in);

    // 3. 执行映射
    ret = apm_user_res_addr_map(devId, &res_info_in, &res_info_out);
    if (ret == 0) {
        *va = res_info_out.va;     // 输出: 映射后的用户态 VA
        *len = res_info_out.len;   // 输出: 映射长度
    }
    return ret;
}
```

`apm_user_res_addr_map` 内部:

```c
static int apm_user_res_addr_map(unsigned int devId, struct res_map_info_in *res_info_in,
                                  struct res_map_info_out *res_info_out)
{
    struct apm_cmd_res_map para;
    // 1. 参数检查
    // 2. 填充 ioctl 参数
    para.devid = devId;
    para.res_info = *res_info_in;
    // 3. 发送 ioctl APM_RES_ADDR_MAP
    ret = apm_cmd_ioctl(APM_RES_ADDR_MAP, &para);
    // 4. 解析结果
    res_info_out->va = para.va;
    res_info_out->len = para.len;
}
```

### Step 3: ioctl 发送机制 (apm_ioctl.c)

```c
int apm_cmd_ioctl(unsigned long cmd, void *para)
{
    // 1. 打开 APM 字符设备 (/dev/apm)
    // 2. 调用 ioctl(apm_fd, cmd, para)
    // 3. 错误码转换 (errno → drvError_t)
    // 4. 返回结果
}
```

### Step 4: 内核态 ioctl 入口 (`apm_res_map.c:apm_fops_res_map`)

```c
static int apm_fops_res_map(u32 cmd, unsigned long arg)
{
    struct apm_cmd_res_map para;

    // 1. 从用户态拷贝参数
    ka_base_copy_from_user(&para, usr_arg, sizeof(para));

    // 2. 转换逻辑 devid → 统一 devid (udevid)
    uda_devid_to_udevid(para.devid, &para.devid);

    // 3. 初始化私有数据 (从用户态拷贝 priv)
    apm_init_res_map_info_priv(&para.res_info);

    // 4. 执行核心映射逻辑
    ret = apm_res_addr_map(para.devid, &para.res_info, &para.va, &para.len);

    // 5. 结果拷贝回用户态
    ka_base_copy_to_user(usr_arg, &para, sizeof(para));
}
```

### Step 5: 核心映射逻辑 (`apm_res_addr_map`)

```c
int apm_res_addr_map(u32 udevid, struct res_map_info_in *res_info, u64 *va, u32 *len)
{
    // 1. 参数校验
    apm_fops_res_info_check(res_info);

    // 2. 确定 master/slave 进程关系
    //    - 如果是 master 进程调用: 查询 slave_tgid
    //    - 如果是 slave 进程调用: 查询 master_tgid
    apm_res_map_get_map_tgid(udevid, res_info, &master_tgid, &slave_tgid);

    // 3. 检查资源归属权限
    apm_res_is_belong_to_proc(master_tgid, slave_tgid, udevid, res_info);

    // 4. 更新资源信息 (TRS 注册的 update_res_info)
    apm_update_res_info(udevid, res_info);

    // 5. 通过 ops 调用实际的映射函数
    //    → host 版本发送 APM_MSG_TYPE_MAP 消息到设备侧
    task_res_ops->res_map(&para);

    // 6. 记录映射节点 (用于进程退出时自动清理)
    apm_res_map_add_node(res_map_domain, tgid, &para);

    // 7. 输出 VA 和长度
    *va = para.va;
    *len = para.len;
}
```

### Step 6: Host 版映射实现 (apm_res_map_host.c)

```c
int apm_res_map_host_res_map(struct apm_res_map_info *para)
{
    return apm_res_map_host_res_map_unmap(APM_MSG_TYPE_MAP, para);
}

static int apm_res_map_host_res_map_unmap(enum apm_msg_type msg_type,
                                           struct apm_res_map_info *para)
{
    struct apm_msg_map_unmap msg;

    // 1. 填充消息头
    apm_msg_fill_header(&msg.header, msg_type);
    msg.para = *para;

    // 2. 拷贝私有数据
    if (msg_type == APM_MSG_TYPE_MAP && para->res_info.priv != NULL) {
        memcpy_s(msg.res_map_priv, ..., para->res_info.priv, para->res_info.priv_len);
    }

    // 3. 通过 PCIe mailbox 发送消息到设备侧 TS CPU
    ret = apm_msg_send(para->udevid, &msg.header, sizeof(msg));

    // 4. 设备侧处理完成后, 返回映射结果
    if (ret == 0 && (msg_type == APM_MSG_TYPE_MAP || ...)) {
        para->va = msg.para.va;     // 从设备侧得到的映射 VA
        para->pa = msg.para.pa;     // 物理地址
        para->len = msg.para.len;   // 映射长度
    }
}
```

### Step 7: 消息发送到设备侧 (apm_host_msg.c)

```c
int apm_msg_send(u32 udevid, struct apm_msg_header *header, u32 size)
{
    // 通过 devdrv_common_msg_send 发送到设备
    // 通信通道: DEVDRV_COMMON_MSG_SMMU
    // 底层使用 PCIe mailbox 机制
    ret = devdrv_common_msg_send(udevid, header, size, size,
                                  &real_out_len, DEVDRV_COMMON_MSG_SMMU);

    // 同步等待设备侧处理完成
    if (header->result != 0) {
        return header->result;   // 设备侧返回的错误码
    }
}
```

### Step 8: 设备侧接收和处理

设备侧通过 `apm_msg_recv` 接收消息，根据 `msg_type` 分发到对应处理函数。对于 `APM_MSG_TYPE_MAP`：

1. 解析 `res_map_info_in` 中的 `res_type` 和 `res_id`
2. 调用 `res_ops[res_type]->get_res_addr()` 获取资源的物理地址
3. 通过 `ioremap` 将物理地址映射到内核虚拟地址空间
4. 获取设备的 SSID (substream ID) 用于 SMMU 配置
5. 返回映射后的 VA 和长度

对于 `RES_ADDR_TYPE_STARS_NOTIFY_RECORD` 类型，TRS 注册了专属的 `apm_res_map_ops`:

```c
// trs_host_res_map.c
struct apm_res_map_ops g_res_map_ops = {
    .owner = KA_THIS_MODULE,
    .res_is_belong_to_proc = trs_host_res_is_belong_to_proc,  // 权限检查
    .update_res_info = trs_host_update_res_info,               // 更新远端 ID
};
```

### Step 9: 结果返回给调用者

```
设备侧处理完成
    → msg.va / msg.len 填充
    → devdrv_common_msg_send 返回
    → apm_res_map_host 读取 para->va, para->len
    → apm_res_addr_map 输出 *va, *len
    → apm_fops_res_map 拷贝到用户态
    → apm_cmd_ioctl 返回
    → apm_user_res_addr_map 解析结果
    → halResAddrMap 输出 *va, *len
    → TRS 的 trs_res_map_reg_init 记录映射地址
```

---

## 五、完整流程图

```
TRS: trs_res_map_reg_init()
  │
  ├─ 构造 res_addr_info {res_type, res_id, ...}
  │
  ├─ halResAddrMap(devId, &res_info, &va, &len)       ◄── 对外 API
  │     │
  │     ├─ [用户态 APM] apm_user_res_map.c
  │     │     │
  │     │     ├─ apm_user_res_info_in_convert()        参数转换
  │     │     ├─ apm_cmd_ioctl(APM_RES_ADDR_MAP)       ioctl
  │     │     │     │
  │     │     │     └─ [内核态] apm_fops_res_map()
  │     │     │           │
  │     │     │           ├─ copy_from_user             拷贝参数
  │     │     │           ├─ uda_devid_to_udevid()       devid转换
  │     │     │           ├─ apm_res_addr_map()          核心逻辑
  │     │     │           │     │
  │     │     │           │     ├─ apm_res_map_get_map_tgid()  确定master/slave
  │     │     │           │     ├─ apm_res_is_belong_to_proc() 权限检查
  │     │     │           │     │     └─ res_ops[type]->res_is_belong_to_proc()
  │     │     │           │     │           └─ [TRS] trs_host_res_is_belong_to_proc()
  │     │     │           │     │
  │     │     │           │     ├─ apm_update_res_info()        更新资源信息
  │     │     │           │     │     └─ res_ops[type]->update_res_info()
  │     │     │           │     │           └─ [TRS] trs_host_update_res_info()
  │     │     │           │     │
  │     │     │           │     ├─ task_res_ops->res_map()      实际映射
  │     │     │           │     │     │
  │     │     │           │     │     └─ [Host] apm_res_map_host_res_map()
  │     │     │           │     │           │
  │     │     │           │     │           ├─ 填充 APM_MSG_TYPE_MAP 消息
  │     │     │           │     │           ├─ apm_msg_send(udevid, msg)
  │     │     │           │     │           │     │
  │     │     │           │     │           │     └─ devdrv_common_msg_send()
  │     │     │           │     │           │           │  (PCIe Mailbox)
  │     │     │           │     │           │           ▼
  │     │     │           │     │           │     [设备侧 TS CPU]
  │     │     │           │     │           │       │
  │     │     │           │     │           │       ├─ apm_msg_recv()
  │     │     │           │     │           │       ├─ 解析 msg_type
  │     │     │           │     │           │       ├─ res_ops[type]->get_res_addr()
  │     │     │           │     │           │       │     → 获取物理地址
  │     │     │           │     │           │       ├─ ioremap() / mmap
  │     │     │           │     │           │       │     → 映射到内核空间
  │     │     │           │     │           │       ├─ 填充 va, len
  │     │     │           │     │           │       └─ 返回结果
  │     │     │           │     │           │
  │     │     │           │     │           └─ 读取返回的 va, len
  │     │     │           │     │
  │     │     │           │     └─ apm_res_map_add_node()     记录映射
  │     │     │           │
  │     │     │           └─ copy_to_user                     返回结果
  │     │     │
  │     │     └─ 返回 va, len
  │     │
  │     └─ 输出: *va = 映射后的用户态虚拟地址
  │             *len = 映射长度
  │
  └─ trs_register_reg(dev_id, va, len)         注册到进程地址空间
```

---

## 六、关键设计点总结

| 设计 | 说明 |
|------|------|
| **弱符号机制** | trs_res.c 提供弱符号默认实现（返回空），apm_user_res_map.c 覆盖为实际实现 |
| **双层参数** | V1 (`res_addr_info`) 为对外 API 参数，V2 (`res_map_info_in`) 为内核扩展参数 |
| **进程模型** | 支持 Master/Slave 进程模型，映射需先建立绑定关系 |
| **权限检查** | 通过 `res_is_belong_to_proc` 检查资源归属 |
| **跨设备通信** | Host 侧通过 `devdrv_common_msg_send` (PCIe mailbox) 发送到设备侧处理 |
| **插件式 ops** | `apm_res_map_ops` 按 `res_type` 注册不同类型资源的操作 |
| **TRS 接入** | TRS 为 notify/cnt_notify 类型注册了归属检查 (`trs_host_res_is_belong_to_proc`) 和更新 (`trs_host_update_res_info`) |
| **自动清理** | 通过 `apm_res_map_add_node` 记录映射，进程退出时自动 unmap |
| **物理资源保护** | 物理地址不直接暴露给用户态，通过内核态的 `ioremap` 安全映射 |

Completed: *输出端到端实现和原理* (5/5)

以上就是 `halResAddrMap` 的完整端到端实现分析。它本质上是一个**跨主机-设备的资源物理地址映射通道**，通过分层架构（用户态 HAL → ioctl → 内核态权限检查 → PCIe mailbox → 设备侧处理），将设备侧的硬件寄存器物理地址安全地映射到 Host 侧的用户态进程地址空间中。