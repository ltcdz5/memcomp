# memcomp 移植到 Linux 6.1.157（OPPO/一加 SM8650，ColorOS 17）

状态：**P1–P3 已完成，代码能编、能开机、符号 CRC 全绿；P4 卡在"开 memcomp 必崩"**
最后更新：2026-11-24（真机崩溃定位中）

## 0. 一句话结论

能移植。上游是 6.12.23 的 RFC，和 6.1 差的不是几个函数名，而是三块基础设施：
`folio->swap`、mTHP、reclaim 路径结构。按"单页 + PMD THP"砍下来改动可控，
但 `shrink_folio_list` 那一处必须手写，不能靠 fuzz。

**现状**：编译层和 ABI 层已经全部打通，卡在运行时——开 memcomp 后几十秒到两分钟
必崩（无 Oops、成片用户态 SIGSEGV）。详见 §5。

## 1. 上游是什么

| 项 | 事实 |
|---|---|
| 仓库 | `nakixii/memcomp`（论文/基准）、`nakixii/linux` 分支 `memcomp/series-v1` |
| 基线 | Linux 6.12.23 + 8 个前置 commit |
| 形态 | 16 个 patch，`mm/memcomp.c` 6181 行单片 + `mm/memcomp/*.h` 36 个头 |
| 作者自述 | "proof of concept…not meant for merging" |
| 硬约束 | `depends on MMU && 64BIT && CRYPTO && SWAP && ZRAM=n && !ZSWAP` |
| 已知短板 | 有真 swap 时单页重缺页 p99 是 zram 的 4.8 倍；无 swap 时优于 zram |

## 2. 我们的底座差什么

实测环境：`gt5pro157-base/common`（6.1.157 vendor 树，Clang 17）。

| 6.12 依赖 | 6.1.157 现状 |
|---|---|
| `folio->swap` 具名成员 | 没有。只有 `folio->private`（`void *`） |
| mTHP（`PMD_ORDER`、`thp_vma_suitable_orders`、`deferred_split_folio`） | 没有。只有经典 PMD THP |
| `set_ptes()` 批量装 PTE | 没有（6.5 才有） |
| `swap_pte_batch()` | 没有 |
| `__pte_offset_map_lock()` | 没有，6.1 是 `pte_offset_map_lock(mm, pmd, addr, ptlp)` |
| `update_mmu_cache_range()` | 没有，6.1 是 `update_mmu_cache(vma, addr, pte)` |
| `folio_add_new_anon_rmap(folio, vma, addr, flags)` | 没有，6.1 是 `page_add_new_anon_rmap(page, vma, addr)` |
| `shrinker_alloc()` 动态分配 shrinker | 没有，6.1 是静态 `struct shrinker` + `register_shrinker()` |
| `get_obj_cgroup_from_folio()` | 没有，6.1 是 `get_obj_cgroup_from_page()` |
| `arch_do_swap_page_nr()` | 没有，6.1 是 `arch_do_swap_page(mm, vma, addr, pte, pteval)` |
| `gfp_has_io_fs()` | 没有，手写 `(gfp & (__GFP_IO \| __GFP_FS))` 判断 |
| `CRYPTO_ACOMP` Kconfig | 6.1 叫 `CRYPTO_ACOMP2` |
| 用户态异步压缩（`crypto_alloc_acomp`） | **6.1 根本没有 zstd 的 acomp 驱动；连 `crypto_alloc_scomp()` 都没有** |

## 3. 编译层实测（已完成）

做法：独立树 `/home/builder/kwork/mc/common`，只把 memcomp 自有文件 +
`mm/Kconfig`/`mm/Makefile` 落进去，`make mm/memcomp.o` 让编译器报错。

- 首轮 20 条就撞 `-ferror-limit`；放开限制后 **108 条 error**
- 归并后约 **30 类**，其中约 10 类只属于 mTHP 路径 → 砍掉即消失
- 最终落地：`mm/memcomp.c` 6181 → 6215 行，相对上游只改 55 行；
  36 个头文件只改 88 行

## 4. 集成层实测（已完成）

| 文件 | 改动量 |
|---|---|
| `mm/vmscan.c` | **+393 / −5，18 个 hunk**（全部手写，非 fuzz） |
| `mm/swapfile.c` | +54 |
| `mm/shmem.c` | +126 |
| `mm/memory.c` / `mm/mincore.c` / `mm/rmap.c` 等 | 少量 |
| 新增 `mm/memcomp/compat_6_1.h` | 147 行适配层 |

### 4.1 两个必须知道的 KABI 陷阱

1. **`struct swap_info_struct` 加一个成员，会让 `__i2c_transfer` 这种完全无关的
   导出符号 CRC 都变**（同一 TU include 集合逐字节相同也变）。改法：不动结构体，
   改用按 type 索引的独立表 `swap_memcomp_maps[MAX_SWAPFILES]`（`mm/swapfile.c`）。
2. **`mm/internal.h` 里 `#include <linux/swap.h>` 会让 `list_lru` / `slab_common`
   的 CRC 变**。改法：把 memcomp 的 inline helper 整块搬到 `include/linux/memcomp.h`。

