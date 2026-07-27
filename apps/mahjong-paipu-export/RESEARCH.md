# RESEARCH — 雀魂 牌譜URL → JSON の実現性調査

> 調査：ニア（リサーチ役） / 2026-07-26
> 対象：`SPEC.md` §5 の未確定4点
> **注意**：本調査はクラウド環境から実施。egress ポリシーで到達できなかったURLが複数ある（末尾に一覧）。
> そこは推測で埋めず「取得できなかった」として残してある。

---

## 結論（一文）

**「牌譜URLをコピペ→JSON」は技術的には作れる。ただし成立させるには「れんの雀魂アカウントの access_token を Worker に持たせて非公式WebSocket/protobuf APIを叩く」ことが事実上必須で、そこが雀魂利用規約 第11条（不正な方法でのコンテンツ取得／リバースエンジニアリング）に真正面から当たる — 技術で詰むのではなく、規約と token 運用で詰む。**

- 技術的には Yes
- 規約的にはグレーではなく**黒寄り**
- 運用的には token 期限と雀魂側の仕様変更で**継続的に壊れる**

---

## 論点1：取得経路と認証

### 確認済み事実

- **牌譜URLを HTTP で GET しても対局データは取れない。** `https://game.maj-soul.com/1/?paipu=<uuid>_a<accountid>` はゲームクライアント（Unity WebGL）を読み込むだけのページで、レコード本体は WebSocket + protobuf(liqi) の `fetchGameRecord` RPC で取得される。**SPEC §5 の懸念は当たっていた。** [S1][S2][S6]
- **`fetchGameRecord` には認証（uid + access_token）が要る。** 公開・未認証で取れる経路は見つからなかった。複数の実装が例外なく「majsoul にログインした状態のトークン」を前提にしている。[S1][S3][S5][S6]
- **CN サーバのみ ID/パスワードでログイン可能。JP/EN はメールコード or ソーシャル認証で、ライブラリ側は「自分で実装しろ」と明記**（`MahjongRepository/mahjong_soul_api` README）[S5]
  → **れんは JP 前提なので、既存 OSS の「ログイン」部分はそのままでは使えない。**
- JP で現実的なのは **ブラウザで一度ログインして uid / access_token を抜き、それを固定値として使う**方式。JP サーバ向けの実運用例あり（`4n3u/majsoul-monthTicket-auto`：ブラウザから UID と TOKEN を取り GitHub Actions で毎日叩く）。ただし **token の有効期限・更新方法についてドキュメントに記述なし**、かつ「本プロジェクト利用による BAN 等の不利益は自己責任」と明記。[S7]
- 別系統として、**ブラウザ内で `window.WebSocket` をフックして `fetchGameRecord` の生レスポンスを横取りする**方式があり、これは**追加の認証情報を持たなくてよい**（ログイン済みの正規クライアントの通信をそのまま使う）。`honvl/Majsoul-to-NAGA`（userscript, MIT）、`kbkn3/MahjongSoul-review-supporter`（Chrome/Edge拡張, Apache-2.0）が実装。[S3][S8]

### 既存OSS（実在確認済み）

