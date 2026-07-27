# RESEARCH_sanma — 雀魂「三麻」牌譜 → 構造化JSON の実現性・規模見積もり

> 調査：ニア / 2026-07-27。対象：`SPEC.md` §5追記の未確定1。
> スコープ：変換のみ。検討エンジン側の三麻対応・認証・規約は対象外。
> **前回の `RESEARCH.md`（および SPEC v0.2 の三麻に関する記述）を一部訂正する。**

---

## 結論（3行）

- **自作できる。というより、ほぼ自作しなくていい。** `Equim-chan/tensoul` の `convert.js` には
  **既に三麻分岐が実装されている**（`if (3 == nplayers)` / `case "RecordBaBei"` → `"f44"` / 三麻ツモ損 `TSUMOLOSSOFF`）。ソースを直接読んで確認。
- **規模：ゼロから書いても 600〜1000行（1ファイル規模）。** tensoul を流用するなら変換部の新規実装は 0 行、検証と微修正だけ。
- **前回の「三麻対応は世に無い」は誤り。** README の "standard games with common rules" を信じてコードを読まなかったのが原因。
  少なくとも4リポジトリが三麻を扱っている。

---

## 論点1：雀魂の三麻レコードの構造

### 確認済み事実（`liqi.proto` の実物を読んだ結果）

参照した定義ファイル（いずれも raw を直接取得）：`EndlessCheng/mahjong-helper` の `platform/majsoul/proto/lq/liqi.proto` [S1]、
`latorc/MahjongCopilot` の `liqi_proto/liqi.proto`（より新しい）[S2]。

#### (a) 三麻でも四麻と同じメッセージ構造。四麻用の型を使い回している

```proto
message RecordNewRound {
	uint32 chang = 1; uint32 ju = 2; uint32 ben = 3;
	string dora = 4;
	repeated int32 scores = 5;
	uint32 liqibang = 6;
	repeated string tiles0 = 7;   // 席0の配牌
	repeated string tiles1 = 8;
	repeated string tiles2 = 9;
	repeated string tiles3 = 10;  // ← 三麻では空
	repeated TingPai tingpai = 11;
	OptionalOperationList operation = 12;
	string md5 = 13;
	string paishan = 14;          // 牌山
	uint32 left_tile_count = 15;
	repeated string doras = 16;
}
```

→ **三麻専用フィールドは無い。`tiles3` が空、`scores` の要素が3、`paishan` が短い、というだけ。**
牌姿は `"1m"`〜`"7z"` 形式の文字列なので、**萬子2〜8は単に出現しないだけ**（欠け牌のための特別なフィールドは無い＝確認済み事実）。

#### (b) 北抜きは専用レコード `RecordBaBei` として存在する（拔北 = ba bei）

```proto
message RecordBaBei {
	uint32 seat = 1;
	repeated string doras = 6;
	repeated OptionalOperationList operations = 7;
	bool moqie = 8;
}
```

関連：リアルタイム側に `ActionBaBei{seat, operation, doras, zhenting, tingpais, moqie}`、
集計側に `RecordBaBeiInfo{seat, is_zi_mo, is_chong, is_bei}`、`GameRoundHuData.babei_count = 12`、
`RecordPeiPaiInfo{dora_count, r_dora_count, bei_count}`。[S1][S2][S3]

→ **北抜きは「打牌」ではなく独立イベント。抜いた牌（常に北）はフィールドに無く `seat` のみ。ここが変換で落としやすい。**

#### (c) 花牌は雀魂の日本式三麻には無い

`liqi.proto` に牌譜レコード側の `hua`/`flower` フィールドは見つからなかった（**不在確認なので確信度は中**）。
代わりに `GameDetailRule` に次のフラグがある [S2]：

```proto
bool have_zimosun = 39;   // ツモ損（三麻ルール分岐に直結）
uint32 guyi_mode  = 42;   // 古役
uint32 dora3_mode = 43;
uint32 muyu_mode  = 46;   // 木魚
uint32 chuanma    = 50;   // 川麻（四川式・3人系の別ルール）
```

→ **`chuanma`（川麻）という別系統の3人ルールが存在する。** 段位戦の三人東/南だけを対象にすれば無関係（**推測・未検証**）。

