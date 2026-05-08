---
title: "LeRobot のインストールとアームのキャリブレーション"
emoji: "🤖"
type: "tech"
topics: ["lerobot", "robotics", "huggingface", "python", "setup"]
published: false
---

> **シリーズ:** LeRobot チュートリアル — 第 1 回 / 全 5 回
> 前回: [LeRobot Primer(第 0 回)](./lerobot-primer-and-tips) ・ 次回: [Teleop で動かす(第 2 回)](./lerobot-teleop-control)

## はじめに

この記事では、LeRobot をクリーンな Python 環境にインストールし、SO-101 アームを 2 本(**leader** と **follower**)キャリブレーションするところまで進めます。シリーズ後半の teleop・データ録画・学習・推論はすべてこのセットアップを前提にします。

キャリブレーションのやり方は **2 通り** 紹介します。明示的に `lerobot-calibrate` を実行する方法と、teleop コマンドが必要なときに勝手にプロンプトを出してくれる方法です。

環境バージョンや用語、キャリブレーションファイルの保存場所などは [Primer(第 0 回)](./lerobot-primer-and-tips) にまとめてあります。

## ゴール

この記事を読み終えると、以下ができるようになります。

- LeRobot を専用の virtualenv にインストールできる
- どの USB ポートがどちらのアームか判別できる
- アームごとにキャリブレーションファイルを `~/.cache/huggingface/lerobot/calibration/` に保存できる
- 「明示的キャリブレーション」と「teleop による再キャリブレーション」の使い分けがわかる

## 前提

