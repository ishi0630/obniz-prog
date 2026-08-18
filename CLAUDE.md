# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

「オンラインボッチャ」（遠隔操作でボッチャのランプ＝ボール発射台を操作するWebアプリ）の**公開デモ用**リポジトリ。obniz（IoTマイコンボード）経由でステッピングモーター（高さ・左右位置）とサーボモーター（ボール投射）を制御する。

本体は単一HTMLファイル **`RAMP-DEMO.html`** で完結する（ビルド工程なし、依存パッケージのインストールも不要）。操作パネルの横に、たかさ・方向を示すゲージ状イラスト（`#illustStage`）を表示する機能を含む（`p=0`/`p=1`両対応。実装の経緯は`docs/ramp-illustration-poc.md`を参照）。**`RAMP-ILLUSTRATION-POC.html`もデモ用として同一内容で維持しており、2ファイルとも同じ内容を保つ運用**（詳細は「既知の残タスク」参照）。

## 動作確認・デプロイ

- ビルド/lint/testの仕組みは存在しない。`RAMP-DEMO.html` をそのままブラウザで開くか、GitHub Pagesで公開したURLにアクセスして動作確認する。
- 現在、GitHub Pagesは `claude/online-boccia-robot-lamp-grtof1` ブランチ（`/`ルート）から公開されている（`main`へは未マージ）。公開URL例: `https://ishi0630.github.io/obniz-prog/RAMP-DEMO.html`
- 動作確認は起動時URLパラメータで切り替える（詳細は次節）。デモとして開く場合は必ず `?d=1` を明示的に付与する（後述の通りデフォルトはd=0のまま変更しない方針）。
  - 例: `RAMP-DEMO.html?d=1&p=1`（デモ・△パネルUI、外部依存なしで最も手軽に確認できる）
  - 例: `RAMP-DEMO.html?d=1&p=0`（デモ・標準UI、外部CDN読み込みあり）

## URLパラメータ

| param | 意味 | 省略時 |
|---|---|---|
| `rn` | ランプ番号(1 or 2)。実機のobniz ID切り替え | `1` |
| `rs` | ランプサイズ(S or L)。ステップ数計算の係数切り替え | `S` |
| `lg` | 表示言語(J=日本語 / それ以外=フランス語)。大文字小文字区別なし | `J`（`d=1`時は指定によらず常にJ固定） |
| `d`  | デモモード(1=シミュレーション / 0=実機接続) | `0`（**このデフォルトは変更しない方針**。デモ配布時はURL側で`?d=1`を付ける運用） |
| `p`  | 操作パネル(1=△の大きいボタンパネルUI / 0=標準の小ボタンUI) | `0` |

## アーキテクチャ

### 2軸設計（最重要・今後も踏襲すること）

「モード軸」（`DEMO_MODE`：実機かデモか）と「表示軸」（`PANEL_MODE`：△パネルUIか標準UIか）は独立した別概念として扱う。

- `DEMO_MODE`の分岐は表示方式を問わず常に効く（予備ボタン非表示、EEPROM代替、複数同時接続対策、なげるギミックのディレイ等）
- `PANEL_MODE`の分岐はレイアウト・文言整形のみに使う
- 表示パターンが今後増えても、この2軸の独立性を崩さないこと

### 起動フロー（`main()` 内、`<script>`最後尾）

1. 二重タブガード（`sessionStorage`、失敗時は無視して継続）
2. URLパラメータ読み取り→ `DEMO_MODE`/`PANEL_MODE`/`LG`確定
3. `DEMO_MODE`なら`obniz.js`自体を読み込まない（実機通信ライブラリは一切ロードしない）。`!DEMO_MODE`の時だけCDNから動的読み込み
4. `PANEL_MODE`に応じてUIライブラリを選択
   - `p=1`: `installLightweightJQuery` / `installLightweightObnizUI` / `installLightweightAi` で自前の軽量実装（外部CDN依存ゼロ）
   - `p=0`: `installOriginalStandardUI` でBootstrap CSS・jQuery（obniz.io）・obniz-parts-kits一式（unpkg.com、howler/opencv/clmtrackr含む）を動的読み込み
5. `FakeObniz`クラス定義（`d=1`時のみ使用。EEPROM相当の値をメモリ上に保持し、`obniz.onconnect`を`setTimeout`で自動発火する疑似実装）
6. 言語別メッセージ文字列(`msg_*`/`btn_*`)を`LG`で出し分け
7. `ObnizUI.Button`/`ObnizUI.Label`のインスタンスをトップレベルで1回だけ生成（`onconnect`内で作ると再接続毎に増殖するため、意図的に外側に出してある）
8. `PANEL_MODE`時は△パネルの5ボタン(`#btnUp`等)から対応する標準ボタンの`.click()`を呼ぶ橋渡し処理、パネルに含まれない予備ボタンの非表示化、タイトル/DISPラベルのDOM移動、`fitPanelToWindow()`によるパネルサイズのJS計算（`MutationObserver`で実機接続時のDOM変化にも追従）を行う
9. `obniz.onconnect`内で本体のモーター制御・状態機械ループ（`while (LOOP_ACTIVE)`）を開始。`ONCONNECT_LOCK`で多重発火を防止

### 状態機械ループ

