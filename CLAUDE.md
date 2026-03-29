# TuxGuitar 開発メモ

## ピッキング装置連携機能（改造目標）

TAB譜のUP/DOWNピッキング情報をMIDI CCとして出力し、ギター自動ピッキング装置を制御する。

### 対象装置

- ギター1本につき1台接続。全弦に1個ずつピックを持ち、UP/DOWNの2ポジションで弦をピッキングする。
- 演奏前に人間が全ピックを手動で初期位置にセット。初期化処理はソフトウェア側では行わない。

### TuxGuitar上の操作

- 既存の **Pick upstroke / Pick downstroke**（`TGPickStroke`）を使用する（Brush stroke＝`TGStroke`とは別）

### MIDI CC仕様

| 弦番号（TAB譜準拠） | CC番号 |
|-------------------|--------|
| 1弦 | CC 102 |
| 2弦 | CC 103 |
| 3弦 | CC 104 |
| 4弦 | CC 105 |
| 5弦 | CC 106 |
| 6弦 | CC 107 |

- DOWN = 0、UP = 127
- チャンネル = ギタートラックと同じMIDIチャンネル
- CC 102〜119はMIDI規格の未定義領域（自由使用可）

### CC送信ルール

| 条件 | タイミング | 方向 | 対象弦 |
|------|-----------|------|--------|
| TGPickStrokeのみ | ビート開始時刻 | pickStroke方向 | 音符がある弦のみ |
| TGStrokeのみ | 各弦のノートON時刻（strokeオフセット後） | stroke方向 | 音符がある弦のみ |
| 両方 | 各弦のノートON時刻（strokeオフセット後） | TGPickStroke優先 | 音符がある弦のみ |
| どちらもなし | 送信しない | — | — |

### 変更ファイル

- `common/TuxGuitar-lib/src/main/java/app/tuxguitar/player/base/MidiControllers.java` — CC番号定数追加
- `common/TuxGuitar-lib/src/main/java/app/tuxguitar/player/base/MidiSequenceParser.java` — `addNotes()` にCC送信ロジック追加

### 実装済み（動作確認済み）

**`MidiControllers.java`** に追加した定数：
```java
// Picking device: CC per string (CC 102-107 = strings 1-6)
// Value: 0 = DOWN, 127 = UP
public static final int PICKING_STRING_BASE = 0x66; // 102
```

**`MidiSequenceParser.java`** の `addNotes()` に追加したロジック：
- `TGPickStroke` または `TGStroke` が設定されているビートで、対象弦のCC送信
- パーカッションチャンネルはスキップ
- `TGPickStroke` が優先、なければ `TGStroke` の方向を使用
- `start`（strokeオフセット適用済み）のタイミングで送信するため、ブラッシュストロークの弦ごとのタイミングずれも自動で反映される

**注意事項：**
- TuxGuitarは1トラックに2つのMIDIチャンネルを使用する（通常ノート用ch + ベンドモード用ch）
- そのためすべてのCCは2チャンネル分出力される（仕様通り、バグではない）
- 装置側はギタートラックのメインチャンネル（ch0など）のみ受信すればよい
- ピッキング記号の入力は `Beat → Pick upstroke / Pick downstroke`（波線付きのBrush strokeとは別）

---

## 情報源

- **使い方・仕様のリファレンス**: `website/files/2.0.1/desktop/help/` 以下のHTMLファイル（機能仕様・UI・操作方法を調べる際はまずここを参照）

## 環境

- OS: Ubuntu 24.04
- Java: OpenJDK 21
- Maven: 3.8.7
- ターゲット: Windows x86_64（Ubuntu上でクロスコンパイル）

## インストール済みツール

```bash
sudo apt install wget unzip default-jdk maven gcc-mingw-w64-x86-64 g++-mingw-w64-i686-win32 build-essential
```

## Linux向けビルド用コンポーネント（開発終了後に削除可）

### apt パッケージ

```bash
sudo apt install libfluidsynth-dev libjack-jackd2-dev libasound2-dev liblilv-dev libsuil-dev qtbase5-dev
```

削除する場合：
```bash
sudo apt remove libfluidsynth-dev libjack-jackd2-dev libasound2-dev liblilv-dev libsuil-dev qtbase5-dev
sudo apt autoremove
```

### Eclipse SWT（Linux版）のセットアップ

