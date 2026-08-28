# FAE 现场排查文档 — UGREEN DXP4800 NAS 存储空间"其他"2.5TB 异常占用排查指南

> Docker 容器可写层膨胀 + UGOS 存储分析器跨卷符号链接误算的综合故障分析

| 项目 | 信息 |
|:---|:---|
| 设备型号 | UGREEN DXP4800 |
| 存储池 | 存储池2 (M.2 SSD) |
| 文件系统 | ext4 |
| 文档版本 | v2.0 |
| 日期 | 2026-08-28 |

---

## 01 故障现象描述

### 1.1 设备存储架构

| 层级 | 名称 | 物理介质 | RAID | 容量 |
|:---|:---|:---|:---|:---|
| 存储池2 | 存储池2 | M.2 SSD（硬盘1） | Basic（单盘） | 938.4GB |
| 存储空间2 | 逻辑卷 | — | — | 922.2GB |
| 文件系统 | /volume2 | — | — | ext4 |

### 1.2 UI 显示的异常现象

UGOS 存储管理界面显示存储空间2（/volume2）空间使用情况如下：

| 项目 | 数值 | 说明 |
|:---|:---|:---|
| 总容量 | 922.2 GB | M.2 SSD，ext4 文件系统 |
| 已用 / 可用 | 922.2 GB / 0B | 100% 满，状态为"警告" |
| 共享文件夹 | 2.5 GB | docker(2.5GB) + 迅雷下载(8KB) |
| **其他** | **2.5 TB** | 远超物理容量，UI 异常 |

> **核心矛盾**："其他"分类显示 **2.5TB**，而存储空间2 的物理总容量仅 **922.2GB**。2.5TB 的数据不可能存在于 922GB 的卷上——这是 UGOS 存储分析器的**跨卷符号链接误算**导致的显示异常。

### 1.3 已排除项

| 排除项 | 排除依据 |
|:---|:---|
| Btrfs 快照 | 文件系统为 ext4，无快照机制 |
| ext4 lost+found | du 扫描显示仅 16KB，可忽略 |
| ext4 预留空间（5%） | 922GB × 5% ≈ 46GB，不足以解释 2.5TB |
| LUN 占用 | UI 显示 LUN = 0B |
| 已删除文件被进程占用 | df 与 du 差值不足以解释 2.5TB 差额 |

---

## 01.5 现场案例截图

以下两张截图为本案排查的核心现场证据。图 1 展示 du 扫描结果与 UGOS UI 显示的矛盾，图 2 通过 docker ps -as 定位到元凶容器。

### 图 1：du 扫描 /volume2 + UGOS 空间使用情况 — 发现矛盾