#### (d) 人数の判別方法

`GameDetailRule` に人数フィールドは無い。実装が使っているのは
`record.head.result.players.length`（tensoul [S4]）、`len(seatList) == 3`（MahjongCopilot [S5]）。
参考：`RecordGame.AccountInfo` に `AccountLevel level = 7;` と **`AccountLevel level3 = 8;`（＝三麻段位）** が並ぶ [S1]。

#### (e) 点数計算の差異はレコードに乗らない（計算済み結果が乗る）

`RecordHule{hules, old_scores, delta_scores, wait_timeout, scores, gameend, doras}`、
`HuleInfo{hand, ming, hu_tile, seat, zimo, qinjia, liqi, doras, li_doras, yiman, count, fans, fu, title,
point_rong, point_zimo_qin, point_zimo_xian, title_id, point_sum}`。

**`delta_scores` が確定値**なので三麻点計算の再実装は原則不要。
ただし tensoul は天鳳形式の要求に合わせて自前再計算しており、そこに三麻分岐が入る。

#### (f) レコードの入れ物と、必ず踏むバージョン分岐

`ResGameRecord{error, RecordGame head = 3, bytes data = 4, string data_url = 5}` → `GameDetailRecords`。
tensoul `client.js` の実コード [S18]：

```js
if (payload.version < 210715 && payload.records.length > 0) { ... }
else { log.data = payload.actions.filter((action) => action.result && action.result.length > 0) ... }
```

三麻とは無関係だが、**自作するなら必ず踏む。** 古いサンプルだけ見ると新形式で死ぬ。

### 失敗事例

- **`latorc/MahjongCopilot` #92「三麻不起作用（已放入三麻的模型并在Akagi中验证）」（2026-01-07 クローズ）** —
  **三麻対応が「ある」＝「動く」ではない**実例 [S6]。
- 継続：tensoul #17 / #10（雀魂API変更に追従できず未解決）。**レコード取得層は三麻と無関係に壊れる。**

---

## 論点2：出力形式は三麻を表現できるか → **両方できる。実装で確認済み。**

### (A) tenhou.net/6 — 表現できる（独立実装2本が一致）

**プレイヤーは4枠のまま、4人目を空で埋める。**

- tensoul（出力側）：`if (3 == nplayers) { res["name"][3] = ""; res["sx"][3] = ""; }` [S4]
- convlog 3p（入力側）：`RawKyoku{ ... haipai_0..2:[Tile;13], haipai_3:[Tile;0], ... }` [S7]

**北抜きは打牌列に副露記法 `"f44"` として入る**（`f`=抜き、`44`=天鳳牌番号の北）。

tensoul（出力）:
```js
case "RecordBaBei" :
{   //kita - this record (only) gives {seat, moqie}
    //NOTE: tenhou doesn't mark its kita based on when they were drawn, so we won't
    //if (e.moqie)
    //    kyoku.discards[e.seat].push("f" + TSUMOGIRI);
    //else
    kyoku.discards[e.seat].push("f44");
    kyoku.ldseat = e.seat; // for nukidora ron
    return;
}
```

convlog 3p（入力）:
```rust
else if naki.contains(&b'f') {
    // nukidora  e.g. "f44" => Nukidora N
    if naki_string.len() != 3 { return Err(ConvertError::InvalidNaki(naki_string.clone())); }
    if &naki[1..3] != b"44" { return Err(ConvertError::InvalidNaki(naki_string.clone())); }
    let ev = Event::Nukidora { actor, pai: tiles_from_tenhou_bytes(&naki[1..3])? };
    ret.push(ev);
}
```

**出力側と入力側が独立実装で `"f44"` に一致＝事実上の標準と見てよい。**[S4][S8]

- **萬子2〜8は「出てこないだけ」**（牌は数値エンコード：萬子11-19/筒21-29/索31-39/字41-47/赤51-53。12〜18が現れないだけ）。
- **ツモ損**：`TSUMOLOSSOFF = (3 == nplayers) ? !record.head.config.mode.detail_rule.have_zimosun : false;`
  / `if (3 == kyoku.nplayers) delta[h.seat] += (TSUMOLOSSOFF ? 0 : liablefor * YSCORE[OYA][KO]);` [S4]

