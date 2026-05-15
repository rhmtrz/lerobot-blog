---
title: "LeRobot — カメラ設定とエピソードデータセットの録画"
emoji: "📝"
type: "tech"
topics: ["lerobot", "robotics", "dataset", "huggingface", "ai"]
published: true
publication_name: "adawarp"
---

> **シリーズ:** LeRobot チュートリアル — 第 3 回 / 全 5 回
> 前回: [キャリブレーションとテレオペレーション(第 2 回)](./lerobot-calibration-and-teleoperate) ・ 次回: [ポリシーのトレーニング(第 4 回)](./lerobot-train-policy)

## はじめに

この記事では、「テレオペでアームを動かせるようになった」状態から、「トレーニングに使えるデータセットができた」状態までを一気通貫で扱います。具体的には、カメラを接続して映像を確認し、`lerobot-record` に渡すカメラ設定を組み立て、エピソードを録画して保存するまでです。ここで作るデータセットは、[第 4 回(ポリシーのトレーニング)](./lerobot-train-policy) でそのまま入力として使います。

[第 2 回](./lerobot-calibration-and-teleoperate) のキャリブレーションは完了済みで、leader が `/dev/ttyACM0`、follower が `/dev/ttyACM1`、キャリブレーションファイルは `~/.cache/huggingface/lerobot/calibration/` 以下にある、という前提で進めます。

## ゴール

この記事を読み終えると、以下ができるようになります。

- USB 接続されている各カメラがどの位置(正面・手首など)に対応しているかを判別する
- 録画前にカメラの解像度・FPS・ピクセルフォーマットを確認する
- `lerobot-record` に `--robot.cameras` を正しく渡す
- leader テレオペで N 個のエピソードを録画し、`~/.cache/huggingface/lerobot/<repo_id>/` に保存する
- (オプション)Hugging Face Hub にデータセットを push する

## 前提

- Ubuntu 22.04(本記事は Linux 環境のみを対象にします)
- SO-101 アーム 2 本のキャリブレーションが完了し、テレオペで動作確認済み([第 2 回](./lerobot-calibration-and-teleoperate))
- USB カメラを 1 台以上。follower の手首にもう 1 台付けるのが強く推奨です
- Hub に push する場合は Hugging Face のアカウント(`huggingface-cli login` を一度実行済み)

## 手順

### 1. カメラを接続して `/dev/video*` を確認する

Linux ではカメラは `/dev/video*` として見えます。物理カメラ 1 台につき **2 つ** のデバイスノードが作られるのが普通で(映像ストリーム用とメタデータ用)、実際の映像が流れるのは片方だけです。

```bash:terminal
ls /dev/video*
```

どのノードがどの物理カメラかを特定するには、片方を抜いてもう一度実行します。消えたノードが抜いたカメラに対応します。1 台のカメラに対して複数ノードが見える場合は、最も小さい番号のノードがストリーム用であることが多いです。

:::message alert
カメラのインデックスは再起動・USB の抜き差し・別ポートへの差し替えで **変わります**。「正面カメラが急に手首映像に見える」原因はほぼこれです。安定運用したいなら、毎回同じ USB ポートに挿すか、`udev` ルールでシリアル番号にひもづけて固定してください。
:::

### 2. `lerobot-find-cameras` でカメラを検出する

LeRobot には接続中のカメラをすべてプローブして、各デバイスからプレビュー JPEG を 1 枚ずつ書き出すヘルパーが用意されています。「どの `/dev/videoN` がどの位置のカメラか」を確認する一番手軽な方法です。

```bash:terminal
lerobot-find-cameras opencv
```

出力例(環境によって変わります):

```text
--- Detected Cameras ---
Camera #0:
  Name: OpenCV Camera @ /dev/video0
  Type: OpenCV
  Id: /dev/video0
  Backend api: V4L2
  Default stream profile:
    Format: 0.0
    Fourcc: YUYV
    Width: 640
    Height: 480
    Fps: 30.0
--------------------
Camera #1:
  Name: OpenCV Camera @ /dev/video2
  Type: OpenCV
  Id: /dev/video2
  Backend api: V4L2
  Default stream profile:
    Format: 0.0
    Fourcc: YUYV
    Width: 640
    Height: 480
    Fps: 30.0
--------------------

Finalizing image saving...
Image capture finished. Images saved to outputs/captured_images
```

`outputs/captured_images/` の中の JPEG を順番に開いて、どのカメラを **front**(作業空間全体を映す側)、どれを **wrist**(follower の手首に取り付ける側)として使うか決めます。

本記事では以下の対応で進めます。

- **front** → `/dev/video0`
- **wrist** → `/dev/video2`

### 3. 解像度と FPS を決める

