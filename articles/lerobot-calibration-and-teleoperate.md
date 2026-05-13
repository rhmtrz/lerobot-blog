---
title: "LeRobot SO-101 アームのキャリブレーションとテレオペレーション"
emoji: "🤖"
type: "tech"
topics: ["lerobot", "robotics", "huggingface", "python", "so101"]
published: false
---

## はじめに

この記事では、SO-101 アームを LeRobot で動かすための **キャリブレーション** と **テレオペレーション(teleop)** の手順を、はじめて触る人向けにまとめます。LeRobot 本体のインストールはすでに済んでいる前提です(`lerobot --help` が通る状態)。

キャリブレーションのやり方は **2 通り** 紹介します。`lerobot-calibrate` を明示的に実行する方法と、teleop コマンドのなかで必要なときにプロンプトを出してくれる方法です。

## Leader と Follower について

最初に紛らわしいポイントを整理します。LeRobot のテレオペは **leader(リーダー)** と **follower(フォロワー)** という 2 本のアームを使う構成です。

- **Leader アーム**: 人間が手でつかんで動かす側。サーボは位置を読み取るモードで動き、関節角度を follower に送ります。
- **Follower アーム**: 受信した関節位置に追従して実際にタスクを実行する側。「ロボット本体」として扱われます。