### (B) mjai — 素の mjai は三麻非対応。が、拡張版が実在する

- **上流 `Equim-chan/mjai-reviewer` の `convlog/src/mjai.rs` の `Event` enum に `Nukidora` は無い**（不在確認）[S9]。
- **3人版フォーク `hidacow/mjai-reviewer3p`（branch `3p`, Apache-2.0）が追加している**[S8]：

```rust
pub enum Event {
    None,
    StartGame { names: [String; 4], kyoku_first: u8, aka_flag: bool },
    StartKyoku { bakaze, dora_marker, kyoku, honba, kyotaku, oya, scores: [i32; 4], tehais: [[Tile; 13]; 4] },
    Tsumo { actor, pai }, Dahai { actor, pai, tsumogiri },
    Chi {..}, Pon {..}, Daiminkan {..}, Kakan {..}, Ankan {..},
    Nukidora { actor: u8, pai: Tile },        // ★三麻用に追加
    Dora {..}, Reach {..}, ReachAccepted {..},
    Hora { actor, target, deltas: Option<[i32;4]>, ura_markers },
    Ryukyoku { deltas: Option<[i32;4]> }, EndKyoku, EndGame,
}
```

**注目：3人版でも配列は `[_; 4]` のまま。＝mjai 側も「4枠のうち1枠をダミーで埋める」設計。**

- 独立裏取り：`MahjongCopilot` は雀魂レコードから直接 mjai を組み立て、
  `LiqiAction.BaBei` → `{'type': MjaiType.NUKIDORA, 'actor': actor, 'pai': 'N'}` を発行 [S5]。
  **イベント名・`pai` に北、で hidacow 版と一致。**
- 参考：3人麻雀AI群でも action label `40 => Event::Nukidora` [S10][S11]。

### (C) 独自JSONは不要（判断）

**mjai + `nukidora`（4枠パディング）** が妥当。既存2実装と表記が揃うので、後から検討エンジンに流す道も残る。
AI に読ませる目的なら、mjai に人間可読なメタ（局・場風・点棒推移）を足すだけで足りる。

### 失敗事例

1. **tenhou/6 は北抜きの「ツモ切りか」を表現できない。** tensoul のコメントが明言：
   `//NOTE: tenhou doesn't mark its kita based on when they were drawn, so we won't`。
   **＝`RecordBaBei.moqie` は tenhou/6 経由で必ず落ちる。二段変換は情報が黙って欠ける。**[S4]
2. **convlog 3p は4人前提の演算を残している**（`(actor + 1) % 4` 等が `conv.rs` 749行に残存）。
   相対席の意味が三麻でずれうる（**推測・未検証**）[S8]。
3. **`mjai-reviewer3p` README の制約**：「You should modify the Mortal engine (Use libriichi3p, modify Grp related code, etc.)
   to make it work with this fork.」＝そのままでは動かない。**「3p対応と書いてあっても即使えるとは限らない」実例**[S10]。
4. 前回分：tensoul #14「ポンの巡目が曖昧なとき tenhou/6 の結果が不正」＝**変換器は静かに間違う。**

---

## 論点3：自作の規模見積もり

### 既存実装のコード規模（実ファイルを取得して計測）

| 実装 | ファイル | 行数 | 役割 | 三麻 |
|---|---|---:|---|---|
| `Equim-chan/tensoul`（MIT, JS） | `convert.js` | **約650** | 雀魂レコード → tenhou.net/6。変換ロジック本体 | **対応済み** |
| 同 | `client.js` | 237 | WS接続・ログイン・fetchGameRecord・version分岐・data_url | — |
| 同 | `index.js` | 56 | CLI / HTTPサーバ（`GET /convert?id=`） | — |
| **tensoul 合計** | | **約950行** | | |
| `hidacow/mjai-reviewer3p`（Apache-2.0, Rust） | `convlog/src/conv.rs` | **749** | tenhou/6 → mjai(3p) | **対応済み** |
| 同 | `convlog/src/mjai.rs` | 124 | mjaiイベント定義（`Nukidora`入り） | 同 |
| 同 | `convlog/src/tenhou/json_scheme.rs` | 180 | tenhou/6 スキーマ | 同 |
| `latorc/MahjongCopilot`（**GPLv3**, Python） | `game/game_state.py` | **732** | 雀魂 liqi → mjai を**直接**変換（3p/4p） | **対応済み** |