| リポジトリ | 何をする | ライセンス | 最終更新 | 認証の扱い | 流用可否 |
|---|---|---|---|---|---|
| `Equim-chan/tensoul` [S2] | 雀魂ログ → tenhou.net/6 JSON。CLI と HTTPサーバ（`GET /convert?id=...`）両対応。Node.js | MIT | **確認できず**（commitsページが空で返る）。未解決issueが2025-09で放置＝**停滞と推測** | `GameMgr.Inst.access_token` を devtools で抜いて設定 | **この用途にほぼ理想形だが要メンテ** |
| `ssttkkl/tensoul-py` [S9] | 同上の Python 実装 | **明記なし＝未確認** | 2025-08-21 | `MajsoulPaipuDownloader(username, password)`。**CNアカウントのみ** | JPでは認証部の差し替えが要る |
| `Equim-chan/mjai-reviewer` [S4] | tenhou.net/6 ログ → Mortal/akochan でレビュー（旧 akochan-reviewer の後継） | Apache-2.0 | 活発 | **自分では雀魂から取らない**。「雀魂は Web アプリを使え」と誘導 | レビュワー。ただし `convlog`(tenhou/6→mjai) は流用価値あり |
| `MahjongRepository/mahjong_soul_api` [S5] | 雀魂 protobuf(liqi) の Python ラッパ | **明記なし＝未確認** | **2026-06-18（生きている）** | CNは user/pass、JP/ENは自前実装 | **protobuf層の土台として現役で最有力** |
| `SAPikachu/amae-koromo` [S10] | 雀魂牌譜屋。対局サマリの収集・表示 | MIT（scripts側） | **2026-07-24 / 07-25（現役）** | 収集側はアカウント必要 | **牌譜「本体」は提供しない。UUIDの索引用途。今回は不要** |
| `jeff39389327/MajsoulPaipuConvert` [S11] | UUID収集→本体DL→mjai/tenhou.net/6出力 | Apache-2.0 + MIT | 不明（83 commits） | **config.env にメール＋パスワード。CNのみ** | 一括収集向けで過剰。JP非対応 |
| `honvl/Majsoul-to-NAGA` [S3] | `window.WebSocket` をフックして replay を捕捉 → NAGA へ | MIT | 不明（25 commits） | **追加認証不要**（ログイン済みセッションに寄生） | **認証問題を回避する唯一の系統。デスクトップブラウザ限定** |
| `kbkn3/MahjongSoul-review-supporter` [S8] | 同上の Chrome/Edge 拡張。天鳳形式に変換して NAGA / mjai-reviewer へ | Apache-2.0 | 活発（118 commits） | 同上 | 同上。**iPhoneでは動かない** |
| `Cryolite/majsoul-rpa`, `Apricot-S/majsoulrpa` [S12] | 画面操作RPA | — | — | アカウント必須 | 過剰・BANリスク最大。**不採用** |
| `zyr17/MajsoulPaipuAnalyzer` [S13] | 牌譜分析ツール | 未確認 | 未確認 | 未確認 | 解析側。スコープ外 |

### 失敗事例（＝この構成が壊れる実例）

1. **`Equim-chan/tensoul` issue #17「KeyError: 'region_urls'」（2025-09-05報告・未解決）** — エンドポイント発見手順が通らない。報告者は「majsoul API 側が変わった」と推測。同リポの #10「Update to work with recent changes to MahjongSoul API」も**オープンのまま**。つまり**雀魂の仕様変更で即死に、メンテが追いついていない**。[S14]
   ※ #17 のトレースバックが `downloader.py`（tensoul-py 側のファイル名）で、**リポ取り違えの可能性あり。この1点は要再確認。**
2. **「雀魂はもう `GameMgr`/`app` グローバルを公開していない」**（Majsoul-to-NAGA README）— **tensoul が案内している `GameMgr.Inst.access_token` の取得手順は、既に通らなくなっている可能性が高い。** だから同スクリプトは WebSocket フックに切り替えた。[S3]
3. **`kbkn3/MahjongSoul-review-supporter` issue #14（オープン）** — 「Mortal に飛ぶが Game URL が空」。以前は動いていたのが URL 途中切れ→未転送へ悪化。**この種の連携は定期的に壊れる**実例。[S15]
4. **2026-03-10 の Yostar ID ログイン必須化（JP）**[S16] — JPのログインフローが移行済み。**2025年以前に書かれた JP の token 取得手順は、この時点で無効化された可能性が高い（未検証・要実機確認）。**

### 推測（明示）

- 現実的な構成は **①れんが手動でブラウザから access_token を抜く → ②Worker に Secret として置く → ③paipu UUID を受けて WS+protobuf で fetchGameRecord → ④変換して返す**。tensoul のサーバモードがまさにこの形。
- **token は永続しないと見るべき**（明示ソースなし＝未検証）。切れるたびに手作業で入れ直す運用になり、**「コピペするだけ」の体験は token 更新の日に崩れる。**
- Cloudflare Workers から majsoul の WS を張るのは技術的には可能（`fetch()` に `Upgrade: websocket`。グローバル `WebSocket` コンストラクタは使えない）[S17]。ただし tensoul(Node) をそのまま載せられるかは**未検証**。

---

## 論点2：出力フォーマット（tenhou.net/6 vs mjai）

### 確認済み事実

