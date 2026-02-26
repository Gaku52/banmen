# 盤面 (Banmen) アプリ設計書 v0.1

## 1. 目的

「今やること」を 3 秒で判断できる状態を維持するための、極小タスク盤面を提供する。

- NOW: 1 件
- NEXT: 最大 3 件
- 完了した項目は即時消去
- 履歴・通知・分析は持たない

## 2. 対象ユーザー

- タスク管理アプリが重くなって離脱した経験がある人
- ADHD 傾向や疲労時に「次に何をするか」で詰まりやすい人
- 開発・学習・事務などで短い集中サイクルを回したい人

## 3. MVP 範囲 (Phase 1)

- PWA 単体 (Web + ホーム画面インストール)
- ローカル保存のみ (サーバーなし)
- 1 画面完結 UI
- 日本語 UI 優先、英語は将来対応

### やること

- タスク追加 (NEXT に追加、上限 3)
- タスク編集 (NOW/NEXT 文字列のみ)
- NEXT -> NOW への昇格
- NOW 完了時の削除アニメーション
- NEXT の並び替え (ドラッグまたは上下移動)
- すべて消去 (確認ダイアログ付き)

### やらないこと

- アカウント
- クラウド同期
- 履歴/統計
- 通知/締切
- タグ/カテゴリ/プロジェクト

## 4. 主要ユースケース

1. アプリを開くと NOW/NEXT が同一画面に表示される
2. NEXT に 1 件追加する
3. NEXT から 1 件を NOW に移す
4. NOW 完了で項目が消える
5. 必要時のみ NEXT から次を選ぶ

## 5. 画面設計

## 5.1 ホーム (唯一のメイン画面)

- ヘッダー: 「盤面」
- NOW セクション: 大きなカード 1 枠
- NEXT セクション: 小さめカード最大 3 枠
- 入力導線: 下部固定の `+ 追加`
- 補助操作: `並び替え`, `全消去` (目立たせない)

### 表示ルール

- NOW が空なら「NEXT から 1 つ選ぶ」を表示
- NEXT が 0 件ならプレースホルダー表示
- NEXT が 3 件のとき追加ボタン無効

## 5.2 モーダル

- 追加/編集モーダル: 1 行テキスト入力、文字数上限 80
- 全消去確認: 「消す/キャンセル」

## 6. 操作ルール (状態遷移)

状態:
- `nowTask: Task | null`
- `nextTasks: Task[]` (0..3)

遷移:
1. `addTask(text)` -> `nextTasks.push(task)` (3 件未満のみ)
2. `promoteTask(taskId)`  
   - `nowTask` が空: 対象を `nextTasks` から取り出して `nowTask` に設定  
   - `nowTask` が埋まっている: 入れ替え確認後、NOW を NEXT 先頭に戻して昇格
3. `completeNow()` -> フェードアウト後 `nowTask = null`
4. `reorderNext(from,to)` -> 配列順変更
5. `clearAll()` -> `nowTask = null`, `nextTasks = []`

## 7. データモデル

```ts
type Task = {
  id: string;            // uuid
  text: string;          // 1..80
  createdAt: string;     // ISO8601
  updatedAt: string;     // ISO8601
};

type BanmenState = {
  nowTask: Task | null;
  nextTasks: Task[];     // max 3
  version: 1;
};
```

保存:
- `localStorage` キー: `banmen.state.v1`
- 書き込みタイミング: 状態変更時に即保存 (debounce 100ms)
- 起動時復元: JSON parse 失敗時は初期化

## 8. 非機能要件

- 初回表示: 1.5 秒以内 (4G 相当)
- 操作応答: 100ms 以内
- Lighthouse (PWA): 90 以上
- オフライン起動可能
- データ送信なし (Analytics も MVP では無効)

## 9. 技術設計 (Phase 1)

- フレームワーク: Next.js (App Router, TypeScript)
- UI: React + CSS Modules (軽量優先)
- 状態管理: React state + custom hook (`useBanmenState`)
- PWA: `next-pwa` + manifest + service worker
- デプロイ: Cloudflare Workers (`@opennextjs/cloudflare`)
- ドメイン: Cloudflare Registrar

推奨ディレクトリ:

```txt
app/
  page.tsx
  layout.tsx
  globals.css
components/
  NowCard.tsx
  NextList.tsx
  TaskModal.tsx
  ConfirmDialog.tsx
hooks/
  useBanmenState.ts
lib/
  storage.ts
types/
  banmen.ts
```

## 10. 将来拡張の境界

MVP はローカル完結だが、将来 Bot 連携に備えて「ユースケース関数」を UI から分離する。

例:
- `addTask(text)`
- `promoteTask(taskId)`
- `completeNow()`
- `clearAll()`

この関数群を API 化すれば、LINE/Slack/Teams から同じドメインロジックを利用できる。

## 11. 受け入れ基準 (MVP 完了条件)

1. NOW 1 件 + NEXT 最大 3 件制約が常に守られる
2. 完了した NOW は履歴を残さず消える
3. リロード後に現在の盤面だけ復元される
4. 追加から完了までを 10 秒以内で操作できる
5. 画面を開いて 3 秒以内に「今やること」が判別できる

## 12. 実装順序

1. 静的 UI (NOW/NEXT レイアウト)
2. 状態管理 hook と localStorage
3. 追加/編集/昇格/完了の操作実装
4. PWA 化 (manifest, service worker, install)
5. モバイル表示最適化と軽量化