### 「4麻に三麻を足す」なら触る箇所（tensoul を読んだ限り）

1. 人数判定 `nplayers = record.head.result.players.length`
2. 出力の4枠パディング（`name[3]=""`, `sx[3]=""`, `pad_right(initscores, 4, 0)`）
3. 北抜きイベント `case "RecordBaBei"` → `"f44"` ＋ `kyoku.ldseat`（抜き北ロン用）
4. 点数計算 `TSUMOLOSSOFF`（`have_zimosun`）と和了 delta の三麻分岐
5. ルール表示文字列 `RUNES.sanma`

**＝実質5箇所、100行未満の追加。**（`data.json` には既に `sanma_fish.png` 等の三麻段位定義も入っている [S13]）

### 3案の比較

| 案 | 内容 | 新規実装量 | リスク |
|---|---|---|---|
| **①tensoul をそのまま使う**（出力=tenhou/6） | 変換0行。認証層だけ JP 向けに差し替え | **0行** | 上流停滞（#10/#17未解決）。北抜き moqie が落ちる |
| **②tensoul + tenhou/6→mjai** | ①に convlog 3p 流用 or 自前 | 200〜400行 | 二段変換で情報欠落・誤差 |
| **③protobufから直接自作**（liqi → mjai系JSON） | MahjongCopilot `game_state.py`(732行) が同じことをしている | **600〜1000行** | 正しさを全部自分で担保。ただし**情報の落ちが無い** |

**推奨（判断・断定ではない）：③、ただし tensoul `convert.js` を答え合わせ用の参照実装として横に置く。**
理由：(a) 目的が「AIに読ませる」で tenhou/6 を通す必然性が無い、(b) 二段変換の情報欠落を避けられる、
(c) 600〜1000行なら1人で書ける、(d) **MIT の tensoul は読む・GPLv3 の MahjongCopilot は読まない**という切り分けで安全。

> ⚠ **新規論点・ライセンス**：`MahjongCopilot` は **GPLv3**。取り込むと自作物も GPLv3。
> 個人利用・非公開なら義務は発生しないが、混ぜたことを忘れると後で詰む。
> `tensoul`(MIT) / `mjai-reviewer3p`(Apache-2.0) は取り込み可。

### 「三麻対応の形跡」再調査（前回の見落とし）

**英語圏**：`hidacow/mjai-reviewer3p`（Apache-2.0 / 2025-02-23 / branch `3p`）[S10][S12]、
`hidacow/mjai-batch-review`（TS / MIT / 2024-11-09、三麻の明記なし）[S14]、
`hidacow/tensoul`（JS / MIT / 2024-11-10、README は上流と同文）[S15]、
`smly/RiichiEnv` / `VictorZXY/Meowjong` / arXiv 2202.12847（AI側の三麻）[S11]。

**中国語圏**：`latorc/MahjongCopilot`（GPLv3、「Now supports Majsoul 3-person and 4-person game modes」、
**雀魂 liqi → mjai を三麻込みで直接実装している唯一の明示例**）[S2][S5]、
`shinkuan/Akagi`（Apache-2.0、「3-player mahjong (sanma) — full pipeline」を alpha.8 で完了、変換は `src/bridge/majsoul/`）[S16]。

※ Akagi / MahjongCopilot は**リアルタイム支援ツール**＝前回確認した摘発告知（「対局の公平性を損なう行為」）が
まさに名指しする類型。**コードを参考にするのと、そのツールを使うのは別問題。**

### 失敗事例

- MahjongCopilot #92（上記）。
- **hidacow の三麻系リポは全て 2024-11〜2025-02 で更新停止**（2026-07 時点で1年半）。雀魂API変更に追従していない可能性（**推測**）。
- **前回の自分の失敗**：README の記述と他リポの「4人麻雀のみ」から推測で結論した。
  **コードを読めば1回の fetch で分かった。README は実装より古い/雑。**

---

## リサーチ不足（肯定/否定が揃っていない）

- **tensoul の三麻出力が「実際に正しいか」は未検証。** コードに分岐があることは確認したが実行結果は見ていない
  （牌譜も token も無く原理的に不可）→ **ren の実機確認事項。**
