---
name: j-jev
description: TypeSafe AI の Jev（typesafe-ai/jev）を Vercel AI Gateway 経由または TypeSafe の直接 API で使う実装・設計・デバッグ時に使う。API の形と制約、経路ごとの違い（CORS・形式・料金）、ハマりどころ、429/503 の実測傾向（一時的な値）と向き合い方をまとめる。JEV / Jev / typesafe-ai/jev / jev-latest / AI Gateway の evaluate API / TypeSafe の systemone API / Choice・Score・Noul を扱う時に読むこと。
---

# JEV（typesafe-ai/jev）を使うときの知識

JEV はテキストを生成せず、「状態（state）」と「型付きの質問」を受け取って確率付きの判断を返すモデル。実装と実測で得た知見をまとめる。

## 経路は 2 つ

| 経路 | エンドポイント | モデル名 | キー | ブラウザから直接 |
|---|---|---|---|---|
| Vercel AI Gateway | `POST https://ai-gateway.vercel.sh/v1/evaluate` | `typesafe-ai/jev` | `AI_GATEWAY_API_KEY` | 呼べる（CORS 許可。`retry-after` / `x-should-retry` も読める） |
| TypeSafe の直接 API | `POST https://api.typesafe.ai/v1/systemone` | `jev-latest` | `TYPESAFE_API_KEY` | **呼べない**（CORS 不可。プリフライトが 400） |

- どちらも `Authorization: Bearer <key>`。
- 経路で形式・料金・混雑の傾向が違うので、既定の経路を暗黙に決めず、どちらを使うかを明示する設計にする。
- ブラウザから直接 API を使うときは、CORS に答える透過プロキシを挟む。プロキシは `OPTIONS` に答え、`Access-Control-Expose-Headers` で `retry-after` / `retry-after-ms` などを公開する必要がある。
- CORS で拒否されると、ブラウザではネットワーク断と区別できない（`fetch` が TypeError になるだけ）。一度も応答を得ていない URL への接続失敗は、少ない回数で打ち切って設定の誤りを疑わせる。

## API の要点

- 質問の型: `choice`（criteria = {キー: 説明}、**最大 255 候補**）/ `score`（criteria = 段階の配列、2〜10）/ `boolean`（Noul = yes の確率）
- 直接 API では boolean を `noul` と呼び（回答は `{ type: 'noul', noul: 0..1 }`）、usage は snake_case（`input_tokens`）、score には `legend` が付く。Gateway は camelCase。
- state は文字列・オブジェクト・配列で、**32k トークン**まで。各質問は独立に評価される
- 料金: Gateway は入力のみ（公表 $0.042 / 1M トークン）で、応答の `providerMetadata.gateway.marketCost` が定価ベースの料金。直接 API は料金を返さない（単価・出力の課金は未確認。`output_tokens` は 0 でない値が返る）
- 上限を超えた質問はサーバーが 4xx で拒否する。仕様が変わり得るので、送信前にクライアント側でチェックしない
- 公式ドキュメント: https://vercel.com/kb/guide/typesafe-jev-and-ai-sdk ／ https://vercel.com/changelog/ai-gateway-now-supports-typesafe-clients-and-http-api-for-jev ／ https://docs.typesafe.ai/api （仕様は変わり得るので、実装前に最新を確認すること）

## ハマりどころ（実際に踏んだもの）

1. `providerOptions.gateway.zeroDataRetention: true` は **Vercel Pro 以上のみ**。Hobby だと 403 になる。
2. 静的サイトでは、ユーザー自身のキーをブラウザ内だけに保存し、CORS を許可している Gateway を直接呼ぶ。
3. 開発時のキーは `.env`（**`VITE_` を付けない**）に置き、dev サーバーの透過プロキシでサーバー側から付与する。こうするとキーがバンドルに混入しない。キーを持つ中継は、loopback にだけ bind し、localhost 以外の Origin を拒否する（他サイトから裏で送られて課金されるのを防ぐ）。
4. OpenAI 互換クライアントからは使えない。
5. 直接 API に `type: 'boolean'` を送ると 400（`noul` にする必要がある）。キー無しは 401 ではなく 403。
6. 候補が 255 を超えるときは、コード側で「良さそうな候補」に絞らず、構造（例: セクション）で階層 Choice にする（判断を JEV に残すため）。
7. 応答が返らず固まる呼び出しがまれにある（成功時の p90 は 0.5 秒前後）。時間切れで打ち切って再試行する。

## エラー（429/503）への向き合い方

- 2026-09-25 の実測では、Gateway の 429 は毎分約 30 回の上限超過として振る舞い（`retry-after` は次の分の区切りまでの秒数）、503 は上流障害で数秒単位で連続した。エラーは強く連続する（直前がエラーなら次も 98%）。同時刻の直接 API は毎分 120 回で 429・エラーとも 0 件だった（直接 API で 429 を受けたときの挙動は未確認）。
- エラーが連続するので、呼び出しごとに独立して再試行するより、全呼び出しで待機を共有する（1 件失敗したら全員が `retry-after` まで待つ）ほうがよい。流量制御の状態は経路ごとに分ける（片方の混雑で空いている経路まで止めない）。
- **これは JEV 公開直後の混雑による一時的な傾向とユーザーは見ている。固定値として実装に焼き込まない**（回数上限は 429 を受けたときだけ学習し、止めば解除する、など）。
- 傾向を前提にした判断（経路の選択など）をする前に、少量のリクエストで**再計測**する（少量なら費用は 1 円未満）。
- JEV が失敗したとき、JEV 以外（コードや他の AI）の判断で黙って代打ちしない。開発・テストで代替（ダミー・録画）を使う場合は、誰が答えたかを結果に持たせて区別し、正式な結果に混ぜない。