![图1：左侧 du 命令扫描 /volume2 目录输出，右侧 UGOS 空间使用情况弹窗显示其他 2.87TB](https://raw.githubusercontent.com/xushir/fae-docker-storage-troubleshoot/main/assets/case-01-du-ugos.png)

**图 1**：左侧 SSH 终端在 `/volume2#` 执行 `du -sh .[!.]* *`，扫描 /volume2 下所有目录大小。右侧 UGOS 弹窗显示"空间使用情况"。

#### du 扫描结果（终端左侧）

| 大小 | 目录 | 说明 |
|:---|:---|:---|
| **882G** | @docker | Docker 底层数据目录（镜像/容器可写层/卷） |
| 19G | @appstore | 应用商店数据 |
| 11G | @video | 视频服务缓存 |
| 5.1G | @thumbnail | 缩略图缓存 |
| 3.3G | @aiconsole | AI 控制台 |
| 2.6G | docker | Docker 共享文件夹（用户可见） |
| 746M | @exif | EXIF 元数据索引 |
| 15M | @search | 搜索索引 |
| 16K | lost+found | ext4 恢复目录 |
| 8.0K | 迅雷下载 | 迅雷下载目录（空） |

**du 合计：约 923GB**

#### UGOS 空间使用情况（界面右侧）

| 分类 | 占用 | 说明 |
|:---|:---|:---|
| 已用 / 总量 | 922.03GB / 2.91TB | 存储空间2（ext4） |
| 可用 | 1.84TB | — |
| 共享文件夹 | 2.50GB | docker(2.5GB) + 迅雷下载(0B) |
| LUN | 0B | 未使用 |
| **其他** | **2.87TB** | 远超 du 扫描结果（~923GB） |

> **核心矛盾**
> - **du 扫描 /volume2 合计 ~923GB**，但 UGOS 显示"其他" = **2.5TB**
> - "其他"(2.5TB) > /volume2 物理容量(922GB)，说明 UGOS 存储分析器跟随了 `@docker` 符号链接，将其他卷的 Docker 数据误算到 /volume2
> - **下一步**：需要用 `docker system df` 和 `docker ps -as` 进入 Docker 层面排查

### 图 2：docker ps -as — 定位元凶容器

![图2：终端 docker ps -as 命令输出，transmission 容器 2.56TB 被红框标注](https://raw.githubusercontent.com/xushir/fae-docker-storage-troubleshoot/main/assets/case-02-docker-ps.png)

**图 2**：SSH 终端在 `/volume2/@docker#` 执行 `docker ps -as`，列出所有容器（含已停止）及可写层大小。红框标注 transmission 容器 SIZE = **2.56TB**。

#### 容器列表（按可写层大小排序）

| 容器名 | 镜像 | 状态 | 可写层大小 | 创建时间 |
|:---|:---|:---|:---|:---|
| **transmission** | linuxserver/transmission:4.0.5 | Exited(255) | **2.56TB** | 21个月前 |
| moviepilot-v2 | jxxghp/moviepilot-v2:latest | Exited(255) | 817MB | 5周前 |
| qbittorrent | linuxserver/qbittorrent:4.6.7 | Up 34s | 22.8kB | 2个月前 |
| emby | linuxserver/emby:latest | Exited(0) | 1.62MB | 21个月前 |
| music-tag-web | ugreen/music_tag_web:v1 | Exited(0) | 1.88MB | 12个月前 |
| IYUUPlus | iyuucn/iyuuplus | Exited(137) | 431kB | 21个月前 |
| v2raya | mzz2017/v2raya | Exited(255) | 155B | 10个月前 |
| lucky | ugreen/lucky:v3 | Exited(2) | 690B | 4个月前 |
| firefox | ugreen/firefox:v2 | Exited(0) | 37.8kB | 12个月前 |
| metatube | metatube/metatube-server:latest | Exited(2) | 0B | 21个月前 |

> **定位结果**
> - **transmission 容器可写层 = 2.56TB**，与其余 9 个容器（均 < 1GB）形成数量级差距
> - 状态 `Exited(255)`，已停止 3 小时，可安全删除
> - 该容器 21 个月前创建，下载目录未映射到宿主机卷，所有下载数据堆积在 overlay2 可写层
> - 2.56TB 与图 1 中 UGOS"其他"2.87TB 基本吻合，确认 transmission 容器是"其他"异常占用的主要来源

> **两图证据链**
> **图 1**（du 扫描 /volume2 合计 ~923GB，但 UGOS"其他"显示 2.87TB → 发现跨卷误算）→ **图 2**（docker ps -as 定位 transmission 容器可写层 2.56TB → 确认根因）。两张截图完成从"发现异常"到"定位元凶"的完整闭环。

---

## 02 根因分析

### 根因结论（双重问题）

- **问题一（显示异常）**：UGOS 存储分析器在扫描 /volume2 时跟随了 `@docker` 符号链接/绑定挂载，将实际存储在其他卷（如 /volume1 HDD 阵列）上的 Docker overlay2 数据错误归算到 /volume2 的"其他"分类，导致显示 2.5TB > 922GB 物理容量的荒谬结果。
- **问题二（实际占用）**：Transmission Docker 容器（ID: 18f4b9f3f59b）的下载目录未映射到宿主机卷，2.56TB 下载数据全部写入容器可写层（overlay2 upperdir）。虽然物理上存储在另一卷，但确实需要清理。
- **问题三（volume2 确实满了）**：/volume2 真正被占满（922GB/922GB）的原因并非 2.5TB 的 Docker 数据，而是 Docker 镜像/容器元数据/系统服务等在 /volume2 上的实际占用，需进一步排查。

### 2.1 存储架构与数据流

```
┌─────────────────────┐     ┌─────────────────────────────────┐
│   物理存储层         │     │   UGOS 存储分析器               │
│                     │     │                                 │
│  ┌───────────────┐  │     │  ┌─────────────┐                │
│  │ M.2 SSD       │  │     │  │ 扫描         │                │
│  │ 938.4GB       │  │     │  │ /volume2     │                │
│  │ 存储池2       │  │     │  └──────┬──────┘                │
│  │ → /volume2    │  │     │         │                       │
│  │ ext4          │  │     │    ┌────┴────┐                   │
│  └───────────────┘  │     │    │         │                   │
│                     │     │    ▼         ▼                   │
│  ┌───────────────┐  │     │  ┌──────┐  ┌────────────────┐    │
│  │ HDD 阵列      │  │     │  │跟随   │  │错误归算        │    │
│  │ 存储池1       │  │     │  │符号   │  │"其他"=2.5TB    │    │
│  │ → /volume1    │  │     │  │链接   │  │(跨卷误算)      │    │
│  │ (更大容量)    │  │     │  └──┬───┘  └────────────────┘    │
│  └───────────────┘  │     │     │                            │
└─────────────────────┘     └─────┼────────────────────────────┘
                                  │
┌─────────────────────────────────┼─────────────────────────────┐
│   Docker 数据层                  ▼                             │
│  ┌──────────────────────────────────────┐                      │
│  │ /volume2/@docker (符号链接/绑定挂载)  │                      │
│  └──────────────┬───────────────────────┘                      │
│                 │ 符号链接指向                                    │
│                 ▼                                                │
│  ┌──────────────────────────────────────┐                      │
│  │ Docker Root Dir                      │                      │
│  │ /volume1/@docker                     │                      │
│  │ (实际数据存储位置)                   │                      │
│  └──────┬──────────────────┬────────────┘                      │
│         │                  │                                    │
│         ▼                  ▼                                    │
│  ┌──────────────┐  ┌────────────────┐                          │
│  │ transmission │  │ Docker 镜像    │                          │
│  │ 容器可写层    │  │ /元数据 ~19GB │                          │
│  │ 2.56TB       │  └────────────────┘                          │
│  │ overlay2     │                                                │
│  │ upperdir     │                                                │
│  └──────────────┘                                                │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 Docker overlay2 可写层机制

Docker 使用 overlay2 存储驱动，每个容器的文件系统由两部分组成：

| 层 | 路径 | 说明 |
|:---|:---|:---|
| 镜像层（lowerdir） | 只读，多容器共享 | 镜像本身内容 |
| 可写层（upperdir） | `<docker-root>/overlay2/<hash>/diff/` | 容器运行时写入的所有数据，删除容器即释放 |

当容器内写入数据但未使用 volume mount 时，数据堆积在可写层。容器停止后数据不丢失，但无法通过 UGOS UI 识别。

### 2.3 本案容器数据明细

| 容器名 | 镜像 | 状态 | 可写层大小 | 创建时间 |
|:---|:---|:---|:---|:---|
| transmission | linuxserver/transmission:4.0.5 | Exited(255) | **2.56TB** | 21个月前 |
| moviepilot-v2 | jxxghp/moviepilot-v2 | Exited(255) | 817MB | 5周前 |
| qbittorrent | linuxserver/qbittorrent:4.6.7 | Up | 22.8kB | 2个月前 |
| emby | linuxserver/emby | Exited(0) | 1.62MB | 21个月前 |
| 其他6个容器 | — | 已停止 | <5MB 合计 | — |

### 2.4 时间线还原

| 时间 | 事件 | 影响 |
|:---|:---|:---|
| ~21个月前 | 创建 transmission 容器，下载目录未映射到宿主机 | 数据开始堆积在可写层 |
| ~21个月内 | transmission 持续下载，可写层膨胀至 2.56TB | Docker 数据占据大量空间 |
| ~2个月前 | 改用 qbittorrent，transmission 停用但未删除 | 2.56TB 数据残留 |
| 近期 | /volume2 空间耗尽，UGOS 报警告 | UI 显示"其他 2.5TB"引发用户疑问 |
| 排查时 | docker system df 确认 Containers SIZE = 2.558TB | 定位根因 |

---

## 03 排查决策流程

以下流程图展示从发现 UI 异常到定位根因的完整决策路径：

```
UGOS UI 显示存储空间2
'其他' = 2.5TB
'共享文件夹' = 2.5GB
'总容量' = 922GB
        │
        ▼
   ┌─────────────┐
   │ 2.5TB > 922GB? │
   │ 物理矛盾？     │
   └─────┬───────┘
    是   │    否
   ┌────┘    └────┐
   ▼              ▼
判定：UGOS       常规排查：
跨卷误算          du 扫描 + df 对比
@docker 为
符号链接
   │
   ▼
Step 1: 确认 Docker Root Dir
docker info | grep 'Docker Root Dir'
        │
        ▼
   ┌─────────────┐
   │ Root Dir 在   │
   │ /volume2?    │
   │ 还是其他卷？  │
   └─────┬───────┘
  其他卷  │  /volume2 本地
   ┌────┘    └────┐
   ▼              ▼
确认跨卷符号链接   针对性排查对应目录
ls -ld /volume2/@docker
        │
        ▼
Step 2: du 扫描顶层目录
        │
        ▼
   ┌─────────────┐
   │ @docker      │
   │ 目录最大？   │
   └─────┬───────┘
    是   │    否
   ┌────┘    └────┐
   ▼              ▼
Step 3:          检查 Images/Volumes
docker system df docker system prune
分解 Docker 空间
        │
        ▼
   ┌─────────────────┐
   │ Containers SIZE  │
   │ 异常大？(TB级)   │
   └─────┬───────────┘
    是   │    否
   ┌────┘    └────┐
   ▼              ▼
Step 4:          检查 Images/Volumes
docker ps -as    docker system prune
定位具体容器
        │
        ▼
找到超大容器
transmission: 2.56TB
        │
        ▼
Step 5: 验证数据
是否需要保留
        │
   ┌────┴────┬──────────┐
   ▼         ▼          ▼
方案A:     方案B:      方案C:
docker rm  docker cp   truncate
立即释放   备份后删除   清理日志
   │         │          │
   └────┬────┴──────────┘
        ▼
清理后验证
+ 配置卷映射/日志轮转
```

---

## 04 详细排查步骤

### Step 0：判断 UI 显示是否存在跨卷误算

**目的**：首先判断"其他"显示值是否超过物理容量，决定后续排查方向。

```bash
# 对比 UI 显示的"其他"与存储空间总容量
# 如果 其他(2.5TB) > 总容量(922GB) → 跨卷误算，进入 Step 1
# 如果 其他 < 总容量 → 常规排查，跳到 Step 2
```

> **关键判断点**：如果"其他"超过物理总容量，说明 UGOS 存储分析器跟随了符号链接/绑定挂载，将其他卷的数据误算到当前卷。此时需要先确认 Docker 实际数据根目录位置。

### Step 1：确认 Docker 数据实际存储位置

**目的**：确认 Docker Root Dir 是否在 /volume2 上，还是通过符号链接指向其他卷。

```bash
# 查看 Docker 实际的 data-root
docker info | grep "Docker Root Dir"

# 查看 /volume2/@docker 是否为符号链接
ls -ld /volume2/@docker

# 查看各卷的实际占用
df -h
```

> **预期输出**：如果 `Docker Root Dir` 显示为 `/volume1/@docker` 或其他非 /volume2 路径，而 `/volume2/@docker` 是一个软链接（`lrwxrwxrwx ... -> /volume1/@docker`），则证实跨卷误算。

### Step 2：确认 df 与 du 的差值

**目的**：对比文件系统级别与目录级别的已用空间，判断是否存在"已删除但被进程占用的文件"。

```bash
# 文件系统级别已用空间
df -h /volume2

# 目录级别累加空间
du -sh /volume2

# 扫描所有顶层目录（含隐藏目录）
cd /volume2
du -sh -- * .[!.]*/ 2>/dev/null | sort -hr | head -30
```

> **判断标准**：如果 `df` 显示已用远大于 `du` 算出的总和（差值 > 500GB），说明有已删除文件仍被进程占用，执行 `lsof +L1 /volume2` 查找。如果差值不大，说明空间确实被某个目录占用。

### Step 3：检查 Docker 空间使用分解

**目的**：将 Docker 空间分解为 Images / Containers / Volumes / Build Cache 四类。

```bash
# Docker 磁盘使用分解
docker system df
```

典型输出：

```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          30        10        18.84GB   10.02GB (53%)
Containers      10        1         2.558TB   2.558TB (99%)
Local Volumes   3         2         598.3MB   0B (0%)
Build Cache     0         0         0B        0B
```

> **关键判据**：**Containers** 的 SIZE 列出现 TB 级数值且 RECLAIMABLE 接近 100%，即可判定：**容器可写层膨胀**。正常容器可写层应为 MB 级别。

### Step 4：定位具体超大容器

**目的**：从所有容器中找出可写层最大的那一个。

```bash
# 列出所有容器（含已停止），显示可写层大小
docker ps -as
```

典型输出：

```
CONTAINER ID   IMAGE                                  NAMES            SIZE
7a942731dd36   jxxghp/moviepilot-v2:latest            moviepilot-v2    817MB (virtual 2.5GB)
40e6ad4e4444   linuxserver/qbittorrent:4.6.7           qbittorrent       22.8kB (virtual 194MB)
18f4b9f3f59b   linuxserver/transmission:4.0.5         transmission     2.56TB (virtual 2.56TB)
d08ae62f2970   metatube/metatube-server:latest         metatube          0B (virtual 40.6MB)
```

> **定位结果**：容器 `transmission`（ID: 18f4b9f3f59b）可写层为 **2.56TB**，状态 `Exited(255)`，已停用。该容器 21 个月前创建，下载目录未映射到宿主机，所有下载文件堆积在可写层。

### Step 5：获取容器 ID 并验证容器内数据

**目的**：从 Step 4 的输出中获取超大容器的 CONTAINER ID，在删除前确认数据是否需要保留。

#### 5.1 获取容器 ID

Step 4 中 `docker ps -as` 输出的第一列即为 CONTAINER ID。也可以用以下命令精确提取：

```bash
# 方法一：按容器名获取完整 ID
docker inspect --format='{{.Id}}' transmission
# 输出示例：18f4b9f3f59b...（完整 64 位 ID）

# 方法二：按容器名获取短 ID（前 12 位，通常够用）
docker inspect --format='{{.Id}}' transmission | cut -c1-12
# 输出示例：18f4b9f3f59b

# 方法三：用 docker ps 过滤出目标容器的 ID
docker ps -a --filter "name=transmission" --format "{{.ID}}"
# 输出示例：18f4b9f3f59b

# 方法四：列出所有容器名与 ID 的对应关系
docker ps -a --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Size}}" | grep -i trans
```

> **说明**
> - **完整 ID**（64 位）：`docker inspect` 返回的 `.Id` 字段，用于 `--volumes-from` 等需要精确引用的场景
> - **短 ID**（前 12 位）：Docker 命令接受短 ID，日常操作更方便
> - 本案中 transmission 容器的短 ID 为 `18f4b9f3f59b`，完整 ID 可通过方法一获取

#### 5.2 验证容器内数据

拿到容器 ID 后，使用 `--volumes-from` 临时启动一个 busybox 容器来查看目标容器内的文件大小：

```bash
# 查看容器内各目录大小（以 /downloads 为例，实际路径可能不同）
docker run --rm --volumes-from 18f4b9f3f59b busybox du -sh /downloads 2>/dev/null