- `paishan` の三麻での中身（108枚か等）は定義から読めず**未検証**。
- `chuanma`（川麻）モードのレコード構造は**未検証**。
- 花牌の不在は「proto に該当フィールドが無い」という**弱い不在証明**のみ。

## 取得できなかったURL

| URL | 症状 | 種別 | 影響 |
|---|---|---|---|
| `https://mortal.ekyu.moe/docs/mjai.html` | **403** | egressポリシー | mjai仕様の一次ドキュメント読めず → convlog ソースで代替（むしろ確実） |
| `https://blog.kobalab.net/entry/20170312/1489315432` | **403** | egressポリシー | 天鳳形式の日本語解説読めず → `json_scheme.rs` で代替 |
| `raw.../ssttkkl/tensoul-py/main/tensoul/convert.py` | 404 | **こちらのパス推測ミス** | tensoul-py の三麻対応有無は**未確認** |
| `github.com/shinkuan/Akagi/tree/v2/src/bridge/majsoul` | 404 | パス推測ミス | Akagi 変換部の行数を計測できず（READMEのみ） |
| `raw.../hidacow/mjai-reviewer3p/main/...` | 404 | ブランチが `main` でなく `3p` | 再取得で解決済み |

※ 前回403だった `mjai.ekyu.moe` / 雀魂公式 / note.com は今回スコープ外のため再試行せず。

## ソース表

