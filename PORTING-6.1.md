# memcomp 移植到 Linux 6.1.157（OPPO/一加 SM8650，ColorOS 17）

状态：**可行性已实测，未开始写代码**
最后更新：2026-11-24

## 0. 一句话结论

能做，但不能"打补丁"式照搬。上游是 6.12.23 的 RFC，
和 6.1 之间差的不是几个函数名，而是三块基础设施：
`folio->swap`、mTHP、reclaim 路径结构。
按下面的方案砍到"单页 + PMD THP"范围，改动量可控，
但 vmscan 那一处必须手写，不能靠 fuzz。

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
| `folio->swap` 具名成员（swap cache 从 page->private 迁到 folio） | 没有。只有 `folio->private`（`void *`） |
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

## 3. 实测数据（不是估计）

做法：独立树 `/home/builder/kwork/mc/common`（与原树同深度，符号链接才不折），
只把 memcomp 自有文件 + `mm/Kconfig`/`mm/Makefile` 落进去，
`make mm/memcomp.o` 让编译器报错。

- 首轮 20 条就撞 `-ferror-limit`；放开限制后 **108 条 error**
- 归并后约 **30 类**，分布：
  - 约 10 类**只属于 mTHP 路径** → 砍掉即消失
  - 约 6 类 6.12 重构，有干净的 6.1 等价物（上表）
  - 3 类是真设计问题：`folio->swap`（16 处）、memcg per-object id、shrinker API
  - 其余是 `mm/internal.h` 的声明 hunk 没打（我故意没打，用于隔离）

也就是说：**语法层失败集中在少数几类，不是散弹**。这是能做的信号。

## 4. 移植范围（第一版砍到最小）

保留：
- anon 页回收 → zsmalloc 压缩 → token PTE → 缺页解压回映射
- 单页（order 0）+ PMD THP（order 9）
- memcg 记账（否则内存账目会歪）
- 零页（same-filled）快路径

第一版**不做**：
- shmem/tmpfs（上游 patch 0006）
- 冷页回写 swap（0007）
- 异步批量压缩（0008）
- mTHP 批量 unmap / 拆分（0009、0010）
- KUnit、sysfs 精确统计（0011、0012 只取必要部分）
- userfaultfd 两条（0001、0004）——先不接 UFFD，风险最低

## 5. 分阶段计划

### P0 环境（已完成）
- 独立树 `/home/builder/kwork/mc/common`，不改 pulse 主线
- 上游最终源码来自 `nakixii/linux:memcomp/series-v1`（sparse clone），
  不逐个叠 patch，避免 patch 顺序依赖

### P1 编译骨架（预估 0.5 天）
1. `0002` swap 类型：`SWP_MEMCOMP`（已产出 `patches/6.1/0002-6.1-swap-entry-type.patch`，对 6.1 干净）
2. `mm/internal.h` 的声明 hunk 手抄
3. `folio->swap` → 加 shim：
   ```c
   static inline swp_entry_t folio_swap_entry(struct folio *f);
   static inline void folio_set_swap_entry(struct folio *f, swp_entry_t e);
   ```
   底层落到 `folio->private`（与 6.1 swapcache 用 private 存 swp_entry 的语义一致）
4. 砍 mTHP：`fault.h`/`zero_load.h`/`folio.h` 里的 order>9 分支去掉，
   `deferred_split_*` 整块删，`thp_vma_*_orders` 换成 `pmd_order` 常量
5. `set_ptes` 循环化、`__pte_offset_map_lock` → `pte_offset_map_lock`、
   `update_mmu_cache_range` → `update_mmu_cache`、`folio_add_new_anon_rmap` →
   `page_add_new_anon_rmap`、`shrinker` 静态化、`get_obj_cgroup_from_page`
6. Kconfig：`CRYPTO_ACOMP` → `CRYPTO_ACOMP2`
验收：`make mm/memcomp.o` rc=0

### P2 集成点（预估 1–2 天，最重）
按文件逐个手改，顺序按依赖：
1. `mm/memory.c`：`copy_nonpresent_pte`（fork）、`zap_pte_range`、`do_swap_page`
2. `mm/vmscan.c`：`shrink_folio_list` 里插压缩回收分支（上游 +311 行，6.1 结构不同，必须重写）
3. `mm/rmap.c`：token PTE 的 rmap 处理
4. `mm/migrate.c`、`mm/swapfile.c`、`mm/madvise.c`、`fs/proc/task_mmu.c`
验收：整核 `make Image` rc=0

### P3 KABI 闸门（预估 0.5 天）
- 三套闸门（vendor_boot 490 / vendor_dlkm 439 / system_dlkm 60）
- 重点盯 `swap_*`、`folio_*`、`shrink_*` 相关导出有没有 CRC 漂移
- 有漂移就退回，改成不改签名的写法

### P4 RAM 引导（预估 1 天，含反复）
- `CONFIG_MEMCOMP=y` + `CONFIG_ZRAM=n` + `CONFIG_ZSWAP=n`
- 先 `fastboot boot`，验收：开机、`/sys/kernel/mm/memcomp` 出现、
  有压缩量、无 oops、网络正常、`lsmod` 数量不掉
- 只有全过才写 `boot_b`

### P5 压测（3–5 天）
- 后台留存、重缺页延迟、内存占用、温度
- 和 pulse1（zram）同场景对照
- 不达标就整条线判废，不留半成品在主线

## 6. 风险清单

| 风险 | 说明 | 处理 |
|---|---|---|
| **必须关 zram** | `ZRAM=n` 是上游硬约束；本机现在 zram0 10GB 就是 swap。关了之后 Android 的 swap 全由 memcomp 承担 | 先按约束做；能不能放开（zram 只当回写盘）需要单独验证，不在第一版 |
| vmscan 结构差异 | 6.1 `shrink_folio_list` 和 6.12 不同，311 行插不进去 | 手写，不 fuzz |
| `folio->private` 语义 | 6.1 的 `private` 被 swapcache 占用，memcomp 也用它存 token/哈希 | 只在非 swapcache folio 上用；冲突路径加 WARN 兜底 |
| THP 退化 | 没有 mTHP，只能整 PMD 处理 | 第一版接受；order>9 直接回落到普通 swap/zram 路径 |
| LMKD 行为变化 | 没有 zram 后内存压力表现不同 | P5 实测，不达标就判废 |
| 厂商模块 | 关 ZRAM 后 `zram.ko` 没了，vendor 侧可能有模块依赖它 | 提前查 `system_dlkm` 依赖 |

## 7. 纪律

- 独立树、独立分支，**不碰 pulse 主线**
- 只动 `boot_b`，先 `fastboot boot`，验收不过不写盘
- 每个阶段产物单独存：`Image.mc-Pn`、`vmlinux.symvers-mc-Pn`、`config.mc-Pn`
- 回退镜像固定：`work/backup-20261123/boot_b.img`（原厂 157）

## 8. 下一步

等确认两件事再动 P1：
1. 第一版是否就按"关 zram、memcomp 独占"来做
2. 是否接受"不支持 16K/64K mTHP，只做单页 + PMD"