# 查看根目录下所有目录大小
docker run --rm --volumes-from 18f4b9f3f59b busybox du -sh /* 2>/dev/null

# 如果容器还在运行，也可以直接 exec 进去查看
docker exec transmission du -sh /downloads 2>/dev/null
docker exec transmission du -sh /* 2>/dev/null
```

> **判断逻辑**
> - 如果 `/downloads` 显示 TB 级数据 → 说明是下载文件堆积，需决定是否备份
> - 如果各目录都很小但容器 SIZE 仍显示 TB 级 → 可能是 Docker 元数据/日志膨胀，无需备份业务数据
> - 已改用 qbittorrent 且 transmission 数据不需要 → 可直接跳到 Step 6 删除

### Step 6：确认容器卷映射配置（诊断为何数据写入可写层）

**目的**：查看 transmission 容器的卷映射配置，确认下载目录是否未映射到宿主机。

```bash
# 查看 transmission 容器的挂载配置
docker inspect transmission | grep -A 30 '"Mounts"'

# 或者查看完整配置（重点看 Mounts 和 HostConfig.Binds 字段）
docker inspect transmission

# 查看容器启动命令中的 -v 参数（如果用 docker run 创建）
docker inspect transmission | grep -A 5 '"Binds"'
```

> **预期发现**：如果 `Mounts` 字段为空或不含 `/downloads` 的映射，说明下载目录未挂载到宿主机——这就是 2.56TB 数据堆积在可写层的根因。正确配置应该包含类似 `/volume1/共享文件夹/downloads:/downloads` 的映射。

### Step 7：停止并删除 transmission 容器

**目的**：删除已停止的 transmission 容器，释放其可写层占用的 2.56TB 空间。

```bash
# 1. 确认容器已停止（如果还在运行先停止）
docker stop transmission

# 2. 删除容器（释放可写层全部空间）
docker rm transmission

# 3. 确认容器已删除
docker ps -a | grep transmission
```

> **预期效果**：执行 `docker rm transmission` 后，overlay2 可写层立即释放 **2.56TB** 空间。最后一条 grep 命令应无输出，确认容器已彻底删除。

### Step 8：验证空间释放

**目的**：确认删除容器后空间已释放，UGOS UI 恢复正常。

```bash
# 1. 再次查看 Docker 空间分解，确认 Containers SIZE 已下降
docker system df

# 2. 查看 Docker Root Dir 所在卷的空间使用情况
df -h

# 3. 查看 /volume2 的空间使用情况
df -h /volume2

# 4. 重新扫描 /volume2 顶层目录
cd /volume2
du -sh -- * .[!.]*/ 2>/dev/null | sort -hr | head -10
```

> **验证标准**
> - `docker system df` 中 Containers SIZE 应从 2.558TB 降至 MB 级别
> - Docker Root Dir 所在卷的 `df -h` 可用空间应增加约 2.5TB
> - UGOS UI"其他"分类应从 2.87TB 降至正常水平（约 900GB 系统目录占用）
> - /volume2 存储空间状态从"警告"恢复为"正常"

### Step 9：清理其他停止的容器和悬空镜像（可选）

**目的**：顺带清理其他 9 个已停止的容器和可回收的镜像，进一步释放空间。

```bash
# 1. 查看当前可回收空间
docker system df

