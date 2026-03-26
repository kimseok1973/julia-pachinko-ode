# CLAUDE.md — Julia.lab プロジェクト設定

Claude Code がこのディレクトリで作業する際に参照する設定ファイル。

---

## プロジェクト概要

産業の隆盛（ブーム）と衰退のメカニズムを微分方程式モデルで分析する Julia ノートブック群。
現在の主題は「パチンコ」ユーザー人口の動態モデリング。

---

## 環境

- **言語**: Julia 1.12.5（安定版）
- **ノートブック形式**: Jupyter Notebook (.ipynb)、Julia カーネル
- **主要パッケージ**:
  - ModelingToolkit.jl v11
  - Turing.jl v0.43（内部が `VarNamedTuple` に変更）
  - DifferentialEquations.jl
  - DataInterpolations.jl
  - ForwardDiff.jl

---

## ファイル構成

| ファイル | 内容 | GitHub |
|---------|------|--------|
| `pdata2024.csv` | パチンコユーザー・人口データ（1994〜2023年、すべて千人単位） | ✅ |
| `スマートフォン普及率推移20240415_01.csv` | スマートフォン普及率（2010〜2024年、%）、Shift-JIS エンコード | ❌（ローカルのみ） |
| `パチンコの微分方程式モデリング2026.ipynb` | メインノートブック（42セル） | ✅ |
| `qiita_pachinko_ode.md` | Qiita 向け解説記事 | ❌（.gitignore）|

**GitHub リポジトリ**: https://github.com/kimseok1973/julia-pachinko-ode

---

## データ仕様

### pdata2024.csv
- **列**: `Year`, `パチンコユーザー(千人）`, `人口推移（千人）`, `生産年齢人口（千人）`, `備考`
- **読み込み後リネーム**: `P_user`, `V_population`, `V_working`（すべて千人、単位変換不要）
- **総人口モデル**: `Vu = V_population`、**生産年齢人口モデル**: `Vu_w = V_working`
- 2023年は `P_user` が欠損 → `dropmissing` で除外すると 29点（1994〜2022）
- 読み込みコード:
  ```julia
  df = CSV.read("pdata2024.csv", DataFrame)
  rename!(df, "パチンコユーザー(千人）" => :P_user,
              "人口推移（千人）"       => :V_population,
              "生産年齢人口（千人）"   => :V_working)
  dx = df[:, [:Year, :P_user, :V_population, :V_working]] |> dropmissing
  Pu   = Float64.(dx.P_user)
  Vu   = Float64.(dx.V_population)   # 総人口
  year = dx.Year
  n_data = length(Pu)   # 29点（1994〜2022）
  ```

### スマートフォン普及率推移
- エンコーディング: Shift-JIS（`CSV.read(..., encoding="shift-jis")` または事前変換が必要）
- 2行目が普及率（%）、列ヘッダーが年と調査数

---

## ノートブック構成（42セル）

| セル範囲 | 内容 |
|---------|------|
| 0〜12 | タイトル・基本モデル（定数人口変動 b） |
| 13〜20 | データ駆動モデル（`dvu` 補間・微分） |
| 21〜25 | Turing 全期間推定（カタストロフィーなし）+ `calc_waic` 関数 |
| 26〜33 | カタストロフィー概念デモ → Turing 推定 |
| 34〜37 | 生産年齢人口ベースモデル（`V_working`） |
| 38 | 2030 年予測（人口 −900 千人/年） |
| 39 | まとめ |
| 40 | 2030 年予測分布（`dvu_ext` + `sys_ext` + 事後サンプル・90% 確信区間プロット） |
| 41 | 2030 年 90% 確信区間の数値出力 |

---

## 数理モデル

### 基本方程式（SIR類似モデル）

```
dP/dt = e * (V - P) - r * P
N     = V - P                  ← 代数方程式（保存則）
dV/dt = dvu(t)                 ← データ補間、または dvu_w(t)（生産年齢人口版）
```

- `P`: パチンコユーザー人口（状態変数）
- `N`: ノンユーザー人口（代数変数 = V - P、observed）
- `V`: 総人口または生産年齢人口（状態変数）
- `e`: エントリー係数（ノンユーザー→ユーザーの遷移率）
- `r`: リタイア係数（ユーザー→ノンユーザーの遷移率）

**注**: `D(N) ~ r*P - e*(V-P) + D(V)` は `N ~ V - P` に置き換えること。
`structural_simplify` が DAE 化して N(t) の初期化エラーが発生するため。

---

## コーディング規約

