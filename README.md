# Julia Pachinko ODE

Julia を使った産業動態の微分方程式モデリング。現在の主題は「パチンコ」ユーザー人口の動態分析。

---

## 概要

パチンコ産業は1994年のピーク（約2930万人）から2022年には約770万人まで縮小した。この衰退を「プレイヤーとノンプレイヤーの遷移モデル（SIR類似）」で数理的に分析し、ModelingToolkit.jl による微分方程式モデリングと Turing.jl によるベイズパラメータ推定を組み合わせる。

---

## ファイル

| ファイル | 説明 |
|---------|------|
| [`パチンコの微分方程式モデリング2026.ipynb`](パチンコの微分方程式モデリング2026.ipynb) | メインノートブック（40セル） |
| [`pdata2024.csv`](pdata2024.csv) | パチンコユーザー・日本人口データ（1994〜2023年、千人単位） |
| [`qiita_pachinko_ode.md`](qiita_pachinko_ode.md) | Qiita 向け解説記事 |

---

## 使用パッケージ

- **[ModelingToolkit.jl](https://github.com/SciML/ModelingToolkit.jl) v11** — 微分方程式の記号的定義・コンパイル
- **[Turing.jl](https://github.com/TuringLang/Turing.jl) v0.43** — MCMCによるベイズパラメータ推定（NUTS）
- **[DifferentialEquations.jl](https://github.com/SciML/DifferentialEquations.jl)** — ODE ソルバー
- **[DataInterpolations.jl](https://github.com/SciML/DataInterpolations.jl)** — 人口データの補間
- **[ForwardDiff.jl](https://github.com/JuliaDiff/ForwardDiff.jl)** — 補間曲線の数値微分
- **[StatsPlots.jl](https://github.com/JuliaPlots/StatsPlots.jl)** — 可視化

---

## モデルの概要

人口を「パチンコユーザー P」と「ノンユーザー N」に二分し、以下の方程式で動態を記述する。

```
dP/dt = e·(V - P) - r·P
N     = V - P          ← 保存則（代数方程式）
dV/dt = dvu(t)         ← 実データから補間した人口変化率
```

| パラメータ | 意味 |
|-----------|------|
| `e` | エントリー係数（ノンユーザー → ユーザーの遷移率） |
| `r` | リタイア係数（ユーザー → ノンユーザーの遷移率） |
| `V` | 総人口（または生産年齢人口）（状態変数） |
| `N` | ノンユーザー（代数変数 = V − P） |

---

## 分析手順

| # | セクション | 内容 |
|---|-----------|------|
| 1 | 基本シミュレーション | 定数パラメータでモデルの挙動を確認 |
| 2 | データ駆動人口変動 | 実人口データを補間して `dV/dt` を推定 |
| 3 | ベイズ推定（全期間） | NUTS サンプラーで `e`, `r` を一括推定・WAIC 計算 |
| 4 | カタストロフィーイベント | 風営法改正（1997）・東日本大震災（2011）を `discrete_events` で表現 |
| 5 | カタストロフィーモデル推定 | イベントあり版の Turing 推定・WAIC 比較 |
| 6 | 生産年齢人口モデル | `V = V_working`（15〜64歳）版での再推定・WAIC 比較 |
| 7 | 2030 年予測 | 人口 −900 千人/年 を仮定し 2030 年まで外挿・90% 予測区間 |

---

## モデル比較（WAIC）

WAIC（Widely Applicable Information Criterion）で汎化性能を比較する。値が小さいほど良い。

| モデル | 人口変数 | イベント | WAIC |
|--------|---------|---------|------|
| 全期間ベイズ推定 | 総人口 | なし | — |
| カタストロフィーモデル | 総人口 | 1997・2011年 | — |
| 生産年齢人口モデル | 生産年齢人口（15〜64歳） | 1997・2011年 | — |

（WAIC は MCMC 実行後に `calc_waic(P_mat, Pu, s_valid)` で計算）

---

## 実行環境

- Julia 1.12.5
- Jupyter Notebook（Julia カーネル）
- 日本語フォント: PlemolJP-Text（`gr(fontfamily="PlemolJP-Text")`）

---

## データ出典

- パチンコユーザー数・日本人口: 日本生産性本部「レジャー白書」
- スマートフォン普及率: 総務省情報通信白書（推定）

---

## 参考

- [ModelingToolkit.jl ドキュメント](https://docs.sciml.ai/ModelingToolkit/stable/)
- [Turing.jl ドキュメント](https://turinglang.org/docs/)
- [SciML エコシステム](https://sciml.ai/)
- [Qiita 記事](qiita_pachinko_ode.md)