# 2. 清理所有已停止的容器（qbittorrent 在运行中，不会被删除）
docker container prune

# 3. 清理悬空镜像（dangling images）
docker image prune

# 4. 清理未使用的镜像（谨慎：会删除所有未被运行容器引用的镜像）
docker image prune -a

# 5. 最终确认 Docker 空间状态
docker system df
```

> **注意**：`docker image prune -a` 会删除所有未被运行中容器引用的镜像。如果后续还需要用 moviepilot、emby 等已停止的容器，不要执行此命令，只需执行 `docker container prune` 清理停止的容器即可。

### Step 10：验证 UGOS UI 恢复正常

**目的**：在 UGOS 管理界面确认"其他"分类已恢复正常。

```
# 在 UGOS Web 界面操作：
# 1. 打开 存储管理 → 存储空间
# 2. 点击存储空间2 的"空间使用情况"
# 3. 检查"其他"分类是否从 2.87TB 降至约 900GB
# 4. 检查存储空间状态是否从"警告"变为"正常"
# 5. 检查进度条是否从满红变为蓝色正常区间
```

> **排查完成标准**
> - UGOS UI"其他"分类降至约 900GB（@docker 镜像/元数据 + @appstore + @video 等系统目录正常占用）
> - 存储空间2 状态从"警告"恢复为"正常"
> - `docker system df` Containers SIZE 降至 MB 级别
> - Docker Root Dir 所在卷可用空间增加约 2.5TB

---

## 05 解决方案

### 方案 A：直接删除不需要的容器

适用场景：容器内数据确认无需保留。

```bash
# 删除指定容器（释放该容器可写层全部空间）
docker rm transmission

