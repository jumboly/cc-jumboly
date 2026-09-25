---
name: j-jev
description: TypeSafe AI の Jev（typesafe-ai/jev）を Vercel AI Gateway 経由で使う実装・設計・デバッグ時に使う。API の形と制約、ハマりどころ、429/503 の実測傾向（一時的な値）、再利用できる SDK（@jumboly/jev-client）の使い方をまとめる。JEV / Jev / typesafe-ai/jev / AI Gateway の evaluate API / Choice・Score・Noul を扱う時に読むこと。
---

# JEV（typesafe-ai/jev）を使うときの知識

JEV はテキストを生成せず、「状態（state）」と「型付きの質問」を受け取って確率付きの判断を返すモデル。wikipedia-geo-runner（JEV Geo Race）で実装・実測した知見をまとめる。

## まず SDK を使う

自前で fetch を書かず、**@jumboly/jev-client** を使う（流量制御・時間切れ・コスト集計・ダミー／録画再生／代替の連結・計測ツールが揃っている）。

- ソース: `~/src/jev-wiki-geo-runner/packages/jev-client/`（GitHub: https://github.com/jumboly/wikipedia-geo-runner/tree/main/packages/jev-client）
- 詳細な使い方・API の要点・実測結果は、その **README.md が正本**。まず読むこと。
- 現状はリポジトリ内のパッケージ。2 つ目のプロジェクトで使うことになったら、別リポジトリへの切り出しをユーザーに提案する（npm 公開の有無も確認）。

## API の要点

- `POST https://ai-gateway.vercel.sh/v1/evaluate`、`model: "typesafe-ai/jev"`、`Authorization: Bearer <AI_GATEWAY_API_KEY>`
- 質問の型: `choice`（criteria = {キー: 説明}、**最大 255 候補**）/ `score`（criteria = 段階の配列、2〜10）/ `boolean`（Noul = yes の確率）
- state は文字列・オブジェクト・配列で、**32k トークン**まで。各質問は独立に評価される
- 料金は入力のみ（公表 $0.042 / 1M トークン）。応答の `providerMetadata.gateway.marketCost` が定価ベースの料金
- 公式ドキュメント: https://vercel.com/kb/guide/typesafe-jev-and-ai-sdk ／ https://vercel.com/changelog/ai-gateway-now-supports-typesafe-clients-and-http-api-for-jev （仕様は変わり得るので、実装前に最新を確認すること）

## ハマりどころ（実際に踏んだもの）

1. `providerOptions.gateway.zeroDataRetention: true` は **Vercel Pro 以上のみ**。Hobby だと 403 になる。
2. **ブラウザから直接呼べる**（AI Gateway が CORS を許可。`retry-after` / `x-should-retry` も読める）。静的サイトでは、ユーザー自身のキーをブラウザ内だけに保存する方式にする。
3. 開発時のキーは `.env` の `AI_GATEWAY_API_KEY`（**`VITE_` を付けない**）に置き、dev サーバーのプロキシでサーバー側から付与する（SDK の `proxy` モード）。こうするとキーがバンドルに混入しない。
4. OpenAI 互換クライアントからは使えない。
5. 候補が 255 を超えるときは、コード側で「良さそうな候補」に絞らず、構造（例: セクション）で階層 Choice にする（判断を JEV に残すため）。

## エラー（429/503）への向き合い方

- 2026-09-25 の実測では、429 は毎分約 30 回の上限超過として振る舞い、503 は上流障害で数秒単位で連続した。エラーは強く連続する（直前がエラーなら次も 98%）。
- **これは JEV 公開直後の混雑による一時的な傾向とユーザーは見ている。固定値として実装に焼き込まない**（SDK の上限は auto = 429 のときだけ学習し、止めば解除）。
- 傾向を前提にした判断をする前に、SDK の `npm run probe` で**再計測**する（少量なら費用は 1 円未満）。
- JEV が失敗したとき、JEV 以外（コードや他の AI）の判断で黙って代打ちしない。開発・テストで代替を使う場合は、回答の `source` で区別して正式な結果に混ぜない。