### 4.2 swap 类型编码

`SWP_MEMCOMP` 占一个特殊类型，`MAX_SWAPFILES` 相应减一：

```c
#define SWP_MEMCOMP_NUM 1
#define SWP_MEMCOMP (MAX_SWAPFILES + SWP_HWPOISON_NUM + \
		     SWP_MIGRATION_NUM + SWP_DEVICE_NUM)
#define MAX_SWAPFILES ((1 << MAX_SWAPFILES_SHIFT) - SWP_DEVICE_NUM - \
		       SWP_MIGRATION_NUM - SWP_HWPOISON_NUM - SWP_MEMCOMP_NUM - \
		       SWP_PTE_MARKER_NUM - SWP_SWAPIN_ERROR_NUM)
```

同时 `SWP_SWAPIN_ERROR` 的公式必须补上 `SWP_MEMCOMP_NUM`，否则两个特殊类型撞号。
（已产出 `patches/6.1/0002-6.1-swap-entry-type.patch`，对 6.1 干净。）

### 4.3 ABI 验收（已通过）

- `out-mc2/vmlinux.symvers` md5 `3d51e64204788f3a81d585c7da9f3ff9`
- 与 `gt5pro157-base/common/out/vmlinux.symvers` 对比：**CRC 差异 0**，
  只多出 10 个独有的 `memcomp_*` 符号
- 三套闸门：vendor_boot 490 / vendor_dlkm 439 / system_dlkm 60，
  **CRC mismatch 全 0**；28 个 "TRUE missing" 是既有的 HMBIRD/LMGR 家族，不算问题

### 4.4 移植过程中修掉的四个真 bug

1. `mm/vmscan.c`：`goto memcomp_try` 前必须补 `if (!(sc->gfp_mask & __GFP_IO)) goto keep_locked;`
   ——压缩会睡眠，`GFP_NOIO` 回收不能进
2. `mm/memory.c`：6.1 的 zap 循环是单 PTE 步进，`nr = memcomp_pte_batch(...)` 必须改成 `nr = 1`
3. `mm/memcomp.c`：新增 `memcomp.disabled=` 总开关（用于 A/B 对照）
4. `mm/memcomp.c`：新增 `memcomp.shmem=` 开关（默认关；后来证明 shmem 不是崩因，可保留可撤）

## 5. 运行时：开 memcomp 必崩（未解决）

### 5.1 现场数据

```
MemFree:          157772 kB
MemAvailable:    6986492 kB
nr_inactive_anon 641406      nr_active_anon 682331
pgscan_anon 63840            pgsteal_anon 1
pgscan_file 842487           pgsteal_file 767581
--- memcomp ---
14544896 3927082 4612096     ← 逻辑量 / 压缩后 / 池占用
```

特征：

- **无 BUG、无 Oops、无 RCU stall、无 D 状态任务**
- 看门狗复位；复位前 `crash_dump` 抢 CPU 第一
- 用户态**成片 SIGSEGV**
- 死亡时间不稳定：39s / 66s / 93s / 148s
- **第一次崩（39s）时根本没压测**，只读了几条命令 → 不是压测假象，是"随机几十秒必死"
- 同一颗内核加 `memcomp.disabled=1` 跑压测活过 161s、WiFi 正常
  → **问题确实在 memcomp 启用路径**

`stat` 三个数：逻辑存储字节 / 压缩后字节 / 分配器池字节。
**63840 页只压进去 14.5MB**，说明绝大多数 store 失败，但失败原因没有暴露。

### 5.2 已排除的假说

| 假说 | 结论 |
|---|---|
| shmem 路径写坏数据 | **否**。关掉 shmem 照样死 |
| `memcomp_pte_batch` 批量 zap | **否**。改成 `nr = 1` 照样死 |
| acomp 缺失导致退化 | **不是崩因**。退化只影响速度，见 §6 |
| ZRAM/ZSWAP 共存冲突 | **否**。两边 config 都是 n |
| 符号 CRC 漂移导致厂商模块异常 | **否**。三套闸门 mismatch 全 0 |
| `vmlinux.symvers` 被动过 | **否**。md5 稳定 |
| `mm/memcomp.c.rej` 那条 include | **假报警**。源码 4369 行已有该 include |

### 5.3 头号嫌疑：6.1 的大 folio 路径

`shrink_folio_list` 里，`memcomp_reserve(folio)` 之后还会走到这段 6.1 特有代码：

```c
if (folio_test_large(folio)) {
        if (!can_split_folio(folio, NULL))
                goto activate_locked;
        if (!folio_entire_mapcount(folio) &&
            split_folio_to_list(folio, folio_list))
                goto activate_locked;
        if (nr_pages > 1 && !folio_test_large(folio)) {
                sc->nr_scanned -= nr_pages - 1;
                nr_pages = 1;
        }
}
```

上游 0009（大 folio 批量 unmap，151 行）和 0010（大 folio 拆分排队，6 行）被整块跳过。

如果 memcomp 按 `nr_pages` 预留了 N 页 span，而 folio 随后被拆成单页、`nr_pages`
没同步，回收时就会按错的页数释放/写回 → **用户页变垃圾 → 成片 SIGSEGV**。
这与现场"无 Oops + 用户态大规模崩"的形态吻合。

