# 毕设_制热_源数据 — 数据处理流程

## 项目背景

热泵制热/制冷仿真数据处理。原始数据由仿真软件导出，每个 CSV 对应一个充注量工况，文件内包含多时间步的完整状态参数。项目包含两类工况：
- **制热工况**（热泵模式）：环境温度范围 -30℃ ~ 5℃
- **制冷工况**（制冷模式）：环境温度范围 25℃ ~ 50℃

## 目录结构

```
├── 不同充注量/              # 制热工况（早期数据）
│   ├── M-20/               # -20℃环境温度（参考组）
│   └── M5/                 # 5℃环境温度
│
├── 不同充注量(热泵)/         # 制热工况（完整数据集）
│   ├── M-30a/              # -30℃环境温度
│   ├── M-20b/              # -20℃环境温度（参考组）
│   ├── M-10a/              # -10℃环境温度
│   ├── M0a/                # 0℃环境温度
│   └── M5a/                # 5℃环境温度
│
└── 不同充注量（制冷）/        # 制冷工况
    ├── M25c/               # 25℃环境温度
    ├── M35a/               # 35℃环境温度
    ├── M45a/               # 45℃环境温度
    ├── I45/                # 45℃室内温度
    └── I50a/               # 50℃室内温度
```

每个工况文件夹内结构：
- `heating_*.csv` / `cooling_*.csv` — 原始仿真数据（1-15 对应不同充注量）
- `*不同充注量.xlsx` — 整理后的汇总表（处理后生成）

## 数据格式说明

### 原始仿真 CSV 特点
- Windows 换行符（`\r\n`）
- 分隔符为 tab + 空格混合（对齐用空格填充）
- 列头第一列为 `#`（行号占位符，无实际数据对应），导致全部列名右移一位
- 行尾有尾随空格，pandas 解析时会产生额外空列

### 31 个参数列（去除 `#` 后，原始 CSV 中的列名）

原始仿真导出的 CSV 列头使用制热工况命名，31 列如下：

```
time[s], P_suc[bar], T_suc[degC], P_dis[bar], T_dis[degC],
P_gc_mid[bar], T_gc_mid[degC], P_gc_out[bar], T_gc_out[degC],
P_eva_in[bar], T_eva_in[degC], P_eva_out[bar], T_eva_out[degC],
W_comp[kW], Q_heat_s1[kW], Q_heat_s2[kW], Q_heat_total[kW],
COP[null], m_dot[kg/s], T_air_in[degC], T_mid[degC],
T_air_out[degC], V_air_indoor[kg/min], T_amb[degC],
V_air_outdoor[m/s], h_suc[kJ/kg], h_dis[kJ/kg],
h_gc_mid[kJ/kg], h_gc_out[kJ/kg], h_eva_in[kJ/kg],
h_eva_out[kJ/kg]
```

**制热与制冷的区别**：原始 CSV 的数据列顺序完全相同，但制热/制冷模式下各列的物理含义不同：

| 位置 | 制热模式含义 | 制冷模式含义 |
|------|-------------|-------------|
| 5-6 | 气体冷却器中间压力/温度 | **蒸发器**中间压力/温度 |
| 7-8 | 气体冷却器出口压力/温度 | **蒸发器**出口压力/温度 |
| 9-10 | 蒸发器进口压力/温度 | **气冷**进口压力/温度 |
| 11-12 | 蒸发器出口压力/温度 | **气冷**出口压力/温度 |
| 14-16 | 一/二级加热量、总加热量 | 一/二级**制冷量**、**制冷量** |
| 27-28 | 气冷中间焓、气冷出口焓 | **蒸发**中间焓、**蒸发**出口焓 |
| 29-30 | 蒸发进口焓、蒸发出口焓 | **气冷**进口焓、**气冷**出口焓 |

## 处理流程（三步走）

### 第一步：分列（原始 tab+空格 → 规范 CSV）

