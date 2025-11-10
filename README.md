# Cloud Functions - データ & API 設計（ドキュメント）

このドキュメントは `functions/` フォルダ内の Firebase Cloud Functions（招待機能）について、データ構造、各 API の入力・出力、エラー設計、認可要件、シークレット、実装上の注意点を日本語で整理したものです。

対象ファイル（実装参照）:
- `functions/index.js`
- `functions/config.js`
- `functions/invitation/send.js`
- `functions/invitation/verify.js`
- `functions/invitation/complete.js`
- `functions/helpers/database.js`
- `functions/helpers/encryption.js`
- `functions/helpers/email.js`

---

## 目次
1. 概要
2. シークレット／環境変数
3. Firestore のデータモデル
4. Cloud Functions API 一覧
   - sendInvitation
   - verifyToken
   - completeRegistration
   - dataEncryption / dataDecryption
5. 共通の認可・エラー仕様
6. クライアントからの呼び出し例（Firebase Functions Client）
7. シーケンス（フロー）
8. ローカル開発とテスト
9. セキュリティ注意点 / 改善案

---

## 1. 概要
このプロジェクトは、組織への招待を送信・検証・承認するためのサーバーレス機能群です。主に次を提供します。
- 招待トークンの発行とメール送信
- 招待トークンの検証
- 招待を受けたユーザーの登録完了処理（組織にユーザーを追加、カスタムクレーム設定）
- 機密データの暗号化／復号化補助

関数は Firebase Cloud Functions（onCall／httpsCallable）として実装されています（`functions/index.js` 参照）。

---

## 2. シークレット／環境変数
`functions/config.js` で Secret Manager または環境変数から取得します。

必須シークレット:
- `GMAIL_EMAIL` - 招待メール送信用の Gmail アドレス
- `GMAIL_PASSWORD` - Gmail のパスワード（推奨：App Password）
- `AES_KEY` - AES-256 用の鍵（Base64 エンコードされた 32 バイト）
- `APP_DOMAIN` - フロントエンドのドメイン（例: `flow-it.example.com`）

注意:
- Secret Manager が利用可能な本番環境を想定。ローカルでは `.env` もしくは emulator 用の設定を使用。
- `AES_KEY` は base64 形式（32 バイト）で、`encryption.js` 側で Buffer に変換して AES-256-CBC に使用します。

---

## 3. Firestore のデータモデル
実装に登場するコレクション

- `user_invitation` (招待)
  - ドキュメント ID: 自動生成（`createInvitation` で add）
  - フィールド例:
    - `email` (string): 招待対象のメールアドレス
    - `oid` (string): 組織ID
    - `token` (string): 招待トークン（ランダム hex）
    - `status` (string): `pending` | `accepted` | `expired`
    - `role` (string?): 付与するロール（`member` 等）
    - `invitedBy` (string): 招待者の UID
    - `createdAt` (timestamp)
    - `expiresAt` (timestamp)
    - `ciphers` (array): 暗号化された email/oid 用の cipher/iv 情報（実装では配列に格納）
    - `acceptedAt`, `expiredAt`, `userId` など（承認時に追加）

- `master_web_user` (ユーザー)
  - ドキュメント ID: メールアドレス（実装上は email を key にしている）
  - フィールド例:
    - `oid` (string)
    - `email` (string)
    - `role` (string)
    - `password` (string): 暗号化済みのパスワード（ハッシュ化ではなく cipher として保存される設計）
    - `createdAt`, `updatedAt` (timestamp)

注意点:
- 現在の `firestore.rules` はまだ緩め (`allow read, write: if true`) になっています。本番では厳密なルールに置き換える必要があります（後述）。

---

## 4. Cloud Functions API 一覧
以下は各 onCall 関数の仕様（入力・出力・エラーコード）です。クライアントは `firebase/functions` の `httpsCallable` を使って呼び出します。

### 4.1 sendInvitation
- 関数名: `sendInvitation`
- 実装: `functions/invitation/send.js`

目的: 管理者/オーナー権限を持つユーザーが外部のメールアドレスに招待を送る。

認可: `request.auth` が必須。招待送信者のロールを `getUserRole` で確認し、`member` の場合は拒否。

入力 (request.data):
- `email` (string) - 招待先メールアドレス（必須）
- `oid` (string) - 組織ID（必須）
- `role` (string, optional) - 招待時に指定するロール

出力 (成功時):
- { success: true, message: string, invitationId: string }

想定されるエラー（HttpsError を返す）:
- `unauthenticated` - 認証が無い
- `invalid-argument` - email/oid が不足
- `not-found` - 招待者のユーザー情報が見つからない
- `permission-denied` - 招待者が member で権限がない
- `already-exists` - 対象メールが既にユーザー登録済み、または既に pending 招待が存在
- `internal` - その他の内部エラー

