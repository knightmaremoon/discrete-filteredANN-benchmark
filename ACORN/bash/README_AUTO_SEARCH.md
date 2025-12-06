# ACORN自动参数搜索 - Arxiv数据集

## 🎯 功能

这个脚本实现了**一体化的参数搜索**：
1. ✅ 自动构建所有参数组合的索引
2. ✅ 自动测试所有场景的搜索性能
3. ✅ 自动找出最优参数
4. ✅ 断点续传（可随时中断和恢复）
5. ✅ 详细的进度跟踪和结果记录

## 🚀 快速开始

### 1. 检查数据文件

确保以下文件存在（路径相对于ACORN目录）：

```
data/arxiv/
├── arxiv_base.fvecs              # 基础向量数据
├── label_base.txt                # 基础数据标签
├── arxiv_query_equal.fvecs       # Equal查询向量
├── arxiv_query_equal.txt         # Equal查询标签
├── arxiv_gt_equal.txt            # Equal ground truth
├── arxiv_query_or.fvecs          # OR查询向量
├── arxiv_query_or.txt            # OR查询标签
├── arxiv_gt_or.txt               # OR ground truth
├── arxiv_query_and.fvecs         # AND查询向量
├── arxiv_query_and.txt           # AND查询标签
└── arxiv_gt_and.txt              # AND ground truth
```

⚠️ **重要**: 如果你的数据路径不同，请修改脚本中的 `DATA_DIR` 变量！

### 2. 确保编译了ACORN

```bash
cd ACORN
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j
cd ..
```

### 3. 运行参数搜索

```bash
cd ACORN/bash
python auto_param_search_arxiv.py
```

## 📊 输出文件

```
data/param_search_arxiv/
├── summary.csv           # ← 主要结果文件（Excel可打开）
├── progress.json         # 进度文件（用于断点续传）
├── indices/              # 索引文件目录
│   └── arxiv/
│       ├── M=12_Mb=16_gamma=1/
│       ├── M=16_Mb=24_gamma=2/
│       └── ...
└── results/              # 详细搜索结果
    └── arxiv/
        ├── equal/
        ├── or/
        └── and/
```

### summary.csv 格式

| M | M_beta | gamma | scenario | build_time_s | search_time_ms | recall@1 | recall@10 | recall@100 | index_size_mb | status |
|---|--------|-------|----------|--------------|----------------|----------|-----------|------------|---------------|--------|
| 16 | 32 | 1 | equal | 45.2 | 12.5 | 0.85 | 0.95 | 0.99 | 120.5 | success |
| 16 | 32 | 1 | or | 45.2 | 15.3 | 0.82 | 0.93 | 0.98 | 120.5 | success |

## ⚙️ 配置参数

在脚本顶部可以修改：

```python
# 参数范围
Ms = list(range(12, 49, 4))      # [12, 16, 20, ..., 48]
M_betas = list(range(12, 65, 4))  # [12, 16, 20, ..., 64]
gammas = [1, 2, 4]                # gamma值

# 数据路径（根据实际情况修改）
DATA_DIR = "../data/arxiv"
```

### 快速测试（小范围参数）

```python
# 修改为：
Ms = [16, 24, 32]
M_betas = [32, 48, 64]
gammas = [1, 2]
```

这样只测试 3×3×2 = 18个组合，每个3个场景 = 54个任务，约30分钟完成。

## 🔄 断点续传

如果运行中断（Ctrl+C或崩溃），直接重新运行即可：

```bash
python auto_param_search_arxiv.py
```

脚本会自动：
- ✅ 读取 `progress.json`
- ✅ 跳过已完成的任务
- ✅ 从中断处继续

## 📈 查看结果

### 方法1: 脚本自动分析（需要pandas）

```bash
pip install pandas
python auto_param_search_arxiv.py  # 结束时会自动显示最优参数
```

### 方法2: Excel查看

```bash
# 在Excel中打开
open data/param_search_arxiv/summary.csv
```

按 `recall@10` 降序排序，找到最优参数。

### 方法3: Python分析