```python
import pandas as pd
import glob

# 按 tab + 空格混合分隔读取，并修正列名偏移
correct_names = [
    'time[s]', 'P_suc[bar]', 'T_suc[degC]', 'P_dis[bar]', 'T_dis[degC]',
    'P_gc_mid[bar]', 'T_gc_mid[degC]', 'P_gc_out[bar]', 'T_gc_out[degC]',
    'P_eva_in[bar]', 'T_eva_in[degC]', 'P_eva_out[bar]', 'T_eva_out[degC]',
    'W_comp[kW]', 'Q_heat_s1[kW]', 'Q_heat_s2[kW]', 'Q_heat_total[kW]',
    'COP[null]', 'm_dot[kg/s]', 'T_air_in[degC]', 'T_mid[degC]',
    'T_air_out[degC]', 'V_air_indoor[kg/min]', 'T_amb[degC]',
    'V_air_outdoor[m/s]', 'h_suc[kJ/kg]', 'h_dis[kJ/kg]',
    'h_gc_mid[kJ/kg]', 'h_gc_out[kJ/kg]', 'h_eva_in[kJ/kg]',
    'h_eva_out[kJ/kg]'
]

for f in sorted(glob.glob('heating_*.csv')):
    df = pd.read_csv(f, sep=r'\s+', engine='python')
    # 修正：去掉 # 列（col 0），去掉尾随空列（最后1列全NaN），保留31列
    df = df.iloc[:, :31]
    df.columns = correct_names
    df.to_csv(f, index=False, encoding='utf-8-sig')
```

关键点：
- `sep=r'\s+'` 匹配任意空白字符（tab + 空格）
- 原始列头 `#` 是占位符，数据从 `time[s]` 开始，需丢弃首列并保留数据列 0-30
- 尾随空格导致最后一列全 NaN，需丢弃
- 输出使用 `utf-8-sig` 编码（BOM），确保 Excel 直接打开中文不乱码

### 第二步：汇总提取（每个 CSV 取最后一个采样点）

每个仿真文件的最后一行即为稳态工况的最终数据。

```python
import pandas as pd
import glob

# 充注量映射（文件编号 → 克数）
charge_map = {
    1: 200, 2: 250, 3: 300, 4: 350, 5: 400,
    6: 450, 7: 500, 8: 550, 9: 600, 10: 650,
    11: 700, 12: 750, 13: 800, 14: 850, 15: 900
}

rows = []
for f in sorted(glob.glob('heating_*.csv'), key=lambda x: int(x.replace('heating_','').replace('.csv',''))):
    file_num = int(f.replace('heating_', '').replace('.csv', ''))
    charge = charge_map[file_num]
    df = pd.read_csv(f, encoding='utf-8-sig')
    last = df.iloc[-1]  # 最后一个采样点
    rows.append({'充注量(g)': charge, **last.to_dict()})
```

### 第三步：导出汇总 xlsx

```python
from openpyxl import Workbook

# 参数列中文名（与 M-20 参考格式一致）
param_names_cn = [
    '吸气压力(bar)', '吸气温度(℃)', '排气压力(bar)', '排气温度(℃)',
    '气体冷却器中间压力(bar)', '气体冷却器中间温度(℃)',
    '气体冷却器出口压力(bar)', '气体冷却器出口温度(℃)',
    '蒸发器进口压力(bar)', '蒸发器进口温度(℃)',
    '蒸发器出口压力(bar)', '蒸发器出口温度(℃)',
    '压缩机功率(kW)', '一级加热量(kW)', '二级加热量(kW)', '总加热量(kW)',
    'COP', '质量流量(kg/s)', '进口空气温度(℃)', '中间空气温度(℃)',
    '出口空气温度(℃)', '室内风量(kg/min)', '环境温度(℃)', '室外风速(m/s)',
    '吸气焓(kJ/kg)', '排气焓(kJ/kg)', '气体冷却器中间焓(kJ/kg)',
    '气体冷却器出口焓(kJ/kg)', '蒸发器进口焓(kJ/kg)', '蒸发器出口焓(kJ/kg)'
]

# 注意：汇总表去掉 time[s] 列，保留其他 30 个参数
param_cols = correct_names[1:]  # 跳过 time[s]

wb = Workbook()
ws = wb.active
# 表头
ws.cell(row=1, column=1, value='充注量(g)')
for ci, name in enumerate(param_names_cn):
    ws.cell(row=1, column=ci+2, value=name)

# 数据
for ri, row in enumerate(rows):
    ws.cell(row=ri+2, column=1, value=row['充注量(g)'])
    for ci, col in enumerate(param_cols):
        ws.cell(row=ri+2, column=ci+2, value=row[col])

wb.save('M5不同充注量.xlsx')
```