- SO-101 アーム 2 本(leader / follower)を USB で接続済み
- macOS または Ubuntu
- Python 3.10 以上 と [`uv`](https://docs.astral.sh/uv/) がインストール済み
- [Primer(第 0 回)](./lerobot-primer-and-tips) に一度目を通していること

## 環境

| コンポーネント | バージョン |
|----------------|------------|
| `lerobot`      | `<commit or tag>` |
| Python         | `3.10` 以上 |
| OS             | macOS 14 / Ubuntu 22.04 |
| ハードウェア   | SO-101 × 2(leader + follower) |

## 手順

### 1. LeRobot をインストール

`lerobot/.venv` に専用の venv を作って、その中にだけ LeRobot を入れます。Primer で挙げた「Python 環境の混在」という最頻トラブルを避けるためです。

```bash:terminal
mkdir -p lerobot && cd lerobot
uv venv .venv
source .venv/bin/activate
uv pip install lerobot
```

CLI が PATH に通っていることを確認します。

```bash:terminal
lerobot --help
lerobot-calibrate --help
```

### 2. アームを接続して USB ポートを確認

両方のアームを USB で接続し、シリアルデバイスをリストします。

```bash:terminal
# macOS
ls /dev/tty.usbmodem*

# Linux
ls /dev/ttyACM* /dev/ttyUSB*
```

2 つのポートが表示されるはずです。どちらがどちらのアームかを判別するには、片方のアームを抜いて再度実行します。消えたほうのポートが、抜いたアームに対応します。

この記事では以下の対応で進めます。

- **Leader アーム** → `/dev/ttyACM0`
- **Follower アーム** → `/dev/ttyACM1`

ご自分の環境に合わせて以下のコマンドを読み替えてください。

### 3. キャリブレーション — Option A: 明示的に事前実行

キャリブレーションは、各モーターのエンコーダ生値を物理的な関節角度にマッピングする処理です。アームごとに 1 回(モーターのファームを書き換えたら再度)実行します。

長いフラグをシェル履歴に残さないために、ラッパースクリプトを用意しておくと便利です。

```bash:scripts/calibrate.sh
#!/bin/bash
source "$(dirname "$0")/../lerobot/.venv/bin/activate"

if [ "$1" = "--follower" ] || [ "$1" = "-c" ]; then
    lerobot-calibrate \
        --robot.type=so101_follower \
        --robot.port=/dev/ttyACM1 \
        --robot.id=raha_follower_arm
else
    lerobot-calibrate \
        --robot.type=so101_follower \
        --robot.port=/dev/ttyACM0 \
        --robot.id=raha_leader_arm
fi
```

実行権限を付けて、アームごとに 1 回ずつ実行します。

```bash:terminal
chmod +x scripts/calibrate.sh
./scripts/calibrate.sh                # leader  (/dev/ttyACM0)
./scripts/calibrate.sh --follower     # follower (/dev/ttyACM1)
```

画面の指示にしたがって各関節を可動域全体に動かし、限界位置で Enter を押します。完了するとキャリブレーションファイルが以下の場所に保存されます。

```
~/.cache/huggingface/lerobot/calibration/<robot.id>.json
```

:::message
**Tip:** `--robot.id` は物理的なアームごとに必ず一意な名前を付けてください(本記事では `raha_leader_arm` と `raha_follower_arm`)。後続のすべての LeRobot コマンドはこの id をキーにキャリブレーションを引きます。teleop や record で別の id を指定すると、せっかく作ったファイルが見つからずに毎回再キャリブレーションを要求されます。
:::

### 4. キャリブレーション — Option B: teleop に任せる

実は Option A を先にやらなくても大丈夫です。teleop コマンドはキャリブレーションが見つからない、もしくはモーター側の値とズレているときに、その場でキャリブレーションを実行するか聞いてくれます。

```
./scripts/teleoperate.sh
INFO 2026-05-08 13:27:37 eoperate.py:211 {'display_compressed_images': False,
 'display_data': False,
 'display_ip': None,
 'display_port': None,
 'fps': 60,
 'robot': {'calibration_dir': None,
           'cameras': {},
           'disable_torque_on_disconnect': True,
           'id': 'raha_follower_arm',
           'max_relative_target': None,
           'port': '/dev/ttyACM1',
           'use_degrees': True},
 'teleop': {'calibration_dir': None,
            'id': 'raha_leader_arm',
            'port': '/dev/ttyACM0',
            'use_degrees': True},
 'teleop_time_s': None}
INFO 2026-05-08 13:27:37 so_leader.py:72 Mismatch between calibration values in the motor and the calibration file or no calibration file found
Press ENTER to use provided calibration file associated with the id raha_leader_arm, or type 'c' and press ENTER to run calibration:
```

ここで `c` + Enter を押せば、teleop の起動シーケンスのなかでキャリブレーションが走ります。試行錯誤しながら触る段階では便利ですが、スクリプトに組み込みづらいので、初回セットアップは Option A を推奨します。

### 5. 動作確認

作成されたキャリブレーションファイルをリストします。

```bash:terminal
ls ~/.cache/huggingface/lerobot/calibration/
```

`--robot.id` ごとに JSON ファイルが 1 つずつ存在していれば成功です。両方のアームのファイルが揃ったら、[Teleop で動かす(第 2 回)](./lerobot-teleop-control) に進めます。

## つまずきポイント

:::message
**エラー**: `Could not open port /dev/ttyACM0`
**原因**: OS がアクセスを拒否している、または他のプロセスがポートを掴んでいる。
**対処(Linux)**: `sudo usermod -aG dialout $USER` で `dialout` グループに追加し、ログインし直す。
**対処(macOS)**: USB を抜き差しする。「システム設定 → プライバシーとセキュリティ」でブロックされたドライバがないか確認する。
:::

:::message
**症状**: `Connected to N motors` の N が想定より少ない
**原因**: デイジーチェーンか USB ケーブルの接触不良、もしくはモーターの電源が入っていない。
**対処**: モーター間ケーブルを挿し直し、アームの電源 LED を確認する。
:::

:::message alert
**毎回キャリブレーションを促される**: teleop で指定している `--robot.id` が、キャリブレーション時の id と一致しない可能性が高いです。両方のコマンドで id が完全一致しているか必ず確認してください。
:::

## まとめ

- `uv pip install lerobot` で `lerobot/.venv` に LeRobot をインストールしました
- 2 本のアームをそれぞれ USB ポート(本記事では leader が `/dev/ttyACM0`、follower が `/dev/ttyACM1`)で識別しました
- ラッパースクリプトで明示的にキャリブレーション(Option A)するか、teleop に任せる(Option B)を選べます
- キャリブレーションファイルは `~/.cache/huggingface/lerobot/calibration/<robot.id>.json` に保存されます

## 次回

次は [Teleop で動かす(第 2 回)](./lerobot-teleop-control) に進みます。leader で follower を動かし、ここまでのセットアップが正しいか実際の動きで確認します。

## 参考リンク

- [LeRobot リポジトリ](https://github.com/huggingface/lerobot)
- [SO-100 / SO-101 ハードウェアセットアップ(TheRobotStudio)](https://github.com/TheRobotStudio/SO-ARM100)
- シリーズ: [Primer(第 0 回)](./lerobot-primer-and-tips)