- **mjai 形式**：1行1JSONオブジェクトの JSON Lines。イベント型が語（`start_kyoku`/`tsumo`/`dahai`/`chi`/`pon`/`ankan`/`daiminkan`/`kakan`/`reach`/`reach_accepted`/`dora`/`hora`/`ryukyoku`/`end_kyoku`/`end_game`）。牌表記も可読（`7s`、赤5ピンは `5pr`、伏せ牌は `?`）。**JSON Schema が `Cryolite/mjai` の `/schema` にある。**[S18]
- mjai には in-game モード（プレイヤー視点）と replay モード（全公開）の2面。牌譜変換で欲しいのは replay。[S18]
- **mjai 標準化は未完成。** `Cryolite/mjai` は hora / ryukyoku など主要部分が **(TODO)** のまま。[S18]
- **tenhou.net/6 形式**：公式仕様書は見つからず。牌は数値（萬11–19/筒21–29/索31–39/字41–47、赤は51/52/53）、鳴きはビット演算でパックされた**非可読エンコード**（`c192021`, `p2222` 等）。実装からの逆算でしか仕様が分からない。[S19][S20]
- **相互変換ツールは存在する**：雀魂→tenhou.net/6 は `tensoul`/`tensoul-py`、tenhou→mjai は `NikkeTryHard/tenhou-to-mjai`（mjai-reviewer の `convlog` 参照）、`tomohxx/mjai-gateway`。雀魂→mjai 直変換は `MajsoulPaipuConvert`。[S1][S11][S21]

### 判断

- **AIに読ませる用途なら mjai 形式が明確に上。**(a) イベント名と牌表記が自然言語的でLLMが前処理なしに意味を取れる (b) JSON Lines で局単位に切りやすい (c) 公開スキーマがある。tenhou.net/6 の `p2222` 的な鳴きエンコードをLLMに正しくデコードさせるのは困難（**推測。実測していない**）。
- ただし**ツール互換性は tenhou.net/6 が上**（NAGA / mjai-reviewer Web / 天鳳牌譜エディタが受ける）。
- **推奨：内部は tenhou.net/6 経由（変換実装が最も枯れている）、最終出力は mjai。** ただし二段変換は誤差が乗る（下記）。

### 失敗事例（＝変換器は静かに間違う）

- **tensoul issue #14「ポンの巡目が曖昧なとき tenhou/6 の結果が不正」（クローズ済）** — 変換器は「動く／動かない」の二値ではなく、**黙って間違った牌譜を吐くことがある。**[S14]
- **tensoul-py の 2025-08 のコミットが「加槓イベント欠落の修正」「パオ計上の修正」** — リリース後に見つかったバグ。**SPEC §4 の「鳴き」「点数移動」の受け入れ条件は、変換器のバグでそのまま落ちうる。**[S9]
- **tensoul の明示的制約：「標準ルールの段位戦等でのみ動作」。イベント戦・特殊ルールは undefined behavior。**[S2] 三麻も対象外の実装が多い。[S11]

---

## 論点3：iPhone からの入口

### 確認済み事実

- iOSショートカットの `URLの内容を取得` は **POST/PUT/PATCH で JSON ボディを送れる**。任意APIを叩ける。[S22]
- **共有シート**：「共有シートに表示」をONにすると、ホストアプリからの入力が**最初のアクションに渡る**。[S22]
- **雀魂 iOS アプリからの牌譜URL取得は「シェア」ボタン → URL表示 → 長押しコピー。一発コピーボタンは無い**（二次情報・確信度中）。[S23]

### 判断（★SPEC の修正が要る）

- **共有シート経由は成立しない可能性が高い（推測）。** 雀魂アプリが出すのは「URLの表示＋長押しコピー」であって、iOSの共有シートを開く導線ではないため。
- **現実的な入口は「クリップボードから読むショートカット」**（ホーム画面/ウィジェットから起動 → クリップボード取得 → Worker に POST → 結果をクリップボードへ）。
- ショートカット→自前Worker の構成自体は一般的で実例多数。ただし**「雀魂 × iOSショートカット × Cloudflare Workers」の前例は見つからなかった（＝自作になる）。**

### 失敗事例

- iOSショートカットの既知の踏み抜き：**Base64アクションが既定で76文字ごとに改行を入れて文字列を壊す**（"None"に変更が要る）、**`Content-Type` ヘッダが無視されることがある**、**ヘッダのキーにコロンを重ねると "The network connection was lost"**。[S22]
- 半荘1本のJSONは数十〜数百KBになるので、**クリップボード経由で大きな文字列を扱えるかは実機確認が要る**（AIに貼る時点でも同じ問題）。

---

## 論点4：利用規約とレート

### 確認済み事実（※一次ページに到達できず。確信度：中）

雀魂 利用規約 **第11条（禁止事項）** に、確認できた範囲で以下が含まれる（引用元はミラーサイト。**公式ページは egress でブロックされ確認できていない**）[S24]：

