# 架构与技术栈

本页说明 TA-Lib 仓库的内部架构、代码生成管线、所提供算法的分类，以及关于 SIMD / GPU 硬件加速的现状与立场。

## 项目定位

TA-Lib（Technical Analysis Library）是一个开源的金融时间序列技术分析算法库。**C/C++ 是权威实现**，其它语言（.NET、Java、Rust 等）通过仓库内的代码生成器从同一份 C 源生成，以保证多语言绑定的算法语义完全一致。

当前版本：见仓库根目录 `VERSION` 文件。

## 顶层目录结构

| 路径 | 作用 |
|------|------|
| `src/ta_func/` | 算法实现，一函数一文件（约 158 个 TA 函数） |
| `src/ta_abstract/` | 抽象层与函数元数据；`ta_func_api.c` / `ta_func_api.xml` 是函数签名/参数的单一事实源 |
| `src/ta_common/` | 全局状态、内存管理、返回码、版本信息 |
| `src/tools/gen_code/` | 代码生成器（C 写的）：读取 GENCODE 段 + 元数据，输出多语言代码 |
| `src/tools/ta_regtest/` | 回归测试 |
| `include/` | 公开 C 头文件，包括跨语言宏 `ta_defs.h` |
| `rust/` | Rust crate（生成产物 + 手写胶水） |
| `dotnet/`、`java/`、`swig/` | .NET、Java、SWIG 绑定（生成产物） |
| `cmake/`、`CMakeLists.txt`、`configure.ac`、`Makefile.am` | 构建系统 |
| `docs/`、`mkdocs.yml` | 本文档站点 |

## 构建系统

- **CMake**（主，`CMakeLists.txt`）和 **Autotools**（`autogen.sh` / `configure.ac`）是当前唯二支持的构建系统。
- 历史上的 Visual Studio `.sln`、Xcode 工程以及 `make/` 目录已不再维护（详见 `CHANGELOG.md`）。
- Rust 子 crate 通过 `rust/Cargo.toml` 独立构建，无运行时依赖；`criterion` 仅用于基准测试。

典型开发循环（来自仓库 `CLAUDE.md`）：

```
修改生成器  →  bin/gen_code  →  cargo check && cargo fmt  →  bin/ta_regtest
```

> 关键原则：**改生成器，不改生成代码**。生成产物会在下一次 `gen_code` 运行时被覆盖。

## 代码生成管线

TA-Lib 的核心架构特征是「单一 C 源 → 多语言输出」：

```
                       ┌──────────────────────────────┐
                       │  src/ta_func/*.c             │
                       │   含 GENCODE 标记段          │
                       │  ta_abstract/ta_func_api.xml │
                       │   函数元数据（签名/参数）    │
                       └──────────────┬───────────────┘
                                      │
                                      ▼
                ┌─────────────────────────────────────────┐
                │  bin/gen_code  (src/tools/gen_code/)    │
                │   gen_code.c    通用流程 / 校验          │
                │   gen_rust.c    Rust 签名/语法           │
                │   ta_abstract/templates/*.template       │
                └──────────────┬──────────────────────────┘
                               │
        ┌──────────────┬───────┴───────┬───────────────┐
        ▼              ▼               ▼               ▼
       C/C++         .NET            Java           Rust
       (原地)        dotnet/         java/          rust/src/ta_func/
```

跨语言差异通过 `include/ta_defs.h` 中的宏统一抹平，例如：

- `FOR_EACH_OUTPUT(start, end, i, outIdx) ... FOR_EACH_OUTPUT_END(outIdx)`：循环结构在 C 与 Rust 之间表达不同。
- `OUTPUT_F64(val)`：在 Rust 单精度路径下显式 `as f64`，在 C 中是 no-op。
- `DECLARE_LOOP_VAR(i)`：在 Rust 下为 no-op（`for` 自带绑定），在 C 下声明 `int i`。
- 索引校验：`startIdx < 0` 在 Rust 的 `usize` 下无意义，生成器会跳过该分支。

详细记录见 `RUST_CONVERSION_PLAN.md` 与 `CLAUDE.md`。

## 多语言绑定状态