デフォルトの 640x480 / 30 fps / YUYV は、ACT や Diffusion Policy のレシピが想定している組み合わせとほぼ一致するので、最初はこの設定で十分です。変更する前に知っておくと良い点:

- SO-101 のマニピュレーションタスクでは解像度を上げてもポリシーの学習はほとんど改善せず、それより先に USB 2.0 の帯域が飽和します。1080p MJPEG を 1 本流すだけでも、同じバスにある 2 台目のカメラがコマ落ちすることがあります。
- `--dataset.fps` は実際にカメラが出せる値と一致している必要があります。30 fps を指定したのにカメラが 15 fps しか出せない場合、フレームがドロップするか、ループが止まります。
- カメラがネイティブで対応している YUYV / MJPEG のいずれかにしてください。ソフトウェアでフォーマット変換するとフレームが落ちます。

OS レベルでカメラの能力を確認したいときは `v4l2-ctl` が便利です。

```bash:terminal
v4l2-ctl --device=/dev/video0 --list-formats-ext
```

### 4. record 用のラッパースクリプトを書く

`lerobot-record` のフラグはすぐ長くなります(カメラ・データセットのメタデータ・両方のアーム、それぞれに専用のフラグがある)。一度スクリプトにまとめて、Git で管理するのがおすすめです。

```bash:scripts/record.sh
#!/bin/bash
set -e
source "$(dirname "$0")/../lerobot/.venv/bin/activate"

HF_USER=${HF_USER:-$(whoami)}
DATASET_NAME=${DATASET_NAME:-so101_pick_place}
NUM_EPISODES=${NUM_EPISODES:-50}

lerobot-record \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=my_follower_arm \
    --robot.cameras="{ 
      front: {type: opencv, index_or_path: /dev/video0, width: 640, height: 480, fps: 30}, 
      wrist: {type: opencv, index_or_path: /dev/video2, width: 640, height: 480, fps: 30} 
      }" \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0 \
    --teleop.id=my_leader_arm \
    --display_data=true \
    --dataset.repo_id="${HF_USER}/${DATASET_NAME}" \
    --dataset.single_task="Pick up the cube and place it in the box" \
    --dataset.num_episodes=${NUM_EPISODES} \
    --dataset.episode_time_s=12 \
    --dataset.reset_time_s=3 \
    --dataset.fps=30 \
    --dataset.push_to_hub=false
```

各ブロックの意味:

- **`--robot.*`** — follower(実際にタスクを実行する側)の設定。カメラはここに紐付けます。観測(observation)は follower の視点から取得するためです。
- **`--robot.cameras=...`** — `{name: {type, index_or_path, width, height, fps}}` の辞書です。名前(`front`、`wrist`)はそのままデータセットのカラム名になります。後から見返しやすい名前を付け、同じタスクの録画では同じ名前で統一してください。
- **`--teleop.*`** — leader 側の設定。録画(と teleop の動作確認)でしか使いません。推論時にはこのフラグは丸ごと外します。
- **`--display_data=true`** — 録画中にカメラ映像とモーター状態をライブ表示する Rerun ビューアを開きます。「wrist カメラがワークスペースじゃなく自分の手を映していた」など、50 エピソード無駄にする前に気付くための強力なツールです。
- **`--dataset.repo_id`** — Hugging Face 形式の `user/name` スラッグ。ローカルには `~/.cache/huggingface/lerobot/<repo_id>/` に保存されます。
- **`--dataset.single_task`** — タスクの自然言語による説明。ACT や言語条件付きポリシーはこれをデータセットから読みます。
- **`--dataset.episode_time_s`** / **`reset_time_s`** — プライマーの用語集を参照。1 エピソードの録画時間と、エピソード間の環境リセット時間です。
- **`--dataset.push_to_hub=false`** — データセットを目視で確認するまでは false のままにしておきます。

### 5. エピソードを録画する

スクリプトに実行権限を付けて録画を開始します。

```bash:terminal
chmod +x scripts/record.sh
./scripts/record.sh
```

CLI が `Recording episode 0` と表示し、Rerun ウィンドウが開きます。leader アームを動かしてタスクを実演してください。録画中のキー操作:

| キー | 動作 |
|------|------|
| `→`(右矢印) | 現在のエピソードを途中で確定して保存。すぐにリセット時間が始まります。 |
| `←`(左矢印) | 現在のエピソードを破棄して録り直し。 |
| `Esc` | セッション全体を停止。すでに保存済みのエピソードは残ります。 |

エピソードとエピソードの間は `--dataset.reset_time_s` 秒だけ `Reset the environment` が表示されます。キューブを元に戻し、箱を動かして初期状態にしたら、次のエピソードが自動的に始まります。

