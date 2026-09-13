# pskeeb5 3x6 - カスタムキーマップ

[klesh/zmk-config-pskeeb5-3x6](https://github.com/klesh/zmk-config-pskeeb5-3x6) をforkし、キーマップ・Auto Mouse Layer・PS/2ドライバーをカスタマイズしたリポジトリです。

## PS/2ドライバーの更新について

トラックポイント用のPS/2ドライバー([kb_zmk_ps2_mouse_trackpoint_driver](https://github.com/klesh/kb_zmk_ps2_mouse_trackpoint_driver))を[qnaiv/kb_zmk_ps2_mouse_trackpoint_driver](https://github.com/qnaiv/kb_zmk_ps2_mouse_trackpoint_driver)としてforkし、以下の機能を追加しています(`config/west.yml`でこのforkを参照するよう設定済み)。

- **`excluded-positions`プロパティを追加**: Auto Mouse Layerが有効な間、指定したキー位置(マウスレイヤー自身のクリック/スクロールキー)以外のキーが押されたら、タイムアウトを待たずに即座にレイヤーを解除する機能。ZMK標準のCaps Word機能と同じ仕組み(`zmk_position_state_changed`イベントの直接購読)で実装しており、既存のマウス移動/クリック処理(HIDレポート送信経路)には一切手を入れていない。
- 実装箇所: `src/mouse/input_listener_ps2.c` / `dts/bindings/zmk,input-listener.yaml`
- **注意(修正済みの不具合)**: `zmk_position_state_changed`はZMKがそのキー位置のbehaviorを解決・実行する**前**に発行されるため、当初の実装(イベント受信時に即座にレイヤーを無効化)では、マウスレイヤー上のクリックキー自身を除外リストから外すと、クリックがそのキー自体を無効化してしまい発動しなくなるバグがあった。システムワークキューに遅延実行するよう修正し、現在のキー入力の処理が完了してからレイヤーを無効化するようにして解決した。

## 現在の設定内容

### ハードウェア構成
- pskeeb5 3x6(6列コラムナー分割キーボード、44キー、`nice_nano_v2` × 2)
- PS/2トラックポイント(右側central)
- ロータリーエンコーダー × 2(左右各1個)

### レイヤー構成
| # | 名前 | 内容 |
|---|------|------|
| 0 | Base | メインレイヤー |
| 1 | Num | F1〜F12・数字・テンキー記号(スペース長押しで入る) |
| 2 | Sym | 記号(`'` `"` `!` `@` `#` `(` `)` 等) |
| 3 | Nav | 矢印・Home/End・PageUp/Down・BTプロファイル(親指キー長押しで入る) |
| 4 | Special | BTプロファイル選択・トラックポイント感度調整・レイヤー直接切替 |
| 5〜14 | (予備) | 未使用のプレースホルダー |
| 15 | Auto Mouse | トラックポイント操作で自動的に有効化 |

### Baseレイヤー
- 左端: Tab / Shift / Ctrl、右端: Bksp / Enter / `-`(6列化に伴う追加列)
- ホームロウはホールドタップ: 薬指(S/L)=Alt、中指(D/K)=Ctrl、人差し指(F/J)=Shift。**Cmdだけは小指ではなくG/Hキー**(タップでG/H、ホールドでCmd)
- 親指クラスター: `Power` / `Alt` / `Cmd` / `Space(長押しでNum)` / `Space(長押しでNum)` / `Nav(長押し)` / `/` / `再生・一時停止`
- Hold-Tapには`require-prior-idle-ms = 150ms`を設定し、高速タイピング中の意図しない修飾キー暴発を抑制
- **Positional Home Row Mods**: ホームロウのホールドタップ(`hml`/`hmr`)は、`hold-trigger-key-positions`で**反対側の手のキー位置のみ**をhold判定のトリガーにし、`hold-trigger-on-release`でキーを離したタイミングで判定するよう設定。これにより同じ手だけで完結するタイピング(例: "bad "のような単語)はtap優先で安定し、反対の手との組み合わせ(実際のショートカット、例: D長押し+右手側のSpace同時押しでCtrl+Space)は確実にhold判定になる。半角/全角切り替え(D長押し+Space同時押し)でDがそのままtap入力されてしまう不具合の対策として導入([参考記事](https://zenn.dev/tomori_k/articles/9793e7300488f6))

### ロータリーエンコーダー
- 左: 回転=縦スクロール
- 右: 回転=タブ切替(Ctrl+Tab / Ctrl+Shift+Tab)
- 押込みの物理位置は未確認(pos39/40は独立したスペースキーと判明したため除外済み)

### Auto Mouse Layer(トラックポイント連動)
- トラックポイントを動かすと自動でレイヤー15に切替わり、**10秒**操作がないと自動的に元のレイヤーに戻る
- レイヤー15内でマッピングされているのは以下のみ(それ以外は`&none`)
  - R位置: 上スクロール / F位置: 下スクロール
  - 親指: 左クリック / 右クリック / 中央クリック
- `excluded-positions = <4 16 40>`(スクロールと左クリック)以外のキーを押すとタイムアウトを待たずに即座にレイヤーを抜ける。**右クリック・中央クリックは意図的に除外リストに入れておらず、発動した瞬間にレイヤーを抜ける**(左クリック・スクロールは継続してレイヤーに留まる)

### コンボ
- `Tab + Q` → Esc
- `E + R` → Tab
- スペース2キー同時押し → 中クリック

## キーマップの可視化

現在のキーマップ全体を可視化したArtifactはこちら: [pskeeb5 3x6 Keymap](https://claude.ai/code/artifact/3a70df41-fdfc-427e-a294-ab49dbfbe915)