### ModelingToolkit
- **`@mtkmodel` / `@mtkcompile` は使用しない**（`UndefVarError` が発生する環境のため）
- 代わりに `@named sys = ODESystem(eqs, t, vars, params)` + `structural_simplify` を使用
- 時間変数・微分演算子:
  ```julia
  const t = ModelingToolkit.t_nounits
  const D = ModelingToolkit.D_nounits
  ```
  （`using ModelingToolkit: t_nounits as t` の `as` エイリアス構文は使わない）
- モデル定義パターン:
  ```julia
  @parameters e r
  @variables P(t) N(t) V(t)
  eqs = [D(P) ~ ..., N ~ V - P, D(V) ~ ...]
  @named sys = ODESystem(eqs, t, [P, N, V], [e, r])
  sys = structural_simplify(sys)
  ```

### ODEProblem の初期条件
- N は代数変数（observed）なので初期条件に含めない
- 状態変数は P と V のみ:
  ```julia
  u0 = [sys.P => Pu[1], sys.V => Vu[1]]
  ps = [sys.e => 0.03, sys.r => 0.1]
  prob = ODEProblem(sys, vcat(u0, ps), tspan)
  ```

### DataInterpolations
- 補間範囲外アクセスを防ぐため `extrapolation_right = ExtrapolationType.Constant` を指定:
  ```julia
  using DataInterpolations: ExtrapolationType
  vu_d = QuadraticInterpolation(Vu, 0.0:Float64(n_data-1);
                                 extrapolation_right = ExtrapolationType.Constant)
  ```
- `@register_symbolic dvu(t)` は `dvu` 関数を定義した**後**に必ず実行すること
- 2030年予測用の外挿関数パターン（t > 28 で -900千人/年）:
  ```julia
  dvu_ext(τ) = τ <= Float64(n_data - 1) ?
      ForwardDiff.gradient(x -> vu_d(x[1]), [τ])[1] : -900.0
  @register_symbolic dvu_ext(t)
  ```

### Turing
- サンプラー: `NUTS()` を推奨
- 事前分布: `truncated(TDist(1), 1e-5, 0.1)`（裾の重い Cauchy 系、範囲を狭く制限）
- 解法失敗ガード（カタストロフィーなしモデル）: `length(solv[sys.P]) < n` で `Turing.@addlogprob! -Inf`
- 解法失敗ガード（イベントありモデル）: `solv.retcode != ReturnCode.Success` で `Turing.@addlogprob! -Inf`
- パラメータ更新: `remake(prob; p=[sys.e => entry, sys.r => retire])`
- Turing v0.43 では `MAP()` 初期化（`optimize(model, MAP())`）は削除済み。NUTS を直接実行する
- `x[i] ~ dist` の `x` は `AbstractArray` のみ可（Dict 不可）

### 離散イベントと saveat の注意
- `tstops` を指定すると停止直前・直後の点が追加保存され `saveat` の点数がずれる
- `saveat` は使わず、スカラー時刻補間で観測点を取り出す:
  ```julia
  solv(Float64(i - 1); idxs=P)   # DiffEqArray ではなく Float64 が返る
  ```
- `sol(t_array; idxs=P)` は `DiffEqArray` を返し Plots に渡せない → スカラーループで対処

### WAIC 計算
- `calc_waic` 関数（`waic-func-001` セル）を定義して全モデルで使いまわす:
  ```julia
  function calc_waic(P_mat::Matrix, obs::Vector, s_vec)
      n_obs, n_samp = size(P_mat)
      ss = collect(s_vec)[1:n_samp]
      log_lik = [logpdf(Normal(P_mat[i,j], ss[j]), obs[i]) for i in 1:n_obs, j in 1:n_samp]
      lppd   = sum(let c = log_lik[i,:]; mx=maximum(c); mx+log(mean(exp.(c.-mx))) end for i in 1:n_obs)
      p_waic = sum(var(log_lik[i,:]) for i in 1:n_obs)
      round(-2*(lppd-p_waic), digits=1)
  end
  ```
- 呼び出し: `calc_waic(P_mat, Pu, s_valid)`

### プロット規約
- `fillalpha = 0.5` に統一（`RGBA` 埋め込みアルファは使わない）
- `fillcolor` は `RGB(r, g, b)` を使用
- `titlefont = font(...)` など明示的なフォントサイズ指定は行わない（`default` の設定に従う）
- 90% 信用区間の境界線: `lw=0.8, ls=:dash, alpha=0.5`

### 一般
- 単位はすべて「千人」
- `Pu = Float64.(dx.P_user)` — Turing の型推論のため明示的に Float64 変換
- Plots 日本語設定: `ENV["GKS_ENCODING"] = "utf8"` + `gr(fontfamily="PlemolJP-Text")`