`obniz.onconnect`内の`while (LOOP_ACTIVE)`ループが、各`ObnizUI.Button`の`isClicked()`をポーリングして分岐する形で実装されている（`PIT`/`U`/`D`/`UU`/`DD`/`L`/`R`/`LL`/`RR`/`H100`/`H50`/`H0`）。各分岐は`ACTION_LOCK`で多重実行を防止し、`finally`で必ず解除する。

- **高さ制御**: `moveHeight(delta, isBig, playFreq)`（微調整）と`setHeightPreset(target, label, voice)`（いちばんうえ/した/まんなか直行）。ステップ数は`step_u`/`step_d`が`RS`（ランプサイズ）で係数を変える
- **左右制御**: `moveLR(delta, speedValue, playFreq)`。デモモードのみ移動演出用に500ms待つ（`DEMO_MODE`分岐の一例）
- **EEPROM**: `writeEEPROM`/`readEEPROM`（`ADDR_HEIGHT=0x0000`、`ADDR_DIRECTION=0x0010`）で高さ・左右位置を実機I2C EEPROMに保存・復元。**左右位置(`ADDR_DIRECTION`)は現状保存されているが、高さと違い起動時の初期表示ロジックが薄いので変更時は要確認**
- **表示整形**: `fmtIdle`/`fmtHeightMoving`/`fmtHeightArrived`/`fmtLRMoving`が`PANEL_MODE`で改行区切り複数行 or 標準UIの1行文字列を出し分ける
- **なげるギミック**: 全角スペースを使ったボール移動アニメーション。実機/デモ・パネル/標準UIの区別なく常に同じフルギミックを使う仕様（過去にデモ×パネル限定の簡略版があったが統一済み）。`PANEL_MODE`時は演出中だけ`.info`に`pit-anim`クラスを付け、パネル幅から逆算した`--pit-font`変数でフォントサイズを可変にする

### 複数同時接続対策（実機特有の既知バグ対策）

`obniz.onconnect`内の初期化処理では、`motor1.freeWait()`/`motor2.freeWait()`/`obniz.io11.output(false)`を**意図的に呼んでいない**（コメントに理由あり）。他セッションが動作中に接続してくると、これらの呼び出しがステップずれ・ロック誤解除を引き起こしていたため。ここを復活させないこと。

## 安全上・運用上の重要ルール（必ず守ること）

- **本リポジトリ（GitHub）は不特定多数がアクセスする公開デモ専用**。本番の実機制御コードはobniz社リポジトリ側（現時点、将来的にはAzureへ移行予定）で別途管理している。
- そのため `RAMP-DEMO.html` 内のobniz device IDは**ダミー値`"00000000"`で固定**している（`RN===1`/`RN===2`どちらの分岐も同一のダミー値）。**実機のdevice IDを絶対にこのリポジトリに含めないこと。**
- `d`パラメータのデフォルト（省略時=`"0"`＝実機モード）は**変更しない方針**。デモとして配布するURLには、コード側でなく**URL側に`?d=1`を明示して**運用する。
- `p=0`（標準UI）は現状、jQuery（obniz.io）・Bootstrap CSS（stackpath）・obniz-parts-kits一式（unpkg.com、opencv.js等の大きいファイル含む）を外部CDNから動的読み込みする仕様のまま。自リポジトリへの取り込み（vendoring）は未着手（将来検討事項。obniz社の「リポジトリ」機能は今年度末＝2027年3月末までに終了予定だが、obniz.io自体の静的ファイル配信やobnizサービス本体は継続する前提で現状維持としている）。

## 既知の残タスク

- `d=0`（実機モード）＋`p=1`/`p=0`それぞれの、実機・実ネットワーク環境での最終動作確認が未実施
- 左右位置のEEPROM保存・復元ロジックの妥当性再確認（高さほど手厚く検証されていない）
- なげるギミックの`pit-anim`オーバーレイの見た目を実機画面で最終確認
- `RS`（ランプサイズ S/L）パラメータの`p=1`（△パネルUI）での動作検証
- `p=0`用外部ライブラリのvendoring検討
- `main`ブランチへのマージ（現状`claude/online-boccia-robot-lamp-grtof1`ブランチのみで運用中）
- たかさ・方向のゲージ状イラスト表示は、`RAMP-ILLUSTRATION-POC.html`という別ファイルで開発していたが、`RAMP-DEMO.html`へ統合した（その後`RAMP-ILLUSTRATION-POC.html`もデモ用として使いたいとの要望があり、`RAMP-DEMO.html`と同一内容で復元。**現状は2ファイルとも同一内容を維持する運用**。片方を修正したらもう片方にも反映すること）。コードレビュー上は`d=0`実機モードでも(--ramp-colorの事前設定等)問題ないはずだが、**実機・実ネットワーク環境での動作確認は依然として未実施**（上記`d=0`確認タスクと同一）。詳細な経緯は`docs/ramp-illustration-poc.md`参照
  - 既知の懸念点1つ：ボール/針のアニメーション時間（ボール2秒・針0.6秒）は`DEMO_MODE`の疑似ディレイ（2000ms/500ms）に合わせて調整した値であり、実際のモーター動作時間（`RS`・実機の速度特性に依存）とは一致しない可能性がある。実機ではDISPの確定タイミング（`CF()`）より先/後にボールの見た目のアニメーションが終わることがあり得るが、値自体は正しく反映されるため機能上の不具合ではない。実機確認時に見た目が気になれば、CSSの`transition`時間を実測値に合わせて調整すること
