# 世界ランキングのセットアップ（Supabase）

世界ランキングは [Supabase](https://supabase.com)（無料枠あり）を使います。
以下のセットアップが終わるまで、ゲームは世界ランキング非表示のまま（ローカルランキングのみ）で動きます。

## 手順

### 1. Supabaseプロジェクトを作る

1. https://supabase.com にアクセスし、GitHubアカウントなどでサインアップ
2. **New Project** でプロジェクトを作成（名前は `irooni` など何でもOK、リージョンは `Northeast Asia (Tokyo)` がおすすめ）
3. データベースパスワードを設定（このパスワードはゲームでは使いません。控えておくだけでOK）

### 2. スコア用テーブルを作る

プロジェクトのダッシュボードで **SQL Editor** を開き、以下を貼り付けて **Run**：

```sql
-- スコアテーブル
create table public.scores (
  id         bigint generated always as identity primary key,
  name       text not null check (char_length(name) between 1 and 10),
  score      integer not null check (score between 0 and 10000000),
  level      integer not null check (level between 1 and 10000),
  combo      integer not null check (combo between 0 and 10000),
  dev        integer not null check (dev between 0 and 100),
  title      text not null check (char_length(title) between 1 and 20),
  created_at timestamptz not null default now()
);

-- ランキング取得用インデックス
create index scores_score_desc on public.scores (score desc, created_at asc);

-- 匿名ユーザーは「登録」と「閲覧」だけ可能（書き換え・削除は不可）
alter table public.scores enable row level security;
create policy "anon insert" on public.scores for insert to anon with check (true);
create policy "anon select" on public.scores for select to anon using (true);
```

### 3. 接続情報をゲームに書き込む

1. ダッシュボードの **Project Settings → API** を開く
2. **Project URL**（`https://xxxx.supabase.co` 形式）と **anon public** キーをコピー
3. `index.html` の先頭付近にある以下の行に貼り付ける：

```js
window.IROONI_SUPABASE = {
  url: 'https://xxxx.supabase.co',      // ← Project URL
  anonKey: 'eyJhbGciOi...',             // ← anon public キー
};
```

4. コミットしてGitHub Pagesに反映されれば、タイトル画面のランキングに「世界」タブが現れます

> **anonキーは公開して大丈夫？** — はい。anonキーはブラウザに埋め込む前提の公開キーで、
> できることは上のRLSポリシーで許可した「スコアの登録と閲覧」だけです。
> `service_role` キーは絶対に貼らないでください。

## 仕組みと制限

- スコア送信・取得はSupabaseのREST API（PostgREST）を直接呼びます。SDKやビルドは不要です
- リザルト画面で名前（10文字まで）を入れて「世界ランキングに登録」を押すと送信されます（1プレイ1回）
- 名前はブラウザに保存され、次回から自動入力されます
- クライアントだけのゲームなので、原理的に不正スコア送信は防げません（カジュアルなランキングとして割り切っています）。
  荒らされた場合はダッシュボードの **Table Editor** から行を削除してください