待回答：
1. `memcomp_reserve()` 之后到 `anon_backing_ready` 之间，有没有路径改变
   `folio_nr_pages()` 或拆 folio？
2. `memcomp_reclaim_ctx_get(nr_pages, ...)` 页数对不上时，store 会不会写出 span？
3. `split_folio_to_list()` 之后原来的 `memcomp_ctx` 还有效吗？
4. 若改成"reserve 之前就拆"或直接 `activate_locked`，代价是什么？

## 6. 关于异步压缩（上游 0008）——第一版不做

0008 那 727 行异步批量压缩依赖 `crypto_alloc_acomp(..., CRYPTO_ALG_ASYNC, ...)`，
也就是需要一个**异步压缩器**。这棵树里的事实是：

- `crypto/zstd.c` 只注册 scomp（同步，`zstd-scomp`），全树唯一的 acomp 提供者是
  `drivers/crypto/hisilicon/zip/`（海思硬件，与本平台无关）
- `CONFIG_CRYPTO_ACOMP2=y` 只是框架，没有算法实现

结果就是 `memcomp_async_get()` 返回 NULL，整条异步路径直接退化回同步压缩，
等于白移植。所以第一版不做它是正确的判断，不是砍功能。

另外，6.1 连 `crypto_alloc_scomp()` 都没有
（`include/crypto/internal/scompress.h` 只有 `crypto_register_scomp()`）。
将来若确实要做异步，只能走 `crypto_alloc_comp("zstd", 0, 0)` + 私有 workqueue，
每个 batch 独占一个 tfm（comp 的 tfm ctx 不可重入）。

## 7. 下一步（P4 收尾）

1. **加 debugfs 埋点**：`mm/memcomp.c` 现在没有任何 debugfs/procfs 入口，
   而 `memcomp_count_anon_fail()` 那四个计数器（UNMAP_FAIL / FINISH_FAIL /
   INCOMPRESSIBLE_SKIP / normal-swap-fallback）已经存在但看不到。
   先把它们吐出来，否则后面全是猜。
2. 出 `Image` → `mc_accept.sh`：符号 CRC diff 必须 0、三套闸门 mismatch 必须 0、
   `vmlinux.symvers` md5 仍是 `3d51e642...`
3. 打 37.5MB 截断版 → `fastboot boot`
4. 开机第一件事 `echo never > /sys/kernel/mm/transparent_hugepage/enabled`，
   然后**放着不动**，看能否活过 39 / 66 / 93 / 148 秒四档
5. 定性：
   - 关 THP 就稳 → 锁定大 folio，按 6.1 补 0009/0010 或改成 reserve 前先拆
   - 照样崩 → 读计数器决定下一步
6. 成功后再做 P5 压测（后台留存、重缺页延迟、与 pulse1 的 zram 对照），
   不达标整条线判废

## 8. 风险清单

| 风险 | 说明 | 处理 |
|---|---|---|
| **必须关 zram** | `ZRAM=n` 是上游硬约束；本机原厂用 zram0（20GB）当 swap，关掉后 Android 的 swap 全由 memcomp 承担 | 先按约束做；"zram 只当回写盘"不在第一版 |
| vmscan 结构差异 | 6.1 `shrink_folio_list` 和 6.12 不同 | 手写，不 fuzz（已完成，见 §4） |
| `folio->private` 语义 | 6.1 的 `private` 被 swapcache 占用，memcomp 也用它存 token/哈希 | 只在非 swapcache folio 上用；冲突路径加 WARN |
| 大 folio | 没有 mTHP，只能整 PMD 处理 | **当前崩因头号嫌疑，见 §5.3** |
| LMKD 行为变化 | 没有 zram 后内存压力表现不同 | P5 实测，不达标就判废 |
| 厂商模块 | 关 ZRAM 后 `zram.ko` 没了，vendor 侧可能有模块依赖 | 提前查 `system_dlkm` 依赖 |

## 9. 纪律

- 独立树、独立分支，**不碰 pulse 主线**
- 只动 `boot_b`，先 `fastboot boot`，验收不过不写盘
- 每个阶段产物单独存：`Image.mc-Pn`、`vmlinux.symvers-mc-Pn`、`config.mc-Pn`
- 回退镜像固定：`work/backup-20261123/boot_b.img`（原厂 157，md5 `0f6dc971...`）
- 注意：`out-mc2/.config` 会被 `make` 的 syncconfig 自动改写
  （`CONFIG_CC_IS_GCC` → clang），原因是改过 `mm/Kconfig` 导致 mtime 更新；
  这不是跑了 `olddefconfig`，但下次 make 还会再同步一次

## 10. 这项工作的定位

上游目前只面向 6.12 基线。**6.1 GKI 这一档没有公开实现**，
而 Android 14/15/16/17 的厂商内核都在这一档。

所以这是一条独立维护的分叉：不为"能被上游 review"去凑形状，
以"在 SM8650 + ColorOS 17 上稳定"为唯一验收标准。