## 注意事项

1. **环境依赖**：需要 pandas、openpyxl。当前环境需用 `pip install --proxy="" openpyxl` 安装（校园网代理问题）
2. **编码**：始终使用 `utf-8-sig` 读写，确保中文在 Excel 中正常显示
3. **列数验证**：处理后每个 CSV 应为 651 行 × 31 列，汇总表为 15 行 × 31 列（1个充注量 + 30个参数）
4. **充注量映射**：根据实际情况调整 `charge_map` 字典
5. **汇总表去掉 time[s]**：参照 M-20 参考格式，汇总表中不含时间列
6. **最后一个采样点**：使用 `df.iloc[-1]` 而非 `df.tail(1)`，前者返回 Series 便于取值
7. **文件命名**：制热工况用 `heating_*.csv`，制冷工况用 `cooling_*.csv`
8. **制冷工况列名**：处理 CSV 时使用制冷英文名，导出 xlsx 时使用制冷中文名

---

## 制冷工况处理（附录）

制冷工况与制热工况的处理流程完全一致，唯一的区别是参数列名不同。

### 制冷工况 31 个英文列名

```python
cooling_names = [
    'time[s]',
    'P_suc[bar]', 'T_suc[degC]', 'P_dis[bar]', 'T_dis[degC]',
    'P_eva_mid[bar]', 'T_eva_mid[degC]',   # 蒸发中间（原气冷中间位置）
    'P_eva_out[bar]', 'T_eva_out[degC]',     # 蒸发出口（原气冷出口位置）
    'P_gc_in[bar]', 'T_gc_in[degC]',         # 气冷进口（原蒸发进口位置）
    'P_gc_out[bar]', 'T_gc_out[degC]',       # 气冷出口（原蒸发出口位置）
    'W_comp[kW]',
    'Q_cool_s1[kW]', 'Q_cool_s2[kW]', 'Q_cool_total[kW]',  # 制冷量
    'COP[null]', 'm_dot[kg/s]',
    'T_air_in[degC]', 'T_mid[degC]', 'T_air_out[degC]',
    'V_air_indoor[kg/min]', 'T_amb[degC]', 'V_air_outdoor[m/s]',
    'h_suc[kJ/kg]', 'h_dis[kJ/kg]',
    'h_eva_mid[kJ/kg]', 'h_eva_out[kJ/kg]',   # 蒸发焓（原气冷位置）
    'h_gc_in[kJ/kg]', 'h_gc_out[kJ/kg]'        # 气冷焓（原蒸发位置）
]
```

### 制冷 xlsx 汇总表中文列名

```python
cooling_names_cn = [
    '吸气压力(bar)', '吸气温度(℃)', '排气压力(bar)', '排气温度(℃)',
    '蒸发中间压力(bar)', '蒸发中间温度(℃)',
    '蒸发出口压力(bar)', '蒸发出口温度(℃)',
    '气冷进口压力(bar)', '气冷进口温度(℃)',
    '气冷出口压力(bar)', '气冷出口温度(℃)',
    '压缩机功耗(kW)', '一级制冷量(kW)', '二级制冷量(kW)', '制冷量(kW)',
    'COP', '质量流量(kg/s)', '进风温度(℃)', '中间温度(℃)',
    '出风温度(℃)', '室内风量(kg/min)', '环境温度(℃)', '室外风量(m/s)',
    '吸气比焓(kJ/kg)', '排气比焓(kJ/kg)', '蒸发中间比焓(kJ/kg)',
    '蒸发出口比焓(kJ/kg)', '气冷进口比焓(kJ/kg)', '气冷出口比焓(kJ/kg)'
]
```