| 目标 | 状态 |
|------|------|
| C / C++ | 完整、稳定，参考实现 |
| .NET | 通过 `gen_code` 生成至 `dotnet/` |
| Java | 通过 `gen_code` 生成至 `java/` |
| SWIG | Python 等脚本语言绑定的接入点 |
| Rust | **进行中**。目前 `MULT` 已通过 `cargo check`，`SMA` 是下一目标；详见 `RUST_CONVERSION_PLAN.md` |
| R | 通过外部 wrapper（社区贡献，PR #71） |
| Python | 通过 [ta-lib-python](https://github.com/ta-lib/ta-lib-python) 独立项目维护 |

## 提供的算法（约 158 个）

完整字母列表见 [Functions List](functions.md)。下表按用途分组，便于横向了解能力范围：

| 类别 | 代表函数 |
|------|---------|
| 重叠研究 / 均线 | SMA, EMA, WMA, DEMA, TEMA, TRIMA, KAMA, MAMA, T3, BBANDS, MIDPOINT, MIDPRICE, SAR, SAREXT, MAVP, HT_TRENDLINE |
| 动量指标 | RSI, MACD, MACDEXT, MACDFIX, STOCH, STOCHF, STOCHRSI, ADX, ADXR, DX, APO, PPO, AROON, AROONOSC, CCI, CMO, MFI, MOM, ROC, ROCP, ROCR, ROCR100, TRIX, ULTOSC, WILLR, BOP, MINUS_DI, MINUS_DM, PLUS_DI, PLUS_DM, IMI |
| 成交量 | OBV, AD, ADOSC |
| 波动率 | ATR, NATR, TRANGE |
| 价格转换 | AVGPRICE, MEDPRICE, TYPPRICE, WCLPRICE |
| 周期 / Hilbert 变换 | HT_DCPERIOD, HT_DCPHASE, HT_PHASOR, HT_SINE, HT_TRENDMODE |
| 统计 | BETA, CORREL, LINEARREG, LINEARREG_ANGLE, LINEARREG_INTERCEPT, LINEARREG_SLOPE, STDDEV, TSF, VAR |
| 数学算子 | ADD, SUB, MULT, DIV, MAX, MIN, MAXINDEX, MININDEX, MINMAX, MINMAXINDEX, SUM, ACCBANDS, AVGDEV |
| 数学变换 | ACOS, ASIN, ATAN, COS, SIN, TAN, COSH, SINH, TANH, CEIL, FLOOR, EXP, LN, LOG10, SQRT |
| K 线形态识别（CDL\*） | 约 60 个，例如 CDLDOJI、CDLENGULFING、CDLHAMMER、CDLMORNINGSTAR、CDL3BLACKCROWS … |

## 硬件加速现状

简短结论：**当前上游仓库不包含任何 SIMD 或 GPU 加速实现**。

| 加速路径 | 是否存在 | 备注 |
|---------|--------|------|
| AVX / AVX2 / AVX-512 | ❌ 否 | 源码与构建脚本中均无 intrinsics、`-mavx*` 编译选项或运行时分发 |
| Intel AMX | ❌ 否 | 同上 |
| ARM NEON / SVE | ❌ 否 | 无 NEON intrinsics、无平台分支 |
| CUDA | ❌ 否 | 无 `.cu` 源、无 nvcc 集成、无运行时调用 |
| ROCm / HIP | ❌ 否 | 无 HIP 源、无相关 CMake 目标 |
| OpenCL / SYCL | ❌ 否 | 无 |
| 多线程并行 | ❌ 否 | 核心算法库为单线程标量实现 |

设计原因：

1. **算法本身的递推性质**。大多数指标（EMA、RSI、ADX、MACD …）是 O(N) 流式更新，前一拍依赖后一拍，难以直接 SIMD 化；强行向量化收益有限。
2. **跨平台与可移植性**。TA-Lib 的目标是「在任何 C 编译器下都能跑」，引入向量/异构后端会显著增加构建复杂度与维护成本。
3. **依赖编译器自动向量化**。在 `-O2` / `-O3` 下，部分算子（向量算术、`MULT` 等无依赖循环）可由编译器自动向量化。

仅有的「精度变体」是 Rust 端的 `single-precision` feature flag，输入接受 `f32`，仍是标量。

如果你的应用对吞吐量极敏感（例如批量回测数百万合约 × 多年分钟级数据），通常的做法是：

- 在 TA-Lib 之上做并行（按合约分片、按指标分片）；
- 使用 NumPy/Pandas 向量化的第三方实现（如 [`pandas-ta`](https://github.com/twopirllc/pandas-ta)）；
- 或在自有代码中用 SIMD/GPU 重写少数热点指标，把 TA-Lib 当作算法正确性的参考实现。

## 关键架构洞见

- **生成器优先**：函数原型、文档、参数校验都由 `gen_code` 产出，源头是 `ta_func_api.xml` + `ta_func/*.c` 中的 GENCODE 段。
- **宏抹平差异**：约 65% 的多语言差异由 `ta_defs.h` 中的宏覆盖，约 25% 由模板与生成器中的专门分支处理，剩余约 15% 仍需手工。
- **类型安全 vs C 兼容**：Rust 端使用 `usize` 索引，与 C 的 `int` 索引在边界条件上语义不同（如负数检查），生成器对此做了显式分流。

## 参考

- 函数清单：[Functions List](functions.md)
- C/C++ API：[C/C++ API](api.md)
- 各语言绑定：[Wrappers](wrappers.md)
- Rust 移植路线：仓库根 `RUST_CONVERSION_PLAN.md`
- 开发约定：仓库根 `CLAUDE.md`、`README-DEVS.md`
