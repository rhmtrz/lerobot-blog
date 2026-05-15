---
title: "LeRobot — 録画したデータセットでポリシーをトレーニングする"
emoji: "🧠"
type: "tech"
topics: ["lerobot", "ai", "machinelearning", "pytorch", "robotics"]
published: true
publication_name: "adawarp"
---

> **シリーズ:** LeRobot チュートリアル — 第 4 回 / 全 5 回
> 前回: [エピソードデータセットの録画(第 3 回)](./lerobot-record-dataset) ・ 次回: [学習済みモデルの実行(第 5 回)](./lerobot-run-trained-model) *(近日公開)*

## はじめに

leader の軌跡、follower の関節状態、1〜2 本のカメラ映像が `LeRobotDataset` 形式でまとまったデータセットができたところです。この記事ではそのデータセットから、後で同じロボット上で実行できる **ポリシーのチェックポイント** を作ります。

デフォルトのポリシーは **ACT(Action-Chunking Transformer)** を使います。SO-101 で 50 エピソード程度の場合は ACT が最も扱いやすく、Diffusion Policy や Pi0 に手を出す前にまず通すべきポリシーです。

## ゴール

この記事を読み終えると、以下ができるようになります。

- 少数エピソードの SO-101 データセットに対して、どのポリシーを選べばいいかがわかる
- `lerobot-train` を実行し、トレーニングログを読める
- チェックポイントがどこに書き出されるか、途中再開するにはどうするかがわかる
- (オプション)学習済みポリシーを Hugging Face Hub に push して別マシンから利用する

## 前提