主な処理:
1. 認証と inviter のロール確認
2. 既存ユーザー/既存招待のチェック
3. トークン生成 (`helpers/encryption.generateToken`)
4. 招待レコード作成 (`createInvitation`) - `expiresAt` は現在 + 7 日
5. 招待 URL (`https://<APP_DOMAIN>/accept?token=<token>`) を生成
6. `createInvitationEmailHtml` で HTML を生成、`sendEmail` で送信

---

### 4.2 verifyToken
- 関数名: `verifyToken`
- 実装: `functions/invitation/verify.js`

目的: クライアント側（招待リンクの読み込み時）でトークンを検証し、招待の有効性を確認する。

入力 (request.data):
- `token` (string) - 招待トークン（必須）

出力 (成功時):
- { valid: true, email: string, oid: string }

想定されるエラー（HttpsError）:
- `invalid-argument` - token が無い
- `not-found` - 招待が見つからない
- `deadline-exceeded` - 招待が期限切れ（この場合、招待を `expired` に更新する処理も行う）
- `internal` - その他の内部エラー

主な処理:
1. request.data.token の検証
2. `findInvitationByToken` で招待を検索
3. 有効期限チェック (`isInvitationExpired`)。期限切れなら `markInvitationExpired` を呼んで `deadline-exceeded` を返す。
4. 有効であれば招待の email/oid を返す。

---

### 4.3 completeRegistration
- 関数名: `completeRegistration`
- 実装: `functions/invitation/complete.js`

目的: 招待を受け取ったユーザーがアカウントを作成した後、招待を承認して組織に登録する処理を行う。

認可: `request.auth` が必須（クライアント側で `createUserWithEmailAndPassword` 等でログイン済みの状態で呼ぶ想定）。

入力 (request.data):
- `token` (string) - 招待トークン（必須）
- `userPasswordHash` (string) - クライアント側で `dataEncryption` を使って暗号化したパスワード情報（JSON 文字列）
  - 形式例: `JSON.stringify({ cipher, iv })`

出力 (成功時):
- { success: true, message: "登録が完了しました", oid: <string>, ... }

想定されるエラー（HttpsError / あるいは関数はオブジェクトを返す）:
- `unauthenticated` - 未認証
- `invalid-argument` - token の欠如
- `not-found` - 招待が見つからない
- `permission-denied` - 招待の email と現在認証ユーザーの email が一致しない
- `deadline-exceeded` - 招待の期限切れ
- `internal` - その他

主な処理:
1. 認証の確認
2. `findInvitationByToken` で招待を検索
3. 招待 email と認証ユーザー email の一致確認
4. 期限チェック
5. `addUserToOrganization` で `master_web_user` にユーザーを追加（password フィールドには `userPasswordHash` を保存）
6. `admin.auth().setCustomUserClaims` でカスタムクレーム（`oid`, `role`）を設定
7. `markInvitationAccepted` で招待を `accepted` に更新

注意:
- パスワードはフロントエンド側で `dataEncryption` を使って一度暗号化（AES）してから送信する設計です。Cloud Functions 側は復号しないまま保存している実装になっています（保存は暗号化されたまま）。この点は要検討（通常はハッシュ化して保存する方が望ましい）。

---

### 4.4 dataEncryption / dataDecryption
- 関数名: `dataEncryption`, `dataDecryption`
- 実装: `functions/helpers/encryption.js` と `functions/index.js` 経由

目的: クライアントが機密データ（例：パスワード）をサーバーへ送る際に AES-256-CBC で暗号化するための補助関数。

- dataEncryption 入力: `{ text: string }`
  - 出力: `{ cipher: string (base64), iv: string (base64) }`

- dataDecryption 入力: `{ cipher: string (base64), iv: string (base64) }`
  - 出力: 復号された文字列

注意:
- `AES_KEY` は `config.js` から取得される（Secret Manager／env）。
- クライアントは `dataEncryption` を呼んで得た `{cipher, iv}` を `JSON.stringify({ cipher, iv })` のようにして `completeRegistration` に渡す実装になっています。

---

## 5. 共通の認可・エラー仕様
### HttpsError コード（実装で使用されているもの）
- `invalid-argument` - 入力が不正
- `not-found` - リソースがない
- `deadline-exceeded` - 有効期限切れ
- `unauthenticated` - 認証が必要
- `permission-denied` - 権限不足
- `already-exists` - すでに存在
- `internal` - サーバ内部エラー

クライアントは `error.code` と `error.message` を利用して適切にハンドリングすること。

---