### 制冷第一步：分列

```python
for f in sorted(glob.glob('cooling_*.csv')):
    df = pd.read_csv(f, sep=r'\s+', engine='python')
    df = df.iloc[:, :31]
    df.columns = cooling_names
    df.to_csv(f, index=False, encoding='utf-8-sig')
```

第二步（取 `df.iloc[-1]`）和第三步（导出 xlsx）与制热流程完全相同，只需将 `correct_names` 替换为 `cooling_names`，`param_names_cn` 替换为 `cooling_names_cn`。

---

# 充注量泄露诊断 — 机器学习流程

## 背景与目标

基于10个环境温度工况的仿真结果（制热-30℃~5℃，制冷25℃~50℃），构建跨温度通用诊断模型：
1. **可视化证明平台期存在**：因储液罐存在，总充注量增加但实际系统循环充注量存在平台期
2. **适充/欠充二分类**：判断系统是否处于适充状态
3. **欠充程度预测**：对欠充样本预测实际充注量，计算泄露百分比

## 算法方案

| 环节 | 方法 |
|------|------|
| 特征筛选 | Sobol敏感性分析 + 基尼不纯度（双重验证） |
| 分类模型 | 随机森林、SVM、神经网络（三算法对比） |
| 回归模型 | 随机森林、SVR、神经网络（三算法对比） |
| 调参方式 | RF/SVM → GridSearchCV；神经网络 → 贝叶斯优化(Optuna) |
| 模型范围 | 跨温度通用模型（T_amb作为特征输入） |

## 标签定义

- **适充**：实际充注量 >= 该温度下平台期值 × 95%
- **欠充**：实际充注量 < 该温度下平台期值 × 95%
- **平台期值**：自动检测（相邻变化率 < 2g/step 判定进入平台期）
- **泄露百分比** = (平台期值 - 预测实际充注量) / 平台期值 × 100%

## 目录结构（诊断部分）

```
├── 项目报告.md                         # 各步骤成果总结
├── 项目日志.md                         # 每步操作记录
├── 合并数据/                           # 步骤1
├── 平台期可视化/                       # 步骤2
├── 特征筛选/                           # 步骤4
├── 诊断模型/                           # 步骤5
└── 诊断结果/                           # 步骤6
```

## 各步骤概览

### 步骤1：数据整合
- 读取 `不同充注量结果/` 下10个xlsx
- 统一列名为英文key（消除制热/制冷命名差异）
- 添加 `T_amb`（环境温度）和 `mode`（heating/cooling）
- 导出 `合并数据/merged_data.csv`

### 步骤2：平台期可视化
- 实际充注量 vs 总充注量（分温度子图）
- 排气压力/排气温度/COP/吸气压力/制热量制冷量 vs 实际充注量
- 储液罐内充注量 vs 总充注量

### 步骤3：平台期检测 & 标签
- 自动检测各温度平台期起点和平台期值
- 按95%阈值打标签
- 输出带标签的完整数据集

### 步骤4：特征筛选
- Sobol敏感性分析（SALib，全阶指数ST排序）
- 基尼不纯度（随机森林feature_importances_）
- 两种方法对比，取交集Top特征

### 步骤5：模型训练
- 分类：RF + SVM + MLP → 适充/欠充
- 回归：RF + SVR + MLP → 预测实际充注量
- RF/SVM用GridSearchCV，MLP用Optuna贝叶斯优化
- 评价：Accuracy/F1/MAE/RMSE/R²

### 步骤6：结果汇总
- 模型对比图表
- 混淆矩阵 + 回归散点图
- 泄露诊断完整流程汇总