SO-101 は [TheRobotStudio](https://github.com/TheRobotStudio/SO-ARM100) が公開しているオープンソースのアームで、leader 構成と follower 構成のキットがあります。LeRobot CLI ではそれぞれを `so101_leader` / `so101_follower` として指定します。

## ゴール

- 各 USB ポートがどちらのアームか判別できる
- アームごとにキャリブレーションファイルを `~/.cache/huggingface/lerobot/calibration/` に保存できる
- 「明示的キャリブレーション」と「teleop による再キャリブレーション」の使い分けがわかる
- `lerobot-teleoperate` で leader を動かして follower を追従させられる

## 前提

- Ubuntu 22.04(本記事は Linux 環境のみを対象にします)
- LeRobot がインストール済みで、`lerobot --help` / `lerobot-calibrate --help` / `lerobot-teleoperate --help` が実行できる
- SO-101 アームを 2 本(leader + follower)用意し、USB で接続済み

## 手順

### 1. アームを接続して USB ポートを確認

両方のアームを USB で接続し、シリアルデバイスをリストします。

```bash:terminal
ls /dev/ttyACM* /dev/ttyUSB*
```

2 つのポートが表示されるはずです。どちらが leader でどちらが follower かを判別するには、片方のアームを抜いてもう一度実行します。消えたほうのポートが、抜いたアームに対応します。

この記事では以下の対応で進めます。

- **Leader アーム** → `/dev/ttyACM0`
- **Follower アーム** → `/dev/ttyACM1`

自分の環境に合わせて以降のコマンドを読み替えてください。

:::message
ポートを開けない(`Could not open port` などのエラー)場合は、ユーザーが `dialout` グループに入っていない可能性があります。`sudo usermod -aG dialout $USER` でグループに追加し、ログインし直してから再実行してください。
:::

### 2. キャリブレーション — Option A: 明示的に事前実行

キャリブレーションは、各モーターのエンコーダ生値を物理的な関節角度にマッピングする処理です。アームごとに 1 回(モーターのファームを書き換えたら再度)実行します。

長いフラグをシェル履歴に残さないように、ラッパースクリプトを用意しておくと便利です。

```bash:scripts/calibrate.sh
#!/bin/bash
source "$(dirname "$0")/../lerobot/.venv/bin/activate"

if [ "$1" = "--follower" ] || [ "$1" = "-c" ]; then
    lerobot-calibrate \
        --robot.type=so101_follower \
        --robot.port=/dev/ttyACM1 \
        --robot.id=my_follower_arm
else
    lerobot-calibrate \
        --robot.type=so101_leader \
        --robot.port=/dev/ttyACM0 \
        --robot.id=my_leader_arm
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
**Tip:** `--robot.id` は物理的なアームごとに必ず一意な名前を付けてください(本記事では `my_leader_arm` と `my_follower_arm`)。後続のすべての LeRobot コマンドはこの id をキーにキャリブレーションを引きます。teleop や record で別の id を指定すると、せっかく作ったファイルが見つからずに毎回再キャリブレーションを要求されます。
:::

### 3. キャリブレーション — Option B: teleop に任せる

実は Option A を先にやらなくても大丈夫です。teleop コマンドはキャリブレーションが見つからない、もしくはモーター側の値とズレているときに、その場でキャリブレーションを実行するかを聞いてくれます。

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
           'id': 'my_follower_arm',
           'max_relative_target': None,
           'port': '/dev/ttyACM1',
           'use_degrees': True},
 'teleop': {'calibration_dir': None,
            'id': 'my_leader_arm',
            'port': '/dev/ttyACM0',
            'use_degrees': True},
 'teleop_time_s': None}
INFO 2026-05-08 13:27:37 so_leader.py:72 Mismatch between calibration values in the motor and the calibration file or no calibration file found
Press ENTER to use provided calibration file associated with the id my_leader_arm, or type 'c' and press ENTER to run calibration:
```

ここで `c` + Enter を押すと、teleop の起動シーケンスのなかでキャリブレーションが走ります。試行錯誤しながら触る段階では便利ですが、スクリプトに組み込みづらいので、初回セットアップは Option A をおすすめします。

### 4. テレオペレーション

leader と follower のキャリブレーションが揃ったら、いよいよ teleop を実行します。leader を手で動かすと、follower がそれにリアルタイムで追従するはずです。

```bash:scripts/teleoperate.sh
#!/bin/bash
source "$(dirname "$0")/../lerobot/.venv/bin/activate"

lerobot-teleoperate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=my_follower_arm \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0 \
    --teleop.id=my_leader_arm
```

```bash:terminal
chmod +x scripts/teleoperate.sh
./scripts/teleoperate.sh
```

実行すると LeRobot がアームの接続とキャリブレーションを確認し、teleop のループに入ります。停止するときは `Ctrl + C` です。

動作確認のチェック項目:

- leader をゆっくり動かしたとき follower が遅延なく追従するか
- leader のグリッパーレバーを握ると follower のグリッパーも閉じるか
- 関節の可動域端で「カクッ」と止まる違和感がないか(キャリブレーションがズレていると可動域が狂います)

## 参考: 関連スクリプトのソース

CLI の `lerobot-calibrate` と `lerobot-teleoperate` は、それぞれリポジトリの以下のスクリプトから来ています。フラグの意味を正確に知りたい、挙動をカスタマイズしたい、というときに読むと早いです。

- GitHub: <https://github.com/huggingface/lerobot/tree/main/src/lerobot/scripts>
  - `lerobot_calibrate.py`
  - `lerobot_teleoperate.py`
- ローカル(LeRobot をソースから入れている場合): `/home/$USER/lerobot/src/lerobot/scripts/lerobot_calibrate.py` / `lerobot_teleoperate.py`

CLI の各フラグは `lerobot-calibrate --help` / `lerobot-teleoperate --help` でも確認できます。

## つまずきポイント

:::message
**エラー**: `Could not open port /dev/ttyACM0`
**原因**: OS がアクセスを拒否している、または他のプロセスがポートを掴んでいる。
**対処**: `sudo usermod -aG dialout $USER` で `dialout` グループに追加し、ログインし直す。それでも直らない場合は USB を抜き差しし、別の teleop / record プロセスが残っていないか `ps` で確認する。
:::

:::message
**症状**: `Connected to N motors` の N が想定より少ない
**原因**: デイジーチェーンか USB ケーブルの接触不良、もしくはモーターの電源が入っていない。
**対処**: モーター間ケーブルを挿し直し、アームの電源 LED を確認する。
:::

:::message alert
**毎回キャリブレーションを促される**: teleop で指定している `--robot.id` / `--teleop.id` が、キャリブレーション時の id と一致しない可能性が高いです。両方のコマンドで id が完全一致しているか必ず確認してください。
:::

:::message
**teleop しても follower が動かない / 逆向きに動く**: leader と follower の `--port` が入れ替わっている可能性があります。片方のアームを抜き差しして、どちらのデバイスファイルが消えるかで確認し直してください。
:::

## まとめ

- `/dev/ttyACM*` を見て leader と follower のポートを判別しました(本記事では leader が `/dev/ttyACM0`、follower が `/dev/ttyACM1`)
- ラッパースクリプトで明示的にキャリブレーション(Option A)するか、teleop に任せる(Option B)を選べます
- キャリブレーションファイルは `~/.cache/huggingface/lerobot/calibration/<robot.id>.json` に保存されます
- `lerobot-teleoperate` で leader を動かすと follower がリアルタイム追従します
- SO-101 の leader と follower はそれぞれ `--robot.type=so101_leader` / `so101_follower` で指定します

## 参考リンク

- [LeRobot リポジトリ](https://github.com/huggingface/lerobot)
- [LeRobot scripts ディレクトリ](https://github.com/huggingface/lerobot/tree/main/src/lerobot/scripts)
- [SO-101 ハードウェアセットアップ(TheRobotStudio)](https://github.com/TheRobotStudio/SO-ARM100)