# 或一次性清理所有已停止的容器
docker container prune
```

> **预期效果**：删除 transmission 容器后释放 **2.56TB** 空间（在数据实际所在的卷上），UGOS UI"其他"分类恢复正常。

### 方案 B：备份数据后删除

适用场景：容器内数据需要保留。

```bash
# 1. 创建备份目录
mkdir -p /volume1/<共享文件夹>/transmission_backup

# 2. 将容器内数据拷贝到宿主机
docker cp 18f4b9f3f59b:/downloads /volume1/<共享文件夹>/transmission_backup/

# 3. 确认文件完整后删除容器
docker rm transmission
```

> **注意**：`docker cp` 拷贝 2.5TB 数据可能需要数小时，建议在非高峰时段执行。拷贝完成后务必校验文件完整性再删除容器。备份目标应选在有足够空间的卷上。

### 方案 C：清理容器日志膨胀

适用场景：可写层膨胀原因是日志文件而非业务数据。

```bash
# 查找超过 1GB 的容器日志文件
find /volume2/@docker/containers -name "*.log" -size +1G -exec ls -lh {} \;

# 清空指定容器的日志（不删除容器）
truncate -s 0 /volume2/@docker/containers/<CONTAINER_ID>/*.log
```

### 方案 D：清理无用的 Docker 镜像

```bash
# 详细列表
docker system df -v

# 清理所有未使用的镜像、停止的容器、未使用的网络
docker system prune -a
```

> **警告**：`docker system prune -a` 会删除**所有未被运行中容器引用的镜像**。执行前务必确认停止状态的容器后续是否还需要。

### 方案 E：排查 /volume2 本身被占满的原因

如果清理 Docker 后 /volume2 仍然显示 100% 满，需要排查 volume2 本身的占用：

```bash
# 查看 ext4 预留空间
tune2fs -l /dev/sdXX | grep "Reserved block"

# 调低预留比例（大容量盘建议 1%）
tune2fs -m 1 /dev/sdXX

# 查找已删除但仍被占用的文件
lsof +L1 /volume2
```

---

## 06 预防措施

### 6.1 正确配置卷映射

创建容器时，务必将数据写入路径映射到宿主机目录：

```bash
# 检查现有容器的卷映射
docker inspect <CONTAINER_NAME> | grep -A 20 "Mounts"

# docker-compose.yml 正确配置
services:
  transmission:
    image: linuxserver/transmission:4.0.5
    volumes:
      - /volume1/共享文件夹/downloads:/downloads   # 映射到宿主机
      - /volume1/@docker/transmission/config:/config
    logging:
      driver: json-file
      options:
        max-size: "200m"
        max-file: "3"
```

### 6.2 定期巡检

| 巡检项 | 命令 | 建议频率 |
|:---|:---|:---|
| Docker 空间概览 | `docker system df` | 每月一次 |
| 清理停止的容器 | `docker container prune` | 每月一次 |
| 验证卷映射 | `docker inspect <container> \| grep Mounts` | 新建容器时 |
| 检查大文件 | `du -sh /* \| sort -hr \| head -20` | 每季度一次 |
| 确认 Docker Root Dir | `docker info \| grep "Docker Root Dir"` | 系统变更后 |

---

## 07 附录

### 7.1 本案关键数据汇总

| 项目 | 数值 | 说明 |
|:---|:---|:---|
| 存储空间2 总容量 | 922.2GB | M.2 SSD，ext4 |
| 已用空间 | 922.2GB (100%) | 状态：警告 |
| UI 显示"其他" | 2.5TB | 超过物理容量→跨卷误算 |
| 共享文件夹合计 | 2.5GB | docker(2.5GB)+迅雷(8KB) |
| docker system df Containers | 2.558TB (99%可回收) | 根因：容器可写层 |
| transmission 容器可写层 | 2.56TB | 下载目录未映射 |
| 容器创建时间 | 21个月前 | 长期堆积未清理 |

### 7.2 命令速查表

| 命令 | 用途 | 步骤 |
|:---|:---|:---|
| `docker info \| grep "Docker Root Dir"` | 确认 Docker 数据实际存储位置 | Step 1 |
| `ls -ld /volume2/@docker` | 检查是否为符号链接 | Step 1 |
| `df -h /volume2` | 文件系统级已用空间 | Step 2 |
| `du -sh /volume2` | 目录级累加空间 | Step 2 |
| `du -sh -- * .[!.]*/ \| sort -hr` | 扫描所有顶层目录 | Step 2 |
| `docker system df` | Docker 空间分解 | Step 3 |
| `docker system df -v` | Docker 空间详细列表 | Step 3 |
| `docker ps -as` | 列出所有容器及大小 | Step 4 |
| `docker rm <name>` | 删除指定容器 | 方案 A |
| `docker container prune` | 清理所有停止的容器 | 方案 A |
| `docker cp <id>:/path /host/path` | 从容器拷贝数据 | 方案 B |
| `docker inspect <name> \| grep Mounts` | 检查卷映射 | 预防 |
| `lsof +L1 /volume2` | 查找已删除但被占用的文件 | 方案 E |
| `tune2fs -l /dev/sdX \| grep Reserved` | 查看 ext4 预留空间 | 方案 E |
| `tune2fs -m 1 /dev/sdX` | 调整 ext4 预留为 1% | 方案 E |

### 7.3 UGOS 存储分析器"其他"分类说明

| 分类 | 包含内容 | 是否可从 UI 清理 |
|:---|:---|:---|
| 共享文件夹 | 用户创建的共享文件夹数据 | 是（文件管理器） |
| LUN | iSCSI 块存储 | 是（LUN 管理） |
| 其他 | @ 开头的系统目录、Docker 数据、回收站、系统缓存、符号链接目标等 | 需 SSH 排查 |

### 7.4 已知 UGOS UI 限制

> **UGOS 存储分析器限制**
> - 扫描 /volume2 时会跟随符号链接/绑定挂载，导致跨卷数据被误算为当前卷的"其他"
> - 不深入 Docker overlay2 层级，无法区分镜像层与容器可写层
> - "其他"分类过大时（超过物理容量），可判定为符号链接误算而非真实占用
> - 此问题在 Docker Root Dir 配置在其他卷但 /volume2 下存在 @docker 链接时触发