SWT 4.37 (GTK Linux x86_64) をダウンロードしてMavenローカルリポジトリに登録済み。
ファイルは `~/swt-4.37-gtk-linux-x86_64/swt.jar`。
削除する場合：
```bash
rm -rf ~/swt-4.37-gtk-linux-x86_64.zip ~/swt-4.37-gtk-linux-x86_64
rm -rf ~/.m2/repository/org/eclipse/swt/org.eclipse.swt.gtk.linux
```

再セットアップが必要な場合：
```bash
cd ~
wget https://download.eclipse.org/eclipse/downloads/drops4/R-4.37-202509050730/swt-4.37-gtk-linux-x86_64.zip
mkdir swt-4.37-gtk-linux-x86_64
cd swt-4.37-gtk-linux-x86_64
unzip ../swt-4.37-gtk-linux-x86_64.zip
mvn install:install-file -Dfile=swt.jar -DgroupId=org.eclipse.swt -DartifactId=org.eclipse.swt.gtk.linux -Dpackaging=jar -Dversion=4.37
```

### Linux向けビルド

```bash
cd desktop/build-scripts/tuxguitar-linux-swt
mvn -e clean verify -P native-modules
```

成果物: `desktop/build-scripts/tuxguitar-linux-swt/target/tuxguitar-9.99-SNAPSHOT-linux-swt/`

起動:
```bash
cd desktop/build-scripts/tuxguitar-linux-swt/target/tuxguitar-9.99-SNAPSHOT-linux-swt
bash tuxguitar.sh
```

絶対パスで貼り付けるだけのコマンド:
```bash
# ビルド（変更を反映する場合）
cd /home/kozue/dev/githubRepos/tuxguitar/desktop/build-scripts/tuxguitar-linux-swt && mvn -e clean verify -P native-modules

# 起動
bash /home/kozue/dev/githubRepos/tuxguitar/desktop/build-scripts/tuxguitar-linux-swt/target/tuxguitar-9.99-SNAPSHOT-linux-swt/tuxguitar.sh
```

## Eclipse SWT（Windows版）のセットアップ

SWT 4.37 をダウンロードしてMavenローカルリポジトリに登録済み。
再セットアップが必要な場合：

```bash
cd ~
wget https://download.eclipse.org/eclipse/downloads/drops4/R-4.37-202509050730/swt-4.37-win32-win32-x86_64.zip
mkdir swt-4.37-win32-win32-x86_64
cd swt-4.37-win32-win32-x86_64
unzip ../swt-4.37-win32-win32-x86_64.zip
mvn install:install-file -Dfile=swt.jar -DgroupId=org.eclipse.swt -DartifactId=org.eclipse.swt.win32.win32 -Dpackaging=jar -Dversion=4.37
```

## Windows向けビルド

```bash
cd desktop/build-scripts/tuxguitar-windows-swt-x86_64
mvn -e clean verify -P native-modules -P -platform-linux -P platform-windows
```

成果物: `desktop/build-scripts/tuxguitar-windows-swt-x86_64/target/tuxguitar-9.99-SNAPSHOT-windows-swt-x86_64/`

このフォルダをWindowsにコピーし、`jre` サブフォルダにJREを置けば `tuxguitar.exe` で起動できる。
（JREは https://portableapps.com/apps/utilities/OpenJDK64 から取得可能）

## サウンド設定（Linux）

追加インストール不要。`Tools → Settings → Sound` でMIDI Portを切り替えるだけ。

- **音を出す**: MIDI Port を "Gervill" に設定
- **MIDI監視**: MIDI Port を "Midi Through Port-0" に設定し、`aseqdump` で監視

## MIDI確認コマンド

```bash
# ポート/デバイス一覧
aplaymidi -l          # ALSA MIDIポート一覧（再生）
arecordmidi -l        # ALSA MIDIポート一覧（録音）
amidi -l              # MIDI ハードウェアデバイス一覧
cat /proc/asound/cards  # サウンドカード一覧

# リアルタイム監視（TuxGuitarのMIDI出力を確認する場合）
aseqdump -l           # ALSAシーケンサーのポート一覧（TuxGuitarのポート番号を確認）
aseqdump -p 128:0     # 指定ポートのMIDIイベントをリアルタイム表示（番号は要確認）
```

## git設定

```
user.name = kozue
user.email = maxon.onct@gmail.com
```