- 本サービスを改変、毀損し、又は**逆アセンブル、逆コンパイル、リバースエンジニアリングする行為**
- 本サービスの運用・利用を妨げる行為、又はそのおそれのある行為
- **当社が本サービスを通じて提供する各種コンテンツを不正な方法で取得する行為、又はこれを助長する行為**

加えて、公式が「外部ツール等の使用により対局の公平性を損なう行為」に対する停止措置を繰り返し告知している（2025-03、2025-06）。[S25]

### 評価（遠慮なく書く）

- **グレーではなく、黒寄り。**「非公式protobuf APIを、抜き出した access_token で叩いて牌譜を取得する」は、上記「コンテンツを不正な方法で取得する行為」と「リバースエンジニアリング」の両方に素直に当てはまる読みができる。**個人利用・非公開であることは規約上の免罪符にならない**（規約は用途で切っていない）。
- 一方で、**公式の摘発告知は一貫して「対局の公平性を損なう」＝リアルタイム支援ツールとBOTに向いている。** 事後の牌譜変換はそこには当たらない。**実際に NAGA / Mortal / 牌譜屋 / 各種拡張が長年公然と運営されており、摘発された形跡は見つからなかった（＝黙認されていると推測。ただし「摘発事例が見つからない」は「安全」の証明ではない）。**
- **リスクの所在は「BANされるのはれんのアカウント」。** token を使う＝れんのアカウントで叩く、なので雀魂側からは非公式クライアントからのアクセスとして記録される。
- 誤BANを論じた記事が存在するが、**note.com が egress でブロックされ本文を取得できず内容は未検証。**[S26]

### レート

- **雀魂に公開APIドキュメントは存在せず、公式のレート上限は不明（未検証）。** 「安全な頻度」を裏付けるソースは見つからなかった。
- **推奨（根拠は状況証拠のみ）**：1リクエスト＝1牌譜、手動トリガのみ、連続実行を避け、結果はキャッシュして同一UUIDを再取得しない（SPECの冪等要件と一致）。**バルク収集は絶対にしない。**

---

## 新規論点（SPECに無かったが決定に効く）

1. **access_token のライフサイクル管理**が実質的な第一級要件。「切れたら手で入れ直す」導線をSPECに書かないと、初回だけ動いて放置死する。
2. **JPサーバのログイン実装**が既存OSSで埋まっていない（CN前提）。**ここが実装コストの本体。** 2026-03-10 の Yostar ID 移行の影響も未評価。
3. **入口は「共有シート」でなく「クリップボード」。** SPEC §3 の想定を修正すべき。
4. **出力サイズ。** 半荘1本のJSONをiPhoneのクリップボード経由でAIに渡せるか未検証。駄目なら「URLを返してAIに読ませる」形に変わる＝設計が変わる。
5. **代替案A：変換を自作しない。** `mjai.ekyu.moe`（mjai-reviewer Web）は雀魂URLを "out-of-the-box" で受けるとREADMEにある[S4]。**もし同サイトが tenhou/6 JSON を落とさせてくれるなら、作るのは iOSショートカット1本で済み、token も規約リスクも自前で背負わずに済む。**
   ※ mjai.ekyu.moe は egress ブロックで確認できず（親セッションが再試行しても403）。**これが次に潰すべき最短の検証点。**
6. **代替案B：token を持たない設計。** PCブラウザで雀魂を開き、拡張/userscript でフックして Worker に POST（`kbkn3` 方式）。規約リスクは残るが**認証情報を保存しない**分だけ安全で壊れにくい。ただし iPhone 完結にはならない。

## リサーチ不足（肯定/否定が揃っていない）

- **論点3**：否定材料（ショートカットの既知バグ）は集まったが、**雀魂アプリの牌譜URLが実際にどう共有できるかの一次確認ができていない＝ren 確認事項。**
- **論点4のレート**：肯定・否定とも一次ソースなし。**未検証。**
- **`Equim-chan/tensoul` の実際の生死**：commitsページが空で返り、最終更新を確定できず。

---

## 取得できなかったURL（egress ポリシーで403 等）

推測で埋めなかった箇所の一覧。

