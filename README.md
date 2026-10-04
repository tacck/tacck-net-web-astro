# tacck-net-web-astro

[tacck.net](https://tacck.net) のソースです。
[Astro](https://astro.build) と
[`@prosefly/astro-theme-lotus`](https://github.com/prosefly/astro-theme-lotus)
で構築しています。

旧サイト（VuePress 1、[tacck/tacck-net-web](https://github.com/tacck/tacck-net-web)）
から移行中です。

## Project Structure

```text
src/
  content.config.ts      Lotus の docs コレクションを登録
  content/docs/          ページ（MDX）
theme.config.json        サイト名、ナビゲーション、検索、フッターなどの設定
```

`docsBase` は `/` なので、`src/content/docs/index.mdx` がトップページ（`/`）になります。

## Commands

| Command | Description |
| --- | --- |
| `pnpm dev` | 開発サーバーを起動 |
| `pnpm build` | 本番用にビルド（`dist/` に出力） |
| `pnpm preview` | ビルド結果をローカルで確認 |
| `pnpm check` | Astro の型・コンテンツチェック |

## Deploy

AWS Amplify Hosting でデプロイします（設定は移行作業中）。