## 6. クライアントからの呼び出し例（Firebase Functions Client）
フロントエンド実装は `src/pages/AcceptInvitation.jsx` を参照。呼び出し方の例を示します（JavaScript / firebase v9+）：

```js
import { httpsCallable } from 'firebase/functions';
import { functions } from './services/firebase';

// トークン検証
const verify = httpsCallable(functions, 'verifyToken');
const res = await verify({ token });
// 成功: res.data => { valid: true, email, oid }

// 暗号化
const encrypt = httpsCallable(functions, 'dataEncryption');
const encRes = await encrypt({ text: password });
// encRes.data => { cipher, iv }

// 登録完了
const complete = httpsCallable(functions, 'completeRegistration');
await complete({ token, userPasswordHash: JSON.stringify(encRes.data) });
```

注意: onCall 関数は Firebase SDK が自動で認証情報を付与します。ブラウザから呼ぶ場合は `firebase/auth` でログイン済みであることが前提です。

---

## 7. シーケンス（代表フロー）
### 招待送信フロー（管理者側）
1. 管理者がクライアントから `sendInvitation({ email, oid, role })` を呼ぶ
2. Functions: 権限チェック → トークン生成 → Firestore に招待作成 → 招待メール送信（Gmail）

### 招待受け取りフロー（受信者）
1. 受信者がメールの URL (`/accept?token=...`) にアクセス
2. フロントエンド: `verifyToken({ token })` を呼び、有効なら登録フォームを表示
3. 受信者がパスワードを入力 → フロントエンドが `dataEncryption` を呼びパスワードを暗号化
4. フロントエンドでユーザー作成（`createUserWithEmailAndPassword`）を行い、サインイン済みで `completeRegistration({ token, userPasswordHash })` を呼ぶ
5. Functions: 招待検証 → 組織にユーザー追加 → カスタムクレーム設定 → 招待ステータス更新

---

## 8. ローカル開発とテスト
- `firebase emulators:start` により Auth/Functions/Firestore/Hosting のエミュレータが起動します（`firebase.json` 参照）
- Functions は `npm run serve`（`functions` ディレクトリ内）でローカルシェルが使えますが、エミュレータ推奨
- Secret Manager がローカルに無い場合は環境変数で `GMAIL_EMAIL`, `GMAIL_PASSWORD`, `AES_KEY`, `APP_DOMAIN` を設定してください

---

## 9. セキュリティ注意点 / 改善案（推奨）
1. Firestore ルールの強化
   - 現在の `firestore.rules` は `allow read, write: if true` なので、本番では必ず `request.auth` とカスタムクレーム（`oid`, `role`）を使ったアクセス制御を導入する。
2. パスワード取り扱い
   - 現行設計ではクライアントがパスワードを AES で暗号化し、そのまま Firestore に保存（`password` フィールド）しているように見えます。望ましい実装は次の通り：
     - サーバー側で安全なハッシュ（bcrypt など）を作成して保存する
     - もしくは Firebase Authentication のパスワードを用いる（Auth にユーザー登録し、Auth の UID を基にマスターユーザーを紐付ける）
3. シークレット管理
   - `GMAIL_PASSWORD` を直接使わず、SendGrid 等のサービスの利用を検討。または Secret Manager を厳密に運用する。
4. 招待トークンの管理
   - トークンは一度使用したら無効化する（現行は `accepted` に更新しているので OK）。複数回使用を許す場合のリスクを考慮する。
5. ログ出力
   - エラーログに機密情報（メールアドレス、パスワードの断片など）が含まれないように注意。

---

## 付録: 代表的なリクエスト/レスポンス例
### verifyToken 成功例
Request: `{ token: "abc123..." }`
Response: `{ valid: true, email: "invitee@example.com", oid: "org_001" }`

### sendInvitation 成功例
Request: `{ email: "invitee@example.com", oid: "org_001", role: "member" }` (認証必須)
Response: `{ success: true, message: "招待メールを送信しました", invitationId: "<docId>" }`

### completeRegistration 成功例
Request: `{ token: "...", userPasswordHash: JSON.stringify({ cipher: "...", iv: "..." }) }` (認証必須)
Response: `{ success: true, message: "登録が完了しました", data: { oid: "org_001", role: "member" } }

---

## 最後に
このドキュメントは実装（`functions/`）の現状をベースに作成しました。将来的に次の改善をお勧めします：
- Firestore ルールの厳格化
- パスワード処理の再設計（サーバ側ハッシュ化 or Firebase Auth の統合の徹底）
- E2E テストとセキュリティ監査

必要であれば、このドキュメントを英語版に翻訳したり、OpenAPI（Swagger）風の仕様に落とし込むこともできます。ご希望あれば続けます。
