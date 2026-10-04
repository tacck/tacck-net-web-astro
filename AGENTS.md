## Development

When starting the dev server, use background mode:

```
astro dev --background
```

Manage the background server with `astro dev stop`, `astro dev status`, and `astro dev logs`.

## Documentation

Full documentation: https://docs.astro.build

Consult these guides before working on related tasks:

- [Adding pages, dynamic routes, or middleware](https://docs.astro.build/en/guides/routing/)
- [Working with Astro components](https://docs.astro.build/en/basics/astro-components/)
- [Using React, Vue, Svelte, or other framework components](https://docs.astro.build/en/guides/framework-components/)
- [Adding or managing content](https://docs.astro.build/en/guides/content-collections/)
- [Adding styles or using Tailwind](https://docs.astro.build/en/guides/styling/)
- [Supporting multiple languages](https://docs.astro.build/en/guides/internationalization/)

## 協調開発ルール（coord-dev-mcp）

このリポジトリは複数の AI エージェントが同時に開発する。衝突を避けるため、
`coord-dev` MCP サーバのツールで調整すること。あなたの `agent_id` は **`<AGENT_ID>`**。
すべてのツール呼び出しでこの値を `agent_id` に渡す。

### 作業を始めるとき（必ずこの順）

1. `list_open_tasks` で未割当タスクと各タスクの `version` を取得する。
2. 着手するタスクを `claim_task(task_id, agent_id, expected_version)` で占有する。
   `version` には手順 1 で読んだ値を渡す。
   - 戻り値が `ok:false`（`conflict`）なら、他のエージェントが先に取った。
     **そのタスクは諦め、別の open タスクへ回る**。再試行で奪わないこと。
3. 占有できたら `assign_worktree(agent_id, repo_path, task_id)` を呼び、
   返ってきた **worktree のパスの中だけで作業する**。リポジトリ本体の作業ツリーは
   直接編集しない（物理隔離が協調の前提）。`repo_path` はこのリポジトリの絶対パス。

### 作業中

- ファイルを編集したら、こまめに
  `publish_dirty(agent_id, worktree_path, files)` で変更したファイルを公開する。
  戻り値の `overlaps` に他エージェントが出たら、**同じファイルを別の人も触っている**。
  その場で `post_board` か（相手が待っていれば自動で起きる）で調整し、
  手戻りが大きくなる前にすり合わせる。
- 相手に伝えたいこと・引き継ぎは `post_board(agent_id, body, kind, to_agent)` を使う。
  特定の相手に渡すときは `kind:"handoff"` と `to_agent` を指定する。
- 長めに待つなら手を止めて空ポーリングせず、`wait_for_unblock` を使う（下記）。

### 作業を終えるとき

4. worktree 上で変更を **コミットしてから**
   `task_done(agent_id, worktree_path, branch, repo, task_id)` を呼ぶ。
   `repo` はこのリポジトリの絶対パス、`branch` は worktree のブランチ名。
   - 衝突候補が無ければ自動で本流にマージされる。
   - 戻り値が `needs_review` なら、本流が同じ箇所を変えている。
     `list_merge_queue(state:"needs_review")` で確認し、worktree 上で
     コンフリクトを解消・再コミットしてから **もう一度 `task_done`** を呼ぶ。
     勝手に強制マージしないこと。

### 手が空いたとき

5. 次の仕事を探す前に `wait_for_unblock(agent_id)` を 1 回呼ぶ。
   「待っていた依存タスクが完了した / 自分宛の連絡が来た / 自分の担当領域に
   他エージェントが侵入した / タイムアウト」のいずれかで戻り、理由が `woke_by`
   に入る。空ループで `list_open_tasks` を叩き続けないこと。

### 守ること（要点）

- `agent_id` は常に `<AGENT_ID>`。他のエージェントの id を名乗らない。
- claim できなかったタスクは奪わず、別のタスクへ。
- 編集は必ず割り当てられた **worktree の中**で。本流ツリーを直接触らない。
- `overlaps` / `needs_review` は「止まれ」ではなく「人間と相談して調整しろ」の合図。
  自動で握りつぶさない。