:::message
`episode_time_s` をいっぱいまで使う必要はありません。8 秒でタスクが終わったら `→` で次に進んでください。エピソードの最後を無動作で埋めると、モデルは「最後は何もしない」というパターンを覚えてしまい、ポリシーの性能を下げます。
:::

### 6. トレーニング前にデータセットを確認する

データセットは録画と同時に以下のパスに書き出されています。

```
~/.cache/huggingface/lerobot/<repo_id>/
```

録画が完了したら(または途中で `Esc` で抜けたら)、ビジュアライザで再生して内容を確認します。中身が悪いデータセットはトレーニングの数時間を無駄にするので、ここで数分かけて見ておく価値があります。

```bash:terminal
python -m lerobot.scripts.visualize_dataset \
    --repo-id "${HF_USER}/so101_pick_place" \
    --episode-index 0
```

いくつかのエピソードを横断して見ると良いポイント:

- 両方のカメラがワークスペースを問題なく映せているか(視界が遮られていないか、オートフォーカス・露出がチカチカしていないか)
- follower が実際にタスクを完遂できているか(意外と漏れます)
- エピソード長がほぼ揃っているか。バラバラなら、リセット手順が一貫していない可能性があります
- カメラ映像にフレームドロップが出ていないか

### 7. (オプション)Hub に push する

データセットが良さそうなら Hub に上げます。

```bash:terminal
huggingface-cli login    # 初回のみ
```

そのうえで、次回以降の `record.sh` に `--dataset.push_to_hub=true` を付けて再録画するか、既存のフォルダを `huggingface-cli upload` で直接アップロードします。push 後は別マシンからも `--dataset.repo_id="${HF_USER}/${DATASET_NAME}"` で読み込めるので、第 4 回のトレーニングはそのまま走らせられます。

## つまずきポイント

:::message
**エラー**: `Could not open camera /dev/video0`
**原因**: 他のプロセスがカメラデバイスを掴んでいる、もしくはストリームノードではなくメタデータノード(`+1` 番)を指定している。
**対処**: `fuser /dev/video0` で掴んでいるプロセスを終了し、次の `/dev/video*` 番号も試してみてください。
:::

:::message
**症状**: 録画は始まるが、Rerun のカメラ枠の片方が真っ黒のまま
**原因**: USB の帯域が飽和しているか、指定した解像度ではその FPS を出せない。
**対処**: 2 つのカメラを別の USB コントローラに分ける(同じハブの別ポートではダメ)か、解像度・FPS を下げる。
:::

:::message
**症状**: エピソードファイルは保存されるのに、`visualize_dataset` が shape mismatch で落ちる
**原因**: 録画の途中で `--robot.cameras` を変更した(カメラを 1 台外した、など)。データセットのスキーマは最初のエピソードを書いた時点で固定されます。
**対処**: 新しい `--dataset.repo_id` で録り直すか、`~/.cache/huggingface/lerobot/` 以下の該当フォルダを削除して綺麗な状態から録り直してください。
:::

:::message
**症状**: leader の動きが重く、follower がガクガクと低 FPS で追従する
**原因**: 表示ウィンドウと複数カメラのデコードが制御ループと CPU を食い合っている。
**対処**: 最初の数エピソードを録ってセットアップに自信が持てたら、`--display_data=false` で録画してみてください。明らかに滑らかになります。
:::

## まとめ

- Linux ではカメラは `/dev/video*` に出てきます。物理カメラとの対応は `lerobot-find-cameras opencv` のプレビュー画像で確認するのが早いです。
- `--robot.cameras='{ ... }'` には名前付きのカメラ設定の辞書を渡します。名前はそのままデータセットの画像カラム名になります。
- 録画は leader テレオペで行います。`→` で確定、`←` で録り直し、`Esc` で停止です。
- データセットは `~/.cache/huggingface/lerobot/<repo_id>/` に保存されます。トレーニングの前に必ず `visualize_dataset` で再生して確認してください。

## 次回

次は [ポリシーのトレーニング(第 4 回)](./lerobot-train-policy) に進みます。ここで作ったデータセットを `lerobot-train` に流し込んで、ロボット上で実行できるチェックポイントを作ります。

## 参考リンク

- [LeRobot リポジトリ](https://github.com/huggingface/lerobot)
- [LeRobot scripts ディレクトリ](https://github.com/huggingface/lerobot/tree/main/src/lerobot/scripts) — `lerobot_record.py`、`find_cameras.py`、`visualize_dataset.py`
- [LeRobotDataset v3 フォーマット](https://huggingface.co/docs/lerobot/lerobot-dataset-v3)
- [Rerun ビューア](https://www.rerun.io/) — `--display_data=true` で使われます
