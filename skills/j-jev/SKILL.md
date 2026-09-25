---
name: j-jev
description: TypeSafe AI の Jev（typesafe-ai/jev）を Vercel AI Gateway 経由または TypeSafe の直接 API で使う実装・設計・デバッグ時に使う。API の形と制約、ハマりどころ、429/503 の実測傾向（一時的な値）、再利用できる SDK（@jumboly/jev-client）の使い方をまとめる。JEV / Jev / typesafe-ai/jev / jev-latest / AI Gateway の evaluate API / TypeSafe の systemone API / Choice・Score・Noul を扱う時に読むこと。
---

# JEV（typesafe-ai/jev）を使うときの知識

JEV はテキストを生成せず、「状態（state）」と「型付きの質問」を受け取って確率付きの判断を返すモデル。SDK（@jumboly/jev-client）の実装と実測で得た知見をまとめる。

## まず SDK を使う

自前で fetch を書かず、**@jumboly/jev-client** を使う（2 経路対応・流量制御・時間切れ・コスト集計・ダミー／録画再生／代替の連結・計測ツールが揃っている）。

- ソース: `~/src/jev-client`（GitHub: https://github.com/jumboly/jev-client 。npm には公開しない）
- インストール: `npm install github:jumboly/jev-client#v0.1.0`（版を固定）。並行開発中は `npm install ../jev-client`（symlink なので、先にそちらで `npm run build`）
- 詳細な使い方・API の要点は、その **README.md が正本**。まず読むこと。
- 実測結果（混雑の傾向・CORS・遅延）の正本は `docs/probe-results.md`。

## 経路は 2 つ（`mode` は必須）

SDK は経路ごとの形式の違いを吸収し、回答はどちらも同じ形（gateway 形式）で返す。経路で形式・料金・混雑の傾向が違うため、既定の経路は持たない。旧 `key` / `proxy` モードは廃止した。

| `mode` | エンドポイント | モデル名 | キー | ブラウザから直接 |
|---|---|---|---|---|
| `gateway` | `POST https://ai-gateway.vercel.sh/v1/evaluate` | `typesafe-ai/jev` | `AI_GATEWAY_API_KEY` | 呼べる（CORS 許可） |
| `typesafe` | `POST https://api.typesafe.ai/v1/systemone` | `jev-latest` | `TYPESAFE_API_KEY` | **呼べない**（CORS 不可） |

- `url` を指定すると、その経路の形式のまま透過プロキシへ送れる。`apiKey` を省くと `Authorization` を付けない（プロキシ側でキーを付与する場合）。
- ブラウザから `typesafe` を使うときは、CORS に答える透過プロキシを `url` に指定する。プロキシは `OPTIONS` に答え、`Access-Control-Expose-Headers` で `retry-after` などを公開する必要がある。
- CORS で拒否されると、ブラウザではネットワーク断と区別できない。SDK は一度も応答を得ていない URL への接続失敗を 3 回で打ち切る。
- 流量制御は経路ごとに別（`defaultGates[mode]`。`defaultGate` は gateway 用）。
- 回答の `provider` が実際の経路、`source` が「JEV の判断か（jev / replay / mock）」。

## API の要点

- 質問の型: `choice`（criteria = {キー: 説明}、**最大 255 候補**）/ `score`（criteria = 段階の配列、2〜10）/ `boolean`（Noul = yes の確率）
- typesafe の直接 API では boolean を `noul` と呼び、usage は snake_case、score には `legend` が付く（SDK が変換する）。
- state は文字列・オブジェクト・配列で、**32k トークン**まで。各質問は独立に評価される
- 料金: gateway は入力のみ（公表 $0.042 / 1M トークン）で、応答の `providerMetadata.gateway.marketCost` が定価ベースの料金。typesafe は料金を返さないので SDK は公表単価からの概算を出す（直接 API の単価・出力の課金は未確認）
- 上限を超えた質問はサーバーが 4xx で拒否する。仕様が変わり得るので送信前にチェックしない
- 公式ドキュメント: https://vercel.com/kb/guide/typesafe-jev-and-ai-sdk ／ https://vercel.com/changelog/ai-gateway-now-supports-typesafe-clients-and-http-api-for-jev ／ https://docs.typesafe.ai/api （仕様は変わり得るので、実装前に最新を確認すること）

## ハマりどころ（実際に踏んだもの）

1. `providerOptions.gateway.zeroDataRetention: true` は **Vercel Pro 以上のみ**。Hobby だと 403 になる。
2. 静的サイトでは、ユーザー自身のキーをブラウザ内だけに保存し、CORS を許可している gateway を直接呼ぶ。
3. 開発時のキーは `.env`（**`VITE_` を付けない**）に置き、dev サーバーの透過プロキシでサーバー側から付与する（SDK には `url` だけ渡し `apiKey` を省く）。こうするとキーがバンドルに混入しない。
4. OpenAI 互換クライアントからは使えない。
5. typesafe の直接 API に `type: 'boolean'` を送ると 400（`noul` にする必要がある）。キー無しは 401 ではなく 403。
6. 候補が 255 を超えるときは、コード側で「良さそうな候補」に絞らず、構造（例: セクション）で階層 Choice にする（判断を JEV に残すため）。

## エラー（429/503）への向き合い方

- 2026-09-25 の実測では、gateway の 429 は毎分約 30 回の上限超過として振る舞い、503 は上流障害で数秒単位で連続した。エラーは強く連続する（直前がエラーなら次も 98%）。同時刻の typesafe は毎分 120 回で 429・エラーとも 0 件だった（typesafe で 429 を受けたときの挙動は未確認）。
- **これは JEV 公開直後の混雑による一時的な傾向とユーザーは見ている。固定値として実装に焼き込まない**（SDK の上限は auto = 429 のときだけ学習し、止めば解除）。
- 傾向を前提にした判断（経路の選択など）をする前に、SDK の `npm run probe -- --mode gateway|typesafe` で**再計測**する（少量なら費用は 1 円未満）。
- JEV が失敗したとき、JEV 以外（コードや他の AI）の判断で黙って代打ちしない。開発・テストで代替を使う場合は、回答の `source` で区別して正式な結果に混ぜない。