```python
import pandas as pd

df = pd.read_csv('data/param_search_arxiv/summary.csv')
df = df[df['status'] == 'success']

# 查看equal场景的最优参数
equal_df = df[df['scenario'] == 'equal']
best = equal_df.nlargest(5, 'recall@10')
print(best[['M', 'M_beta', 'gamma', 'recall@10', 'search_time_ms']])
```

## 🎯 预期运行时间

基于默认配置（~70个参数组合 × 3个场景）：

| 阶段 | 时间估算 |
|------|----------|
| 构建索引 | ~70组合 × 2分钟 = 2.5小时 |
| 搜索测试 | ~210任务 × 1分钟 = 3.5小时 |
| **总计** | **约6小时** |

💡 建议在云端后台运行：

```bash
# 使用nohup后台运行
nohup python auto_param_search_arxiv.py > search.log 2>&1 &

# 查看实时日志
tail -f search.log

# 查看进度
cat data/param_search_arxiv/progress.json
```

## ❓ 常见问题

### Q1: 数据文件路径不对怎么办？

修改脚本中的 `DATA_DIR` 变量：

```python
# 如果数据在 /home/user/datasets/arxiv/
DATA_DIR = "/home/user/datasets/arxiv"
```

### Q2: 想只测试某个场景怎么办？

修改 `SCENARIOS`：

```python
# 只测试equal
SCENARIOS = ["equal"]

# 或只测试or和and
SCENARIOS = ["or", "and"]
```

### Q3: 磁盘空间不够怎么办？

索引文件会占用较大空间。可以：

**选项1**: 测试完一个参数就删除索引

```python
# 在search_index函数后添加：
if cleanup_index:
    index_path = f"{INDEX_DIR}/{DATASET}/hybrid_M={M}_Mb={M_beta}_gamma={gamma}.json"
    os.remove(index_path)
```

**选项2**: 减少参数范围（快速测试）

### Q4: 如何只重新测试搜索性能？

如果索引已经构建好，只想重新测试搜索：

```python
# 删除进度文件中的搜索任务
# 但保留构建任务
# 然后重新运行
```

### Q5: 构建或搜索失败怎么办？

检查日志文件：

```bash
# 构建日志
cat data/param_search_arxiv/indices/arxiv/M=16_Mb=32_gamma=1/build.log

# 搜索日志
cat data/param_search_arxiv/results/arxiv/equal/M=16_Mb=32_gamma=1/search.log
```

## 🔧 与原始两阶段流程对比

### 原始流程（手动）

```bash
# 步骤1: 构建所有索引
python traverse_param_space.py

# 步骤2: 测试所有场景
python search_in_subspace.py

# 步骤3: 汇总结果
python combine_search_result.py

# 步骤4: 手动分析找最优参数
```

### 新流程（自动）

```bash
# 一步完成
python auto_param_search_arxiv.py

# 自动输出最优参数
```

## 📝 高级用法

### 自定义参数约束

```python
# 在脚本中修改约束条件
for M in Ms:
    for M_beta in M_betas:
        # 原约束
        if M_beta < M:
            continue

        for gamma in gammas:
            # 原约束
            if M_beta > 2 * M * gamma:
                continue

            # 添加自定义约束
            if M < 20 and gamma > 2:  # 例如：M小于20时不测gamma>2
                continue
```

### 并行运行多个参数

```bash
# 分成3个任务并行（需要手动分配参数范围）

# 终端1: 测试 M=12-24
# 修改脚本: Ms = list(range(12, 29, 4))
python auto_param_search_arxiv.py

# 终端2: 测试 M=28-40
# 修改脚本: Ms = list(range(28, 45, 4))
python auto_param_search_arxiv.py

# 终端3: 测试 M=44-48
# 修改脚本: Ms = list(range(44, 49, 4))
python auto_param_search_arxiv.py
```

## 📧 问题反馈

如果遇到问题，请提供：
1. 错误日志（search.log）
2. 数据文件列表（ls -la data/arxiv/）
3. 进度文件内容（cat data/param_search_arxiv/progress.json）
