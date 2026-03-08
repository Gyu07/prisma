# Mora Prisma移行教材（migrate from）

この資料は、Prisma スキーマを Mora のスキーマへ変換する `mora migrate from prisma` の学習用ガイドです。
実際のコマンド実行と、変換結果の見方を段階的に確認できます。

## 1. 前提

- `mora` がインストール済み
- Prisma スキーマ (`schema.prisma`) が準備済み
- Node.js 実行環境（`pnpm`/`npm`）で CLI が使える状態

## 2. 変換を実行する

最低限の実行コマンドは次です。

```bash
mora migrate from prisma --schema ./schema.prisma
```

出力先はデフォルトで `.mora/imports/prisma` になります。
生成されるファイルは以下です。

- `schema.mora.ts` : Mora スキーマ
- `prisma-gaps.json` : 変換できない項目の一覧
- `prisma-import.result.json` : 変換サマリー

## 3. JSON 形式で結果を取得する

CIや自動化向けには JSON 形式で実行します。

```bash
mora migrate from prisma --schema ./schema.prisma --format json
```

JSON には以下が含まれます。

- `status`: `success` / `partial-success` / `failed`
- `discovered.tableNames`: 変換対象テーブル名
- `discovered.relationDefinitions`: Prisma の relation フィールド数
- `gaps`: 自動変換しきれなかった項目
- `artifacts`: 生成ファイルのパス

## 4. シンプル変換の例

Prisma モデル:

```prisma
model User {
  id    Int     @id
  email String  @unique
  name  String?
}
```

実行後の概念例:

- `id` が `integer().primaryKey().notNull()` へ
- `email` が `text().notNull().unique('user_email_uniq')` へ
- `name` が `text()`（任意列）へ

## 5. リレーションを含む例

Prisma モデル:

```prisma
model User {
  id Int @id
}

model Post {
  id     Int @id
  userId Int
  author User @relation(fields: [userId], references: [id], onDelete: Cascade, onUpdate: NoAction)
}
```

変換上のポイント:

- `author` の relation フィールドそのものは、Mora の `table()` 定義にはスキーマ型として追加しない（実体の FK 列のみ扱う）
- `userId` に `.references(() => ({ schema: 'public', table: 'user', column: 'id' }), { onDelete: 'cascade', onUpdate: 'no action' })` が付与される
- relation そのものの高レベル設計は、`gaps` で「手動確認が必要」扱いになることがある

## 6. デフォルト値の扱い

### 変換可能な例

- `@default(1)` → `.default(1)`
- `@default(true)` → `.default(true)`
- `@default(now())` → `.default(sql\`now()\`)`

### 要レビュー扱いの例

- `@default(cuid())` は自動変換対象外で警告として記録されます。
- その場合 `status` は `partial-success` になりやすく、`gaps` に `PRISMA_DEFAULT_REVIEW` が追加されます。

## 7. @map / 名前変換

`@map` が使われている列は、Mora 側の列名を変換します。

```prisma
model Post {
  publishedAt String @map("published_at")
}
```

この場合 `publishedAt` は `published_at` として出力されるため、既存 DB の物理名との互換が保てます。

## 8. 変換結果を読むコツ（重要）

### `status` 判定

- `gaps.length === 0` → `success`
- `gaps.length > 0` → `partial-success`
- 実行時エラーで停止した場合は `failed`

### gap の種類を区別する

- `info`: 設計上の注意（自動変換は成立しているが確認推奨）
- `warning`: 手動で差分対応が必要な可能性が高い
- `error`: 実行不能（または重大）

### 実務運用の進め方

1. まず `status` が `success` なら、`schema.mora.ts` をレビュー
2. `partial-success` は `gaps` を優先度順に潰す
3. `drizzle` と同様に、必要なら生成物を最終整形して本番移行

## 9. チェックリスト（教材演習用）

- [ ] `migrate from` が JSON で終了することを確認
- [ ] `artifacts` に 3 ファイルが生成されることを確認
- [ ] `@id` が `primaryKey()` に変換されることを確認
- [ ] `@map` で物理名が変わる場合、`schema.mora.ts` のキー名が一致することを確認
- [ ] `relation` が1つ以上ある場合、`references` 追加と `partial-success` の意味を理解
- [ ] `PRISMA_DEFAULT_REVIEW` が出たら、スキーマ運用で再設計が必要か判断

## 10. まとめ

`mora migrate from prisma` は、Prisma 全文書から安全に Mora へ移す魔法のコマンドではなく、
**大半を自動化しつつ未対応箇所を明示する変換支援コマンド**です。

実務では次をセットで回すのが定石です。

- 自動生成結果のレビュー
- gap の優先度解消
- 参照制約・既定値の再定義
- 既存テストでの整合確認
