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

## 協調開発ルール（lockage）

このリポジトリは複数の AI エージェントが同時に開発する。衝突を避けるため、
`lockage` MCP サーバのツールで調整すること。あなたの `agent_id` は **`<AGENT_ID>`**。
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
   - このツールが返す `worktree_path` と `branch` が、**あなたの作業場所と作業ブランチ**。
     以降の編集・コミットはすべてこの worktree・このブランチ上で行う。
   - **自分でその worktree に移動（CWD 切り替え）できない場合**（起動時に作業ディレクトリを
     渡されていない・GUI で開き直せない等）は、**本流ツリーで作業を始めてはいけない**。
     `post_board(agent_id, kind:"question", to_agent:"<ORCHESTRATOR_ID>", ref_task_id:<task_id>,
     body:"worktree <worktree_path>（ブランチ <branch>）に切り替えて作業を開始したいが、
     自分で移動できない。この worktree で開き直してほしい")` でユーザー（オーケストレータ）に
     依頼し、`wait_for_unblock` で待つ。本流を直接触る事故を防ぐため、切り替わるまで着手しない。

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
   - 衝突候補が無ければ自動で本流にマージされ、**そのタスクは `done` になる**
     （`task_id` を渡した場合）。マージ成功と同時に done になるので、
     別途ステータスを更新する必要はない。
   - 戻り値が `needs_review` なら、本流が同じ箇所を変えている。タスクは
     `done` にならず `in_progress` のまま残る。`list_merge_queue(state:"needs_review")`
     で確認し、worktree 上でコンフリクトを解消・再コミットしてから
     **もう一度 `task_done`** を呼ぶ。勝手に強制マージしないこと。
   - 戻り値が `ok:false`（`reason` 付き）なら呼び方が誤っている。マージは走らない:
     - `no_active_worktree`: その `worktree_path` に自分の active な割り当てが無い。
       手順 3 の `assign_worktree` が返したパスを渡すこと（リポジトリ本体を渡さない）。
     - `branch_is_base`: `branch` が base ブランチ（main 等）そのもの。worktree の
       ブランチ名を渡すこと。
     - `not_assignee`: その `task_id` を claim しているのは別のエージェント。
       自分が claim したタスクだけを done にできる。

5. **Pull Request は「イシュー単位」で自動化される。あなたは手で PR を作らない。**
   タスクはイシュー（まとまった機能・課題）を分割した片なので、**同じ `issue_id` の
   全タスクが `done` になると、その issue 全体が自動で 1 本の PR として publish される**。
   個々のエージェントがやることは「自分の担当タスクを `task_done` まで回す」ことだけ。
   - あるタスクを done にしたら、`list_tasks`（同じ `issue_id` で状態確認）で
     **同じ issue の残タスクがまだあるか**を見る。残っていればまだ何もしなくてよい——
     次の open タスクへ進む。
   - あなたが同一 issue の**最後の 1 つ**を done にした瞬間、サーバ側が自動で
     `issue_ready`（この issue は PR 化してよい）を board に発行する（全 done 検知。
     二重発行しない）。**ここで `gh pr create` を自分で叩く必要はない**。
   - `issue_ready` を受けて実際に統合ブランチをリモートへ出し PR を作るのは、
     別プロセスの **`lockage-publish`**（cron 等で定期起動、または調整役が手動起動）。
     これが未処理の `issue_ready` を拾って push → PR 作成し、マージ済み PR の統合ブランチ掃除まで
     行う。PR のタイトル・本文・issue 番号の紐付けも publisher が付ける。
   - したがって**あなたは PR 作成を依頼する必要すらない**。例外は「この環境で
     `lockage-publish` が全く走っていない」と分かっている場合だけで、そのときのみ
     `post_board(kind:"question", to_agent:"<ORCHESTRATOR_ID>", ref_task_id:<task_id>,
     body:"issue #N の全タスク完了。lockage-publish の起動（PR 化）を依頼")` で調整役に起動を頼む。