| URL | 症状 | 影響 |
|---|---|---|
| `https://api.github.com/repos/...`（4件） | 403 | 各リポの license / pushed_at をAPIで確定できず。HTMLページ読みで代替 |
| `https://mjai.ekyu.moe/` , `/ja.html` | 403（**親セッションの再試行でも403**） | **最重要**。代替案Aが未評価 |
| `https://mahjongsoul.yo-star.com/` , `https://mahjongsoul.com/` | 403 | **公式利用規約の一次確認ができず。** 論点4はミラー＋検索スニペット依存＝確信度中 |
| `https://mahjongsoul.club/content/利用規約` | 403 | 同上 |
| `https://note.com/yosuke_murasame/n/ncc4a25093334`（誤BAN考察） | 403 | BAN実例の検証ができず |
| `https://github.com/Equim-chan/tensoul/commits/master` | 200だが履歴が空 | 最終更新日を確定できず |
| `raw.githubusercontent.com/Equim-chan/tensoul/master/README.md` | 404 | README原文の逐語確認ができず |
| GitHub issues 一覧（tensoul-py, majsoul-monthTicket-auto） | 200だが本文が空 | issue個別の確認ができず |

---

## ソース表

| source_id | URL | 取得日 | 論点 | 信頼度 | 種別 | 要約 |
|---|---|---|---|---|---|---|
| S1 | github.com/NikkeTryHard/tenhou-to-mjai `/docs/majsoul-scraper.md` | 2026-07-26 | 1,2 | 中 | 事実 | UUIDはamae-koromo API、本体はMajsoul RPCをアカウント認証で取得。protobuf→mjai変換。定期的なRPC再接続 |
| S2 | github.com/Equim-chan/tensoul | 2026-07-26 | 1,2 | 高 | 事実 | 雀魂→tenhou.net/6。MIT。CLI/HTTPサーバ。`GameMgr.Inst.access_token` をdevtoolsで取得。標準ルール以外はundefined behavior |
| S3 | github.com/honvl/Majsoul-to-NAGA | 2026-07-26 | 1 | 高 | 事実 | `window.WebSocket` フックで `fetchGameRecord` を捕捉。MIT。**「雀魂はもうGameMgr/appグローバルを公開していない」と明記**。4人麻雀のみ |
| S4 | github.com/Equim-chan/mjai-reviewer README | 2026-07-26 | 1,2 | 高 | 事実 | Apache-2.0。tenhou.net/6を入力。雀魂は「Webアプリを使え」と誘導 |
| S5 | github.com/MahjongRepository/mahjong_soul_api | 2026-07-26 | 1 | 高 | 事実 | 雀魂protobufラッパ。CNのみuser/pass、**EN/JPは自前実装せよと明記** |
| S6 | 検索結果（pkg.go.dev EndlessCheng/mahjong-helper 他） | 2026-07-26 | 1 | 中 | 事実 | `FetchGameRecord` はWebSocketClient APIのメソッド。uid/access_tokenはpassport.mahjongsoul.comのログイン応答から |
| S7 | github.com/4n3u/majsoul-monthTicket-auto | 2026-07-26 | 1,4 | 中 | 事実 | **JPサーバ向け**。ブラウザからUID/TOKENを抜きGitHub Actionsで自動実行。「BAN等は自己責任」。**token期限の記述なし** |
| S8 | github.com/kbkn3/MahjongSoul-review-supporter | 2026-07-26 | 1 | 高 | 事実 | Chrome/Edge拡張。Apache-2.0。WebSocket傍受→天鳳形式→NAGA/mjai-reviewer |
| S9 | github.com/ssttkkl/tensoul-py + commits | 2026-07-26 | 1,2 | 高 | 事実 | Python実装。最終コミット2025-08-21。**CNアカウントのみ**。直前の修正が「加槓イベント欠落」「パオ計上」 |
| S10 | github.com/topics/majsoul (updated順) | 2026-07-26 | 1 | 高 | 事実 | amae-koromo 2026-07-24、scripts 07-25（MIT）、mahjong_soul_api 2026-06-18。protobuf/収集系は現役 |
| S11 | github.com/jeff39389327/MajsoulPaipuConvert | 2026-07-26 | 1,2 | 中 | 事実 | UUID収集→本体DL→mjai/tenhou.net/6。**config.envにメール＋パスワード、CNのみ**。4人麻雀のみ |
| S12 | github.com/Cryolite/majsoul-rpa , Apricot-S/majsoulrpa | 2026-07-26 | 1,4 | 中 | 事実/意見 | RPA。BOT参加が明示的に許可された場が前提、段位戦は非対応。損害は自己責任 |
| S13 | github.com/zyr17/MajsoulPaipuAnalyzer | 2026-07-26 | 1 | 中 | 事実 | 実在確認。三麻・特殊ルールで想定外挙動 |
| S14 | github.com/Equim-chan/tensoul/issues | 2026-07-26 | 1,2 | 中 | 事実 | #17「KeyError: 'region_urls'」2025-09-05未解決。#10「recent changes to MahjongSoul API」オープン。#14 ポン巡目が曖昧だと変換不正（クローズ）。※#17はリポ取り違えの疑い |
| S15 | github.com/kbkn3/MahjongSoul-review-supporter/issues/14 | 2026-07-26 | 1,3 | 高 | 事実 | Mortal連携でGame URLが渡らない。以前は動作→URL途中切れ→未転送 |
| S16 | 公式告知（x.com/MahjongSoul_JP 他） | 2026-07-26 | 1,4 | 中 | 事実 | 3月10日メンテ後、**Yostar IDログインが必須**に。引継コードを一斉配布。DMM版は従来通り |
| S17 | developers.cloudflare.com/workers/runtime-apis/websockets/ | 2026-07-26 | 3 | 高 | 事実 | Workersは`fetch()`+`Upgrade: websocket`でクライアントWSを張れる（グローバル`WebSocket`は不可） |
| S18 | github.com/Cryolite/mjai (+raw README) | 2026-07-26 | 2 | 高 | 事実 | mjai標準化。JSONベース、in-game/replayの2モード、牌表記`7s``5pr``?`。`/schema`にJSON Schema。**hora/ryukyoku等は(TODO)で未完成** |
| S19 | github.com/mthrok/tenhou-log-utils `parser.py` | 2026-07-26 | 2 | 中 | 事実 | 天鳳形式の鳴きはビット演算エンコード。天鳳のJS実装から起こした＝**公式仕様書が無いことの傍証** |
| S20 | 検索結果（tenhou.net/6 format） | 2026-07-26 | 2 | 低 | 事実 | 公式JSON仕様書は見つからず。OSS実装経由でのみ文書化 |
| S21 | github.com/tomohxx/mjai-gateway , NikkeTryHard/tenhou-to-mjai | 2026-07-26 | 2 | 中 | 事実 | mjai↔天鳳プロトコル変換、天鳳ログ→mjai変換。相互変換の道はある |
| S22 | support.apple.com/guide/shortcuts/ ＋ Apple Community/Developer Forums | 2026-07-26 | 3 | 高 | 事実 | POSTでJSONボディ送信可。共有シートONで入力が第1アクションに渡る。既知の罠：Base64の76文字改行、Content-Type無視、ヘッダのコロン重複で接続断 |
| S23 | jongsta.com（牌譜検討URL発行手順）, nekomimi.ws | 2026-07-26 | 3 | 低 | 事実/意見 | 対局後「シェア」からURL発行。スマホは長押しコピー、一発コピーボタンは無い。リンクがあれば誰でも閲覧可 |
| S24 | 雀魂 利用規約 第11条（ミラー＋検索スニペット。**本文取得は403で失敗**） | 2026-07-26 | 4 | 中 | 事実 | 禁止行為に「リバースエンジニアリング」「運用・利用を妨げる行為」「**各種コンテンツを不正な方法で取得する行為、又はこれを助長する行為**」 |
| S25 | 公式告知（x.com/MahjongSoul_JP） | 2026-07-26 | 4 | 中 | 事実 | 規約違反アカウントへの停止・永久停止措置を告知。「外部ツール等の使用により**対局の公平性を損なう行為**」を名指し |
| S26 | note.com/yosuke_murasame/n/ncc4a25093334 | 2026-07-26 | 4 | — | 未検証 | 「誤BANされる一例の考察」。**403で本文取得できず、内容は未確認** |

---

## れんに返す判断材料（3行）

- **技術的には作れる。** tensoul系が既に同じことをやっている。ただし JP ログインは自作、token は手動投入・期限不明。
- **規約は黙認されているだけで、白ではない。** 第11条「不正な方法でのコンテンツ取得」に当たる読みが自然。**BANされるのはれんのアカウント。ここは L3（ren 判断）。**
- **次に潰すべき最短の1点は `mjai.ekyu.moe` が雀魂URLを直接受けて tenhou/6 JSON を落とせるか。** Yes なら作るのは iOSショートカット1本で済み、token も規約リスクも自前で背負わずに済む。**れんがブラウザで開けば30秒で分かる。**