- [第 3 回](./lerobot-record-dataset) で録画したデータセット。同じタスクで 30〜50 エピソードあると理想的です
- Ubuntu 22.04(本記事は Linux 環境のみを対象にします)
- CUDA がセットアップ済みの NVIDIA GPU。ACT はデフォルト設定なら RTX 3060(12 GB)程度で問題なく学習できます。Apple silicon(MPS)でも動きますが遅く、未実装オペレータに当たることがあります([つまずきポイント](#つまずきポイント)で触れます)
- (オプション)[Weights & Biases](https://wandb.ai/) のアカウント。ターミナルログ以外にブラウザで loss を見たい場合に使います

## 環境

| コンポーネント | バージョン |
|----------------|------------|
| `lerobot`      | `<commit or tag>` |
| Python         | `3.10+` |
| CUDA           | `12.x`(PyTorch のホイールに合わせる) |
| GPU            | ACT のデフォルト設定なら VRAM 8 GB 以上を推奨 |
| ハードウェア   | SO-101 follower(推論時のみ必要。この記事の範囲では不要) |

## 手順

### 1. ポリシーを選ぶ

最初は **ACT** を選んでください。理由:

- SO-100 / SO-101 のような単腕・双腕の模倣学習タスク向けに設計されている
- 30 エピソード程度から収束する
- 純粋な模倣学習で、報酬信号も環境も rollouts も不要
- LeRobot のデフォルトハイパーパラメータが妥当で、最初はほぼ触る必要がない

「ACT で動いたら次に試す」候補:

- **Diffusion Policy** — 長期間タスクで ACT より精度が上がる場合がある。代わりにエピソード数(目安 150 以上)とトレーニング時間が必要
- **Pi0** / **SmolVLA** — 事前学習済みの基盤ポリシー。複数タスクを言語で条件付けたい場合に有用。ファインチューニングも `lerobot-train` でできますが、ベースとなるチェックポイントのフラグを指定します

### 2. (オプション)Weights & Biases にログイン

loss カーブや勾配のグラフをブラウザで見たい場合は次を実行します。

```bash:terminal
pip install wandb
wandb login
```

そのうえで、後述のトレーニングコマンドに `--wandb.enable=true --wandb.project=<your-project>` を追加します。ターミナルログだけで十分ならこの手順はスキップで OK です。

### 3. train 用のラッパースクリプトを書く

`lerobot-train` のフラグも長くなりがちなので、再現性のためにスクリプトにまとめます。

```bash:scripts/train.sh
#!/bin/bash
set -e
source "$(dirname "$0")/../lerobot/.venv/bin/activate"

HF_USER=${HF_USER:-$(whoami)}
DATASET_NAME=${DATASET_NAME:-so101_pick_place}
RUN_NAME=${RUN_NAME:-act_${DATASET_NAME}_$(date +%Y%m%d_%H%M)}

lerobot-train \
    --dataset.repo_id="${HF_USER}/${DATASET_NAME}" \
    --policy.type=act \
    --policy.device=cuda \
    --output_dir="outputs/train/${RUN_NAME}" \
    --job_name="${RUN_NAME}" \
    --batch_size=8 \
    --steps=100000 \
    --save_freq=10000 \
    --log_freq=200 \
    --wandb.enable=false \
    --policy.push_to_hub=false
```

各フラグの意味:

- **`--dataset.repo_id`** — 第 3 回で指定した id と同じものを使います。ローカルにある場合は `~/.cache/huggingface/lerobot/<repo_id>/` から、Hub にある場合は LeRobot が自動的にダウンロードします。
- **`--policy.type=act`** — ACT を選択。`diffusion`、`pi0`、`smolvla`、`tdmpc` なども指定できます。
- **`--policy.device=cuda`** — NVIDIA GPU を使用。Apple silicon なら `mps`、フォールバックなら `cpu`(遅いです)。
- **`--output_dir`** — このランの設定・チェックポイント・ログがすべてここに書き出されます。
- **`--job_name`** — ログや W&B のラン名に表示されます。ラン同士を区別できるよう、毎回ユニークな名前を付けてください。
- **`--batch_size=8`** — VRAM 8 GB 程度の安全なデフォルト。余裕があれば 16 や 32 に上げて構いません。
- **`--steps=100000`** — 勾配ステップ数(エポック数ではありません)。50 エピソードの ACT なら 10 万ステップが妥当な初期値です。
- **`--save_freq=10000`** — N ステップごとにチェックポイントを保存します。小さくすると復旧ポイントが増える代わりにディスクを食います。
- **`--log_freq=200`** — ターミナルに loss を出力する頻度。
- **`--policy.push_to_hub=false`** — 実機での動作確認(第 5 回)が済むまでは false のままにしておきます。

### 4. トレーニングを開始する

```bash:terminal
chmod +x scripts/train.sh
./scripts/train.sh 2>&1 | tee outputs/train/last_run.log
```

`tee` でログ全体をファイルに残しておくと、夜間に何かこけても `screen` / `tmux` を使わずに後から確認できて便利です。

出力は次の順に出てきます。

1. 解決済みのフル設定(デフォルト + 自分のオーバーライド)。データセットのパス・ポリシータイプ・デバイスが意図どおりか、ここで一度目を通します。
2. データセット読み込み: `Loaded dataset with N episodes`。
3. `--log_freq` 間隔のステップログ:

```text
step: 200  loss: 0.812  grad_norm: 1.43  lr: 1.0e-05  update_s: 0.124  data_s: 0.041
step: 400  loss: 0.534  grad_norm: 0.91  lr: 1.0e-05  update_s: 0.118  data_s: 0.038
...
```

最初の数千ステップで loss が大きく下がり、その後落ち着きます。小さめのデータセットでの ACT では 0.05〜0.15 付近で頭打ちになるのが普通です。

4. `--save_freq` ステップごとのチェックポイント保存:

```text
INFO  Saving checkpoint at step 10000 to outputs/train/<RUN_NAME>/checkpoints/010000/
```

### 5. チェックポイントの保存場所

出力ディレクトリの構造:

```
outputs/train/<RUN_NAME>/
├── config.json
├── checkpoints/
│   ├── 010000/
│   │   ├── pretrained_model/
│   │   │   ├── config.json
│   │   │   └── model.safetensors
│   │   └── training_state/
│   │       ├── optimizer.bin
│   │       └── scheduler.bin
│   ├── 020000/
│   ├── ...
│   └── last/ -> ../checkpoints/100000/
└── logs/
```

`pretrained_model/` が推論時に必要なフォルダで、第 5 回や `lerobot-record --policy.path=...` が読み込むのもここです。`training_state/` はトレーニングを再開するときだけ必要です。

### 6. チェックポイントから再開する

電源・OOM・SSH 切断などで途中で止まってしまった場合は、最新チェックポイントから再開できます。

```bash:terminal
lerobot-train \
    --config_path="outputs/train/${RUN_NAME}/checkpoints/last/pretrained_model/" \
    --resume=true
```

`--resume=true` は optimizer・scheduler の状態を `training_state/` から読み込むので、LR スケジュール・ステップカウント・シード(設定していれば)も含めて完全に止まった地点から続行します。

### 7. (オプション)Hub に push する

loss カーブに満足したら、別マシンで使えるように Hub に上げます。

```bash:terminal
huggingface-cli login   # 初回のみ
```

そのうえで、次のいずれかを実行します。

`train.sh` に以下のフラグを追加して再実行する:

```
--policy.push_to_hub=true \
--policy.repo_id="${HF_USER}/act_so101_pick_place"
```

…もしくは、既存の `pretrained_model/` フォルダを手動でアップロードします。

```bash:terminal
huggingface-cli upload \
    "${HF_USER}/act_so101_pick_place" \
    "outputs/train/${RUN_NAME}/checkpoints/last/pretrained_model/"
```

## つまずきポイント

:::message
**エラー**: `torch.cuda.OutOfMemoryError: CUDA out of memory.`
**原因**: バッチサイズが GPU に対して大きすぎる。
**対処**: `--batch_size` を下げる(4 → 2 と段階的に)。それでも厳しい場合、新しめのバージョンの ACT は勾配累積に対応しています。`lerobot-train --help` で現在のフラグ名を確認してください。
:::

:::message
**症状**: 最初の数千ステップで loss は下がるが、その後高い値(0.5 以上)で止まってしまう
**原因**: ボトルネックは最適化ではなくデータセット側です。エピソード数が足りない、デモが一貫していない、カメラの画角がズレている、などの可能性が高いです。
**対処**: 第 3 回の手順 6 のとおり `visualize_dataset` で再生し、本当にタスクが映っているかを目で確認します。怪しいエピソードは録り直し、データセットを綺麗にしてから最初からトレーニングを回し直してください。
:::

:::message
**症状**: macOS で MPS のオペレータ未実装エラー(`aten::...` not implemented)が出る
**原因**: PyTorch の MPS バックエンドが ACT で使われる一部の op に対応していない。
**対処**: `--policy.device=cpu` でまずパイプライン全体が通ることを確認し、トレーニング自体は Linux + CUDA のマシンに移すか、MPS バックエンドの対応が進むのを待ちます。推論側は MPS で動くケースが多いです。
:::

:::message
**症状**: データセット読み込みで `KeyError: 'observation.images.front'`(などのカメラ名)で落ちる
**原因**: データセットのカメラ名と、ポリシーが期待するカメラ名が一致していない。`record.sh` の cameras 辞書とポリシー設定がズレているケースが多いです。
**対処**: カメラ名を揃えて録り直すか、`--policy.input_features=...` のオーバーライドでデータセットの内容に合わせます。
:::

:::message
**症状**: トレーニング最初のステップで `RuntimeError: shape mismatch ...` が出る
**原因**: record と train の間でキャリブレーションがズレた(`--robot.id` が違う、モーターをファーム書き換えして action 次元が変わった、など)。
**対処**: データセットの `meta/info.json` の `action.shape` と `observation.state.shape` を確認し、ポリシー設定と一致しているかをチェック。一致しない場合は正しいキャリブレーションで録り直してください。
:::

:::message
長いエラートレースバックを共有するときは、`:::details` ブロックで折りたたんでおくと記事が読みやすくなります。
:::

## まとめ

- SO-101 で 50 エピソード程度のデータセットなら、まずは ACT を選びます。Diffusion Policy や Pi0 は ACT が動いてから検討するのがおすすめです。
- トレーニングは `lerobot-train` を `--policy.type` / `--dataset.repo_id` / `--output_dir` / `--steps` / `--batch_size` の組み合わせで回します。
- チェックポイントは `outputs/train/<run>/checkpoints/<step>/pretrained_model/` に保存され、最新は `last/` シンボリックリンクから参照できます。
- 途中で止まったら `--resume=true` で再開できます。`--policy.push_to_hub=true` はポリシーの動作確認が済んでからにしてください。

## 次回

第 5 回(学習済みモデルを実機で動かす = 評価)は別記事として公開予定です。先取りすると、`lerobot-record` には `--policy.path=...` フラグがあり、ここに先ほどの `pretrained_model/` フォルダのパスを渡すことで推論モードで動かせます。

## 参考リンク

- [LeRobot リポジトリ](https://github.com/huggingface/lerobot)
- [LeRobot scripts ディレクトリ](https://github.com/huggingface/lerobot/tree/main/src/lerobot/scripts) — `lerobot_train.py`
- [LeRobot ポリシー一覧](https://huggingface.co/docs/lerobot/policies) — ACT、Diffusion Policy、Pi0、SmolVLA
- [Weights & Biases ドキュメント](https://docs.wandb.ai/)