| source_id | URL | 取得日 | 論点 | 信頼度 | 種別 | 要約 |
|---|---|---|---|---|---|---|
| S1 | raw.githubusercontent.com/EndlessCheng/mahjong-helper/master/platform/majsoul/proto/lq/liqi.proto | 2026-07-27 | 1 | **高（一次・定義本体）** | 事実 | `RecordNewRound`(tiles0-3,paishan,scores)、`RecordBaBei{seat,doras,operations,moqie}`、`ActionBaBei`、`RecordHule`/`HuleInfo`、`GameDetailRule`(〜43)、`RecordGame.AccountInfo{level, level3}`、`ResGameRecord{head,data,data_url}`、`GameDetailRecords`、`ActionPrototype{step,name,data}` を逐語取得 |
| S2 | raw.githubusercontent.com/latorc/MahjongCopilot/main/liqi_proto/liqi.proto | 2026-07-27 | 1 | **高（一次・新しい定義）** | 事実 | `have_zimosun=39, guyi_mode=42, dora3_mode=43, muyu_mode=46, chuanma=50`、`RecordBaBeiInfo`、`GameRoundHuData.babei_count=12` |
| S3 | raw.githubusercontent.com/MahjongRepository/mahjong_soul_api/master/ms/protocol.proto | 2026-07-27 | 1 | 中（取得成功だが要約不完全） | 事実 | `muyu_mode=46`、`RecordPeiPaiInfo{dora_count,r_dora_count,bei_count}`、`RecordBaBeiInfo` の存在確認 |
| S4 | raw.githubusercontent.com/Equim-chan/tensoul/master/convert.js | 2026-07-27 | 1,2,3 | **高（一次・実コード）** | 事実 | 約650行。**三麻分岐あり**：`nplayers`、`if (3 == nplayers) {name[3]="" ...}`、`TSUMOLOSSOFF`、`case "RecordBaBei"`→`"f44"`＋`ldseat`、`RUNES.sanma` |
| S5 | raw.githubusercontent.com/latorc/MahjongCopilot/main/game/game_state.py | 2026-07-27 | 1,2,3 | **高（一次・実コード）** | 事実 | 732行。`len(seatList)==3 → MJ3P`、`LiqiAction.BaBei`→`{'type':NUKIDORA,'actor':actor,'pai':'N'}`、`player_scores + [0]` |
| S6 | github.com/latorc/MahjongCopilot/issues?q=is:issue+3p | 2026-07-27 | 1,3 | 高 | 事実 | #92「三麻不起作用」2026-01-07クローズ、#18「三麻功能如何使用」2024-04-18クローズ |
| S7 | raw.../hidacow/mjai-reviewer3p/3p/convlog/src/tenhou/json_scheme.rs | 2026-07-27 | 2 | **高（一次・実コード）** | 事実 | 180行。`RawLog{names:[String;4]...}`、`RawKyoku{... haipai_3:[Tile;0] ...}`、`ActionItem = Tile\|Tsumogiri\|Naki(String)` |
| S8 | raw.../hidacow/mjai-reviewer3p/3p/convlog/src/{mjai.rs, conv.rs} | 2026-07-27 | 2,3 | **高（一次・実コード）** | 事実 | mjai.rs 124行に`Nukidora{actor,pai}`（配列は`[_;4]`）。conv.rs 749行で`"f44"`厳密チェック→`Event::Nukidora`。`(actor+1)%4`残存 |
| S9 | raw.../Equim-chan/mjai-reviewer/master/convlog/src/mjai.rs | 2026-07-27 | 2 | 高 | 事実 | **上流の`Event`に`Nukidora`は無い**（素のmjaiは三麻非対応） |
| S10 | raw.../hidacow/mjai-reviewer3p/3p/README.md | 2026-07-27 | 2,3 | 高 | 事実 | 「fork to support reviewing 3-player mahjong」「modify the Mortal engine (Use libriichi3p...)」「action label (eg. 40 => Event::Nukidora)」。Apache-2.0 |
| S11 | 検索結果（arXiv 2202.12847 / VictorZXY/Meowjong / smly/RiichiEnv / smly/mjai.app） | 2026-07-27 | 2 | 中 | 事実/意見 | 三麻はチー無し・キタ有り、萬子の大半を除く。mjai系で`Event::Nukidora`(label 40) |
| S12 | github.com/hidacow?tab=repositories | 2026-07-27 | 3 | 高 | 事実 | mjai-reviewer3p(Rust/Apache-2.0/2025-02-23)、tensoul(JS/MIT/2024-11-10)、mjai-batch-review(TS/MIT/2024-11-09) |
| S13 | raw.../Equim-chan/tensoul/master/data.json | 2026-07-27 | 3 | 中 | 事実 | `level_definition` に `sanma_fish.png` 等の**三麻段位定義** |
| S14 | github.com/hidacow/mjai-batch-review | 2026-07-27 | 3 | 中 | 事実 | MajSoulログ一括レビュー。**三麻の記載なし**。MIT |
| S15 | github.com/hidacow/tensoul | 2026-07-27 | 3 | 中 | 事実 | tensoulのフォーク。README上流と同文、三麻記載なし。40 commits |
| S16 | github.com/shinkuan/Akagi | 2026-07-27 | 3 | 中 | 事実 | 「3-player mahjong (sanma) — full pipeline」alpha.8完了。変換は`src/bridge/majsoul/`。Apache-2.0。**リアルタイム支援＝規約上の類型が違う** |
| S17 | github.com/Equim-chan/tensoul（README.adoc＋ファイル一覧＋issues） | 2026-07-27 | 2,3 | 高 | 事実 | 「only works with standard games with common rules, like those in the ranked lobbies」。**READMEに三麻の言及は一切無い（コードにはある）**。#17/#10オープン、#14クローズ |
| S18 | raw.../Equim-chan/tensoul/master/{index.js, client.js} | 2026-07-27 | 1,3 | 高 | 事実 | index.js 56行、client.js 237行。`payload.version < 210715` で records/actions 分岐、`data_url`取得、`oauth2Login` |

---

## SPEC に返す差分

1. **§5追記の表「変換系OSSは三麻ほぼ全滅」は撤回・訂正。** tensoul は三麻対応済み（コードで確認）。
2. **§5 残った未確定1（三麻の変換）は解消。** レコード構造・出力形式・前例のいずれも三麻を表現できる。
3. **新規未確定**：tensoul 三麻出力の**正しさ**（ren 実機確認）／`chuanma` モード／MahjongCopilot の GPLv3 汚染回避。
4. **§7 の L3 判断は依然残る。** 三麻がボトルネックでなくなり、**判断は「規約リスクを飲むか（B/C/D）」の一点に絞られた。**

> いい話には裏も探した結果、今回は逆に「無いと思っていたものが有った」側だった。
> ただし**「コードに分岐がある」＝「正しく動く」ではない**（MahjongCopilot #92 がその実例）。実牌譜での確認は ren の手が要る。
