---
title: "TimesFM 3をColabで試す：関連データや予定を足すと予測は変わる？"
emoji: "📈"
type: "tech"
topics: ["timesfm", "時系列", "機械学習", "python", "colab"]
published: false
---

## はじめに

「売上を予測するなら、来店数や来月のセール予定も一緒に見てほしい」。時系列モデルを使うとき、こんな情報も渡せたら、と思うことはありませんか。

TimesFM 3.0でうれしいのが、**関連する系列も、未来の予定も、モデルに直接まとめて渡せる**ところです。あらかじめ学習されたモデルなので、まずは追加学習なしで試せます。

たとえば、こんな比較ができます。

- **関連する系列を一緒に予測** → 1本ずつ予測するより役立つ？
- **週末や販促予定を追加** → 履歴だけでは読みにくい変化を拾える？
- **未来の販促をオフにする** → 予定を変えると予測はどう動く？

「この情報も使えそう」を試せるのがいいですね。今回は人工データで入力を変えながら比べ、最後に予測の誤差と幅も確かめます。手元にデータがなくても、Colabで始められます。

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunyeul/zenn-content/blob/main/notebooks/timesfm3_catchup_colab_ja.ipynb)

## 3.0では、情報をモデルに直接渡せる

2.5にも、補助情報を使うXRegという外部の回帰処理がありました。[3.0](https://github.com/google-research/timesfm)では、予測対象と補助情報をモデルに直接渡せます。系列間の関係を扱うのがVariate Attentionです。

試す側としては、**モデルに渡す情報を変えて、予測を比べられる**のがポイント。ただし、情報が増えれば当たるとは限りません。まずは履歴だけの予測を出して、比較の出発点を作りましょう。

## 0. Colabで試す準備をしよう

:::details 確認日とバージョン

- 環境・仕様の確認日：2026年9月2日
- 元の検証用Colab：`3.0.0`
- 本稿のインストール手順：`3.0.1`

:::

:::message alert

- ソースコード：Apache-2.0
- 事前学習済み重み：TimesFM Non-Commercial License v1.0（非商用・非本番用途に制限）

業務やサービスへの適用前に[ライセンス本文](https://huggingface.co/google/timesfm-3.0-pytorch/blob/main/LICENSE)を確認してください。
:::

試しやすさもうれしいポイントです。

- **約330M（3.3億）パラメータ**：[公式モデルのメタデータ](https://huggingface.co/api/models/google/timesfm-3.0-pytorch)では330,710,976個
- **今回の実行環境はT4 GPU**：このハンズオンの規模なら、ColabのT4で試せます

必要なメモリは系列数や入力の長さでも変わります。今回はランタイムをGPUに切り替え、上からセルを動かしていきましょう。

```python
!pip install -q "timesfm[torch]==3.0.1" matplotlib pandas
```

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import torch

from timesfm3 import TimesFM3Forecaster

SEED = 42
rng = np.random.default_rng(SEED)
torch.manual_seed(SEED)

print("PyTorch:", torch.__version__)
print("CUDA available:", torch.cuda.is_available())

if torch.cuda.is_available():
    print("GPU:", torch.cuda.get_device_name(0))
```

### モデルを取得するためにログイン

1. `google/timesfm-3.0-pytorch`のモデルページで、必要に応じてライセンス条件に同意する
2. Colabのシークレットに認証トークンを`HF_TOKEN`として保存する
3. 次のセルでログインする

```python
try:
    from google.colab import userdata
    from huggingface_hub import login

    hf_token = userdata.get("HF_TOKEN")
    login(token=hf_token, add_to_git_credential=False)
    print("Logged in with HF_TOKEN")
except Exception:
    print("HF login skipped")
```

### 学習済みモデルを読み込む

使うのは`TimesFM3Forecaster`。学習済みモデルを読み込めば、履歴を渡して予測を始められます。

```python
DEVICE = "cuda" if torch.cuda.is_available() else "cpu"

forecaster = TimesFM3Forecaster.from_pretrained(
    "google/timesfm-3.0-pytorch",
    device=DEVICE,
    per_core_batch_size=8,
)

print("device:", forecaster.device)
print("max context:", forecaster.global_context)
print("input patch:", forecaster.config.input_patch_length)
print("output patch:", forecaster.config.output_patch_length)
print("quantiles:", forecaster.config.quantiles)
```

```text
PyTorch: 2.11.0+cu128
CUDA available: True
GPU: Tesla T4
device: cuda
max context: 15360
input patch: 32
output patch: 64
quantiles: [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9]
```

このあと使う予測結果では、中心と幅の両方を見ていきます。

- `forecast`：中心の値、中央値（p50）
- `quantiles`：幅を見るための9段階の分位点（p10〜p90）

p10・p90は分布の下から10%・90%の位置。この間をグラフの帯にすると、予測の幅も見られます。

## 1. 履歴を渡すだけで、まず予測してみる

最初は追加学習なしの「ゼロショット予測」で、入力と出力をつかみます。長さの違う系列も、一度に渡せます。

- **入力**：長さの違う2本の系列。それぞれの履歴だけを使います
- **予測期間**：未来48点（日次データなら48日分）

### 予測のずれを数字とグラフで見る

結果を比べるため、誤差の計算と描画の関数を用意します。まず見るのは**MAE**。どの指標も、小さいほど予測のずれが小さいと読めます。

- **MAE**：予測のずれの絶対値を平均
- **RMSE**：大きなずれほど強く反映
- **sMAPE**：実測値・予測値の大きさに対するずれを割合で表示

```python
QUANTILES = np.asarray(forecaster.config.quantiles, dtype=float)

def quantile_index(q: float) -> int:
    return int(np.argmin(np.abs(QUANTILES - q)))

def metrics(y_true, y_pred):
    y_true = np.asarray(y_true)
    y_pred = np.asarray(y_pred)
    mae = np.mean(np.abs(y_true - y_pred))
    rmse = np.sqrt(np.mean((y_true - y_pred) ** 2))
    denom = np.maximum(np.abs(y_true) + np.abs(y_pred), 1e-8)
    smape = 200 * np.mean(np.abs(y_true - y_pred) / denom)
    return {"MAE": mae, "RMSE": rmse, "sMAPE(%)": smape}

def plot_univariate(context, output, actual=None, title=None, filename=None):
    h = len(output.forecast)
    x_ctx = np.arange(len(context))
    x_h = np.arange(len(context), len(context) + h)

    plt.figure(figsize=(13, 4))
    plt.plot(x_ctx, context, label="context")
    plt.plot(x_h, output.forecast, label="TimesFM p50", linewidth=2)

    if actual is not None:
        plt.plot(x_h, actual, "--", label="actual")

    lo = quantile_index(0.1)
    hi = quantile_index(0.9)
    plt.fill_between(
        x_h,
        output.quantiles[:, lo],
        output.quantiles[:, hi],
        alpha=0.2,
        label="p10–p90",
    )

    plt.axvline(len(context) - 1, linestyle=":", alpha=0.6)
    plt.title(title or "TimesFM 3.0 forecast")
    plt.legend()
    plt.grid(alpha=0.2)
    if filename:
        plt.savefig(filename, dpi=160, bbox_inches="tight")
    plt.show()
```

### データを作って予測する

```python
H = 48

def make_series(n, phase=0.0, trend=0.02, noise=0.12):
    t = np.arange(n, dtype=np.float32)
    y = (
        2.0
        + trend * t
        + 0.8 * np.sin(2 * np.pi * t / 24 + phase)
        + 0.35 * np.sin(2 * np.pi * t / 7)
        + rng.normal(0, noise, n)
    )
    return y.astype(np.float32)

full1 = make_series(256 + H, phase=0.0, trend=0.015)
full2 = make_series(180 + H, phase=0.8, trend=-0.004)

contexts = [full1[:-H], full2[:-H]]
actuals = [full1[-H:], full2[-H:]]

outputs = list(
    forecaster.predict_batch(
        contexts=contexts,
        horizon=H,
        return_quantiles=True,
        use_symmetric_averaging=False,
        make_positive=False,
    )
)

for i, (ctx, actual, out) in enumerate(zip(contexts, actuals, outputs)):
    print(
        f"series {i}: context={ctx.shape}, "
        f"forecast={out.forecast.shape}, quantiles={out.quantiles.shape}"
    )
    print(metrics(actual, out.forecast))
    plot_univariate(
        ctx,
        out,
        actual,
        f"Univariate series {i}",
        filename=f"univariate-series-{i}.png",
    )
```

![系列0の予測結果](/images/timesfm3/univariate-series-0.png)
*系列0の入力した履歴、実測値、p50予測、p10〜p90区間*

![系列1の予測結果](/images/timesfm3/univariate-series-1.png)
*系列1の入力した履歴、実測値、p50予測、p10〜p90区間*

長さをそろえずに2本を渡せました。ここではまだ、互いの情報を使わずに予測しています。

- `context`：入力した履歴
- `actual`：予測と比較する実測値

念のため、`forecast`がp50と同じ値かも見ておきます。

```python
p50_idx = quantile_index(0.5)

for i, out in enumerate(outputs):
    print(
        f"series {i} forecast == q50:",
        np.allclose(out.forecast, out.quantiles[:, p50_idx]),
    )
```

```text
quantiles: [0.1 0.2 0.3 0.4 0.5 0.6 0.7 0.8 0.9]
p50 index: 4
series 0 forecast == q50: True
series 1 forecast == q50: True
```

## 2. 関連する系列も、予測のヒントにしてみる

1本ずつの予測ができたので、今度は系列間の関係も使ってみます。共通のトレンドや周期を持つ3本を新しく作り、入力のまとめ方だけを変えます。

- **まとめて予測（`joint`）**：`(3, T)`の配列で渡し、系列間の関係も使う
- **個別に予測（`independent`）**：1本ずつ渡し、各系列の履歴だけを使う

`T`は履歴の長さです。予測を始める時点は、両方でそろえます。

```python
CTX = 256
H = 64
TOTAL = CTX + H
t = np.arange(TOTAL, dtype=np.float32)

latent = (
    0.015 * t
    + 1.2 * np.sin(2 * np.pi * t / 24)
    + 0.5 * np.sin(2 * np.pi * t / 7)
)

v0 = 20 + 3.0 * latent + rng.normal(0, 0.35, TOTAL)
v1 = 50 + 1.8 * latent + 0.25 * np.roll(v0, 1) + rng.normal(0, 0.45, TOTAL)
v2 = 10 - 1.2 * latent + 0.15 * np.roll(v1, 2) + rng.normal(0, 0.30, TOTAL)

target_full = np.vstack([v0, v1, v2]).astype(np.float32)
target_ctx = target_full[:, :CTX]
target_actual = target_full[:, CTX:]

joint_out = forecaster.predict(
    context=target_ctx,
    horizon=H,
    return_quantiles=True,
    use_symmetric_averaging=False,
    make_positive=False,
)

independent_outs = list(
    forecaster.predict_batch(
        contexts=[target_ctx[i] for i in range(3)],
        horizon=H,
        return_quantiles=True,
        use_symmetric_averaging=False,
        make_positive=False,
    )
)
independent_forecast = np.stack([o.forecast for o in independent_outs])

rows = []
for i in range(3):
    rows.append({
        "variate": i,
        "joint_MAE": metrics(target_actual[i], joint_out.forecast[i])["MAE"],
        "independent_MAE": metrics(
            target_actual[i], independent_forecast[i]
        )["MAE"],
    })

joint_comparison = pd.DataFrame(rows)
joint_comparison
```

```python
fig, axes = plt.subplots(3, 1, figsize=(14, 9), sharex=True)
x_ctx = np.arange(CTX)
x_h = np.arange(CTX, CTX + H)

for i, ax in enumerate(axes):
    ax.plot(x_ctx, target_ctx[i], label=f"variate {i} context")
    ax.plot(x_h, target_actual[i], "--", label="actual")
    ax.plot(x_h, joint_out.forecast[i], label="joint")
    ax.plot(x_h, independent_forecast[i], label="independent", alpha=0.8)
    ax.axvline(CTX - 1, linestyle=":", alpha=0.5)
    ax.legend(loc="upper left")
    ax.grid(alpha=0.2)

plt.suptitle("Native multivariate vs independent univariate")
plt.tight_layout()
plt.savefig("joint-vs-independent.png", dpi=160, bbox_inches="tight")
plt.show()
```

| 系列 | まとめて予測したMAE | 個別に予測したMAE | 誤差が小さい方法 |
| ---: | ---: | ---: | --- |
| 0 | 0.319287 | 0.332795 | まとめて予測 |
| 1 | 0.414138 | 0.432287 | まとめて予測 |
| 2 | 0.286407 | 0.299531 | まとめて予測 |

![まとめて予測した場合と個別に予測した場合の比較](/images/timesfm3/joint-vs-independent.png)
*同じ3系列を共同予測した場合と、1本ずつ予測した場合の比較*

- **3本ともMAEが少し小さくなりました。** この例では、まとめて渡すほうがよい結果です
- ただし、共通の動きを持たせた人工データの1区間だけ。「いつでも有利」とはまだ言えません

## 3. 「来週はセール」も予測に使いたい

ここからは、売上と来店数の例です。履歴に加えて「来週はセール」という情報も伝えてみます。こうした補助情報が「共変量」。**過去しか分からない情報と、未来まで分かる情報**に分けて渡します。

- 予測対象：売上（`sales`）と来店数（`traffic`）
- 過去だけ分かる共変量：競合の動向を表す指標（`competitor_signal`）
- 未来まで分かる共変量：販促の有無（`promotion`）と週末かどうか（`weekend`）

### 入力の形をそろえる

- `V`：予測対象の数
- `T` / `H`：履歴の長さ / 予測する長さ
- `C_po` / `C_pf`：過去のみ / 未来まで分かる共変量の数

| 入力・出力 | 配列の形 | 用意する値・返される値 |
| --- | ---: | --- |
| 予測対象の履歴：`context` | `(V, T)` | 予測したい各系列の過去の値 |
| 過去だけ分かる共変量：`past_only_covariates` | `(C_po, T)` | 予測時点までに観測した補助情報 |
| 未来まで分かる共変量：`past_future_covariates` | `(C_pf, T + H)` | 過去の値と、予測期間の終わりまでの予定など |
| 予測の中央値：`forecast` | `(V, H)` | 各系列のp50予測 |
| 予測の分位点：`quantiles` | `(V, H, 9)` | 各系列のp10〜p90予測 |

```python
CTX = 256
H = 64
TOTAL = CTX + H
t = np.arange(TOTAL, dtype=np.float32)

weekend = ((t.astype(int) % 7) >= 5).astype(np.float32)

promotion = np.zeros(TOTAL, dtype=np.float32)
promotion[60:70] = 1
promotion[145:154] = 1
promotion[CTX + 10:CTX + 18] = 1
promotion[CTX + 38:CTX + 48] = 1

competitor_signal = (
    0.7 * np.sin(2 * np.pi * t / 31)
    + rng.normal(0, 0.15, TOTAL)
).astype(np.float32)

weekly = np.sin(2 * np.pi * t / 7)

sales = (
    100 + 7 * weekly + 10 * weekend + 28 * promotion
    + 5 * competitor_signal + rng.normal(0, 2.0, TOTAL)
)
traffic = (
    220 + 12 * weekly + 25 * weekend + 55 * promotion
    + 8 * competitor_signal + rng.normal(0, 4.0, TOTAL)
)

targets_full = np.vstack([sales, traffic]).astype(np.float32)
target_context = targets_full[:, :CTX]
actual_future = targets_full[:, CTX:]

past_only_cov = competitor_signal[:CTX][None, :]
past_future_cov = np.vstack([promotion, weekend]).astype(np.float32)

print("target:", target_context.shape)
print("past-only:", past_only_cov.shape)
print("past-future:", past_future_cov.shape)
```

```text
target: (2, 256)
past-only: (1, 256)
past-future: (2, 320)
```

ここで気をつけたいのは、**予測した時点で本当に分かっていた情報だけを使うこと**です。

- 週末・確定済みの販促：未来まで入力できる
- 競合の実績：予測時に分かっている過去分だけを使う

### 補助情報を加えると、どれくらい変わる？

```python
out_with_cov = forecaster.predict(
    context=target_context,
    horizon=H,
    past_only_covariates=past_only_cov,
    past_future_covariates=past_future_cov,
    return_quantiles=True,
    use_symmetric_averaging=False,
    make_positive=True,
)

out_without_cov = forecaster.predict(
    context=target_context,
    horizon=H,
    return_quantiles=True,
    use_symmetric_averaging=False,
    make_positive=True,
)

comparison = []
for i, name in enumerate(["sales", "traffic"]):
    comparison.append({
        "target": name,
        "MAE_with_covariates": metrics(
            actual_future[i], out_with_cov.forecast[i]
        )["MAE"],
        "MAE_without_covariates": metrics(
            actual_future[i], out_without_cov.forecast[i]
        )["MAE"],
    })

covariate_comparison = pd.DataFrame(comparison)
covariate_comparison
```

```python
fig, axes = plt.subplots(2, 1, figsize=(14, 8), sharex=True)
x_ctx = np.arange(CTX)
x_h = np.arange(CTX, CTX + H)

for i, name in enumerate(["sales", "traffic"]):
    ax = axes[i]
    ax.plot(x_ctx, target_context[i], label="context")
    ax.plot(x_h, actual_future[i], "--", label="actual")
    ax.plot(x_h, out_with_cov.forecast[i], label="with covariates")
    ax.plot(x_h, out_without_cov.forecast[i], label="without covariates")
    ax.axvline(CTX - 1, linestyle=":", alpha=0.5)
    ax.set_title(name)
    ax.legend(loc="upper left")
    ax.grid(alpha=0.2)

plt.tight_layout()
plt.savefig("with-vs-without-covariates.png", dpi=160, bbox_inches="tight")
plt.show()
```

| 予測対象 | 共変量ありのMAE | 共変量なしのMAE | 誤差が小さい条件 |
| --- | ---: | ---: | --- |
| 売上 | 2.337771 | 8.745473 | 共変量あり |
| 来店数 | 4.999482 | 17.582790 | 共変量あり |

![共変量あり・なしの予測比較](/images/timesfm3/with-vs-without-covariates.png)
*販促・週末・競合の情報を加えた場合と、売上・来店数の履歴だけで予測した場合の比較*

- **売上・来店数の両方でMAEが小さくなりました。** グラフの`with covariates`が補助情報ありです
- 今回は販促や週末が影響するように作ったデータ。手元でも`without covariates`（補助情報なし）と比べてみてください

## 4. 「セールをやめたら？」も比べてみる

補助情報ありの予測が出たので、同じ販売データで予定だけを変えてみます。これが「シナリオ予測」。過去の履歴と他の情報は固定します。

- 計画A：予定どおりに販促を実施する
- 計画B：未来の販促を取りやめる（`promotion`を0にする）

「Aの予測 − Bの予測」を期間全体で平均し、予定の違いが予測にどれくらい表れるかを見てみます。

```python
pfc_no_future_promo = past_future_cov.copy()
pfc_no_future_promo[0, CTX:] = 0.0

out_no_future_promo = forecaster.predict(
    context=target_context,
    horizon=H,
    past_only_covariates=past_only_cov,
    past_future_covariates=pfc_no_future_promo,
    return_quantiles=True,
    use_symmetric_averaging=False,
    make_positive=True,
)

scenario_delta = out_with_cov.forecast - out_no_future_promo.forecast

print("Average forecast difference over horizon")
print("sales:", scenario_delta[0].mean())
print("traffic:", scenario_delta[1].mean())
```

```python
fig, axes = plt.subplots(3, 1, figsize=(14, 9), sharex=True)

x_h = np.arange(CTX, CTX + H)

axes[0].plot(x_ctx, target_context[0], label="context")
axes[0].plot(x_h, out_with_cov.forecast[0], label="promo schedule")
axes[0].plot(x_h, out_no_future_promo.forecast[0], label="future promo off")
axes[0].plot(x_h, actual_future[0], "--", label="actual")
axes[0].set_title("Sales scenario forecast")
axes[0].legend()
axes[0].grid(alpha=0.2)

axes[1].plot(x_ctx, target_context[1], label="context")
axes[1].plot(x_h, out_with_cov.forecast[1], label="promo schedule")
axes[1].plot(x_h, out_no_future_promo.forecast[1], label="future promo off")
axes[1].plot(x_h, actual_future[1], "--", label="actual")
axes[1].set_title("Traffic scenario forecast")
axes[1].legend()
axes[1].grid(alpha=0.2)

axes[2].step(
    np.arange(TOTAL),
    promotion,
    where="mid",
    label="promotion",
)
axes[2].axvline(CTX - 1, linestyle=":", alpha=0.5)
axes[2].set_title("Past + future promotion covariate")
axes[2].legend()
axes[2].grid(alpha=0.2)

plt.tight_layout()
plt.savefig("promotion-scenario.png", dpi=160, bbox_inches="tight")
plt.show()
```

2本の予測線を見比べてみてください。`promo schedule`がA、`future promo off`がB。下段がAの販促日程です。

![販促予定を変えた場合の予測比較](/images/timesfm3/promotion-scenario.png)
*売上・来店数の履歴を固定し、未来の販促予定だけを変えた条件付き予測*

:::message
得られるのは**条件を変えたときの予測の差（Conditional Forecast Difference）**です。販促の因果効果ではありません。
:::

需要が高そうな日に販促を組んでいたら、売上増を全部そのおかげにはできませんよね。実際の効果を知るには、販促日の選び方や他の要因も考えた分析が必要です。

## 5. TimesFMを使う価値があるか、比べてみる

予定の比較とは別に、予測そのものの精度も見ておきたいところです。ここでは**長い1本の人工系列を新しく作り**、開始時点をずらす「ローリングバックテスト」で単純な方法と比べます。

- **比較対象**：直近24点を繰り返す季節ナイーブ法（seasonal naive）
- **評価回数**：開始時点をずらして5回
- **入力範囲**：各時点までの履歴のみ（共変量は使いません）

```python
def seasonal_naive(context, horizon, period):
    context = np.asarray(context)
    pattern = context[-period:]
    repeats = int(np.ceil(horizon / period))
    return np.tile(pattern, repeats)[:horizon]

def rolling_backtest(
    model,
    series,
    context_length=192,
    horizon=24,
    n_windows=5,
    step=24,
    seasonal_period=24,
):
    series = np.asarray(series, dtype=np.float32)
    last_cut = len(series) - horizon
    cuts = [last_cut - step * i for i in range(n_windows)][::-1]

    contexts = []
    actuals = []
    for cut in cuts:
        if cut - context_length < 0:
            raise ValueError("series is too short for the requested backtest")
        contexts.append(series[cut - context_length:cut])
        actuals.append(series[cut:cut + horizon])

    tfm_outputs = list(
        model.predict_batch(
            contexts=contexts,
            horizon=horizon,
            return_quantiles=True,
            use_symmetric_averaging=False,
            make_positive=False,
        )
    )

    rows = []
    for window, (cut, ctx, actual, out) in enumerate(
        zip(cuts, contexts, actuals, tfm_outputs)
    ):
        naive = seasonal_naive(ctx, horizon, seasonal_period)
        tfm_m = metrics(actual, out.forecast)
        naive_m = metrics(actual, naive)

        rows.append({
            "window": window,
            "cut": cut,
            "TimesFM_MAE": tfm_m["MAE"],
            "SeasonalNaive_MAE": naive_m["MAE"],
            "TimesFM_sMAPE": tfm_m["sMAPE(%)"],
            "SeasonalNaive_sMAPE": naive_m["sMAPE(%)"],
        })

    return pd.DataFrame(rows), tfm_outputs, actuals
```

```python
long_series = make_series(700, phase=0.3, trend=0.008, noise=0.18)

bt, bt_outputs, bt_actuals = rolling_backtest(
    forecaster,
    long_series,
    context_length=256,
    horizon=48,
    n_windows=5,
    step=48,
    seasonal_period=24,
)

display(bt)
display(
    bt[[
        "TimesFM_MAE",
        "SeasonalNaive_MAE",
        "TimesFM_sMAPE",
        "SeasonalNaive_sMAPE",
    ]].mean().to_frame("mean")
)
```

| 指標（5区間の平均） | TimesFM | 季節ナイーブ法 | 誤差が小さい方法 |
| --- | ---: | ---: | --- |
| MAE | 0.156712 | 0.451678 | TimesFM |
| sMAPE | 2.405938 | 7.034830 | TimesFM |

- **平均ではTimesFMの誤差が小さくなりました。** MAE・sMAPEともに同じ傾向です
- 毎回勝ったとは限らないので、区間ごとの表も見ておきましょう

## 6. 予測の「幅」も頼りになる？

同じ5回の予測で、今度は「幅」を見ます。p10〜p90の帯は中央80%に相当しますが、実測値がその割合で収まるかも確かめたいところです。

確かめるのは、帯の中に入った割合「被覆率（coverage）」。100点中80点が入れば80%です。

```python
def interval_coverage(actual, quantiles, lower_q=0.1, upper_q=0.9):
    lo = quantiles[..., quantile_index(lower_q)]
    hi = quantiles[..., quantile_index(upper_q)]
    actual = np.asarray(actual)
    return np.mean((actual >= lo) & (actual <= hi))

coverages = []
for window, (actual, out) in enumerate(zip(bt_actuals, bt_outputs)):
    coverage = interval_coverage(actual, out.quantiles, 0.1, 0.9)
    coverages.append(coverage)
    print(f"window {window} p10-p90 coverage: {coverage:.3f}")

print("mean coverage:", np.mean(coverages))
```

```text
window 0 p10-p90 coverage: 0.750
window 1 p10-p90 coverage: 0.792
window 2 p10-p90 coverage: 0.958
window 3 p10-p90 coverage: 0.833
window 4 p10-p90 coverage: 0.854
mean coverage: 0.8375
```

- **平均は83.75%。** 目安の80%に近い値になりました
- ただし、5区間だけで「この幅なら安心」とは言えません
- 手元では系列・評価時点を増やし、「翌日」「数週間先」でも分けて見てみてください

## 7. 手元のデータで試す前に

### クラスを変えるなら、設定もチェック

`Forecaster`と`Evaluator`は既定値が違います。**比較するときは設定もそろえましょう。**

- `TimesFM3Forecaster`：一般推論用。以下の3項目は既定で`False`
- `TimesFM3Evaluator`：公式の性能評価用。同じ3項目が既定で有効

対象は`return_quantiles`、`use_symmetric_averaging`、`make_positive`です。

:::details Evaluatorの系列の扱い

- 入力は最大32系列ずつ処理
- `univariate=True`では各系列を独立に評価

:::

### 64点より先も予測できる

予測したい長さを`horizon`に指定すればOKです。内部では64の倍数に切り上げ、必要な分だけ返します。ここでは160点を指定してみます。

```python
long_out = forecaster.predict(
    context=make_series(384),
    horizon=160,
    return_quantiles=True,
)

print("forecast:", long_out.forecast.shape)
print("quantiles:", long_out.quantiles.shape)
```

```text
forecast: (160,)
quantiles: (160, 9)
```

### 欠損値は埋めてくれる。でも意味は確認しよう

欠損があっても、次の処理で予測を進めてくれます。

1. 先頭からすべての予測対象がNaNになっている区間を除く
2. 残ったNaNを線形補間で埋める

```python
nan_ctx = make_series(200).copy()
nan_ctx[:10] = np.nan
nan_ctx[70:75] = np.nan

nan_out = forecaster.predict(
    context=nan_ctx,
    horizon=24,
    return_quantiles=True,
)

print("forecast shape:", nan_out.forecast.shape)
print("contains NaN:", np.isnan(nan_out.forecast).any())
```

```text
forecast shape: (24,)
contains NaN: False
```

値が返ってくるのは便利ですが、「営業なし」と「計測故障」では欠損の意味が違います。手元のデータでは、何を埋めてよいかを先に決めておきましょう。

## まとめ

最初の「関連する系列やセール予定も使いたい」は、モデルに渡す入力を変えることで試せました。

- **系列をまとめる・補助情報を加える**：それぞれの実験でMAEが小さくなりました
- **販促予定を変える**：条件ごとの予測を並べられます。因果効果とは区別します
- **精度と幅を確かめる**：別の単一系列では、5区間平均の誤差と被覆率を確認しました

いずれも乱数の種を42に固定した人工データの結果です。実データでの改善や、公式の性能評価の再現を示したものではありません。

手元で試すなら、まず「使えそうな情報」を一つ足し、同じ評価条件で誤差が減るか見てみてください。**追加学習なしで、この比較から始められる**のが3.0のうれしいところです。

## 参考

- [Google Research / TimesFM GitHub repository](https://github.com/google-research/timesfm)
- [TimesFM 3.0 PyTorch model card](https://huggingface.co/google/timesfm-3.0-pytorch)
- [TimesFM Non-Commercial License v1.0](https://huggingface.co/google/timesfm-3.0-pytorch/blob/main/LICENSE)
- [timesfm on PyPI](https://pypi.org/project/timesfm/)
- [A decoder-only foundation model for time-series forecasting](https://arxiv.org/abs/2310.10688)