> **補足（現状の実装）**: `task_done` は worktree の成果を**ローカルで base ブランチへ直接マージ**し、
> issue 配下が全 done になると `issue_ready` を発行します（問題A）。その `issue_ready` を
> `lockage-publish` が受けて**統合ブランチの push → PR 作成 → マージ後の掃除**を自動で行います（問題B、PR #9 で導入済み）。
> つまり「個人エージェントは `task_done` まで、PR 化は publisher が自動」という運用がすでに動きます。
> なお `task_done` 本体の直マージを「PR マージ経由」に差し替える #6 は別課題として継続中ですが、
> **PR が自動で作られること自体は問題B で実現済み**です。

### 完了できないとき（中断する — 黙って終わらない）

権限不足・設計判断待ち・前提の欠落などで**タスクを完了できないと判断したら**、
そのまま終了（exit）してはいけない。短命セッションのあなたが黙って消えると、
オーケストレータは「作業中なのか死んだのか」を区別できず、15 分後に lease が切れて
タスクは `open` に戻され、**あなたが詰まった理由も、途中まで進めた worktree の文脈も失われる**。
必ず次の順で「詰まった事実を台帳に書いてから」終わること。

1. **先に理由を残す**: `post_board(agent_id, kind:"question", to_agent:"<ORCHESTRATOR_ID>", ref_task_id:<task_id>, body:"詰まった理由（例: back の API キー権限が無い / レスポンス形式の設計判断が要る）")`。
   必ず `block_task` より**先**に呼ぶ（次手で lease が止まるため、理由を書く前に死ぬと詰まり方が不明になる）。
2. **タスクを中断状態にする**: `block_task(task_id, agent_id)`。これで `in_progress`→`blocked` になり、
   あなたの claim・worktree は**保持されたまま温存**される（lease sweep の対象外＝無期限保持。
   機械は勝手に戻さない）。
3. **worktree はそのまま残して exit する**。`release_worktree` を呼ばない（途中成果を消さない。
   再開時にそのまま続けられる）。

この後の再開は人間（オーケストレータ）が決める:
- 「続行」なら、同じ `agent_id` で起動されたあなたが `resume_task(task_id, agent_id)` を呼び、
  `blocked`→`in_progress` に戻して worktree の続きから進める。起動直後は記憶が無いので、
  `read_board(kind:"question", ref_task_id)` と `search_context` で**自分（前任）が残した
  詰まり理由と作業文脈を読み直してから**続ける。
- 「別の worker に回す」と人間が明示した場合だけ `handoff_task` で別 agent へ一括移管される。
  これはあなたが呼ぶものではない（人間ゲート）。

### 手が空いたとき

5. 次の仕事を探す前に `wait_for_unblock(agent_id)` を 1 回呼ぶ。
   「待っていた依存タスクが完了した / 自分宛の連絡が来た / 自分の担当領域に
   他エージェントが踏み込んだ / タイムアウト」のいずれかで戻り、理由が `woke_by`
   に入る。空ループで `list_open_tasks` を叩き続けないこと。
   - `woke_by:"board"` で戻り、`payload` に `kind:"handoff"` のメッセージがあれば、
     **別の worker から仕事を引き継いだ**合図。`ref_task_id` のタスクはもう自分が claim 済み・
     worktree も自分に移っている（`list_tasks(assignee, status:"in_progress")` で確認できる）。
     前任が残した `question`（詰まり理由）を `read_board` で読み、worktree の続きから進める。

### 守ること（要点）

- `agent_id` は常に `<AGENT_ID>`。他のエージェントの id を名乗らない。
- claim できなかったタスクは奪わず、別のタスクへ。
- 編集は必ず割り当てられた **worktree の中**で。本流ツリーを直接触らない。
  worktree に自分で移動できないときは、本流で始めず `post_board(kind:"question")` で
  ユーザーに切り替えを依頼して待つ。
- Pull Request は **イシュー単位で自動化**。あなたは手で PR を作らない。自分の担当タスクを
  `task_done` まで回し、done のたびに同一 issue の残タスクを確認する。最後の 1 つを done に
  すると自動で `issue_ready` が立ち、`lockage-publish`（別プロセス）が PR を作る。
- `overlaps` / `needs_review` は「止まれ」ではなく「人間と相談して調整しろ」の合図。
  自動で握りつぶさない。
- 完了できないときは黙って終わらない。必ず `post_board(kind:"question")` で理由を残して
  から `block_task` で中断し、worktree は残す（この順を守る）。
