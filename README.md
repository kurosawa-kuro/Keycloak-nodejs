# Keycloak-nodejs

# Keycloak フルスタック Node.js 技術設計ドキュメント

## 1. 概要

Nuxt 3 (SSR) + Express (REST API) + Keycloak (IdP) によるフルスタック構成の認証付き Web アプリ PoC の設計をまとめる。DB は開発簡便性を優先し json.db (lowdb) を採用。

```
┌───────────┐   Auth Code + PKCE   ┌──────────────┐
│  Nuxt 3   │ ───────────────────►│  Keycloak    │
│  (SSR)    │ ◄── tokens (JWT) ───┘ 26.x (IdP)
└────┬──────┘                      ▲
     │Bearer JWT                   │JWKS 署名検証
┌────▼────────┐                   │
│ Express API │◄───────────────────┘
│ bearer‑only │  /me, /items …
└────┬────────┘
     ▼ json.db (lowdb)
```

## 2. 採用技術

| レイヤ        | 技術                             | バージョン例          |
| ---------- | ------------------------------ | --------------- |
| IdP        | **Keycloak**                   | 26.2.0 (Docker) |
| Frontend   | **Nuxt 3**                     | 3.12 以降         |
| Front‑OIDC | `nuxt-oidc-auth`               | 3.x             |
| Backend    | **Node.js 22** + **Express 5** | ‑               |
| Auth MW    | `keycloak-connect`             | 24.x            |
| DB (PoC)   | lowdb + json.db                | 6.x             |
| DevOps     | Docker Compose / Helm / kind   | ‑               |

## 3. Keycloak 設定手順

1. `docker-compose up -d keycloak` で起動。
2. Realm `dev` を作成。
3. Clients:

   * **nuxt-frontend** (Public)

     * Redirect URI: `http://localhost:3000/auth/keycloak/callback`
   * **express-api** (Bearer‑only)
4. Self‑Registration を有効化 (任意)。
5. `nuxt-frontend` の `clientId`, `issuer` を Nuxt 側設定に反映。

## 4. Nuxt 側実装ポイント

```ts
export default defineNuxtConfig({
  modules: ['nuxt-oidc-auth'],
  oidc: {
    defaultProvider: 'keycloak',
    providers: {
      keycloak: {
        issuer: 'http://localhost:8080/realms/dev',
        clientId: 'nuxt-frontend',
        redirectUri: 'http://localhost:3000/auth/keycloak/callback',
        scopes: ['openid','profile','email']
      }
    }
  }
})
```

* **グローバル middleware** で `useOidcAuth()` → 未認証なら `login('keycloak')`。
* SSR ⇆ ブラウザ間で `access_token/refresh_token` を安全同期。

## 5. Express API 実装ポイント

```js
import Keycloak from 'keycloak-connect'
import session from 'express-session'

const store = new session.MemoryStore()
app.use(session({secret:'dev',resave:false,saveUninitialized:true,store}))
const keycloak = new Keycloak({store})
app.use(keycloak.middleware())

app.get('/api/me', keycloak.protect(), (req,res)=>{
  const {sub,email} = req.kauth.grant.access_token.content
  db.read();
  const user = db.data.users?.find(u=>u.sub===sub)
  user ? res.json(user) : res.status(404).json({msg:'not found'})
})
```

* クライアント設定を *bearer‑only* にすることでサーバ側ステートレス。

## 6. ユーザーフロー

| フェーズ   | 処理                           | ステート保存                   |
| ------ | ---------------------------- | ------------------------ |
| ログイン   | Nuxt → Keycloak              | トークン: Nuxt cookie/memory |
| API 呼出 | Nuxt attach Bearer           | Express は検証のみ            |
| サインアップ | Front で登録後 `/api/users/sync` | DB にアプリ独自情報だけ保存          |
| ログアウト  | Nuxt → Keycloak end\_session | Nuxt がトークン破棄             |

## 7. Docker Compose (開発)

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:26.2.0
    command: start-dev
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8080:8080"
  api:
    build: ./api
    environment:
      KC_REALM: dev
    ports:
      - "4000:4000"
    depends_on: [keycloak]
  nuxt:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on: [keycloak]
```

## 8. デプロイメモ

* **kind/EKS**: Keycloak を専用チャート、API & Nuxt は各 Deployment。
* **Ingress**: `/auth/*` → Keycloak、`/api/*` → Express、その他 → Nuxt。
* **TLS** 終端は Ingress Controller (nginx‑ingress) で統一。

## 9. セキュリティ考慮

* HTTPS + HSTS / Secure & SameSite cookies。
* Token Introspection は不要、署名検証 (RS256) のみ。
* Keycloak Admin API へのアクセスキーは環境変数 + Kubernetes Secret。

## 10. 今後の拡張案

1. json.db → PostgreSQL 移行（Knex or TypeORM）。
2. Keycloak → OIDC Provider 切替に備えた抽象レイヤ。
3. Nuxt UI を [Nuxt UI](https://ui.nuxt.com) でブラッシュアップ。
4. Argo CD ＋ Helm で GitOps 化。

---

*Last update: 2025‑06‑29*

### いまの設計ドキュメントを **“実装に落とす”** ときの最適な役割分担

| フェーズ                                                                | 最適ツール                | 依頼内容の粒度                                | なぜそのツール？                                             | 目安コスト           |
| ------------------------------------------------------------------- | -------------------- | -------------------------------------- | ---------------------------------------------------- | --------------- |
| **1. 雛形一括生成**<br>（Docker Compose, Nuxt, Express, Keycloak 設定ファイル一式） | **Devin**            | *1 Run* で<br>「この設計書どおりのリポジトリを作成」       | ⬤ 30 ファイル以上でもまとめて出力できる<br>⬤ Keycloak 連携 CLI の自動入力が得意 | **0.4–0.6 ACU** |
| **2. 動作確認 & 粗バグ修正**<br>（ビルドエラー／環境変数漏れ）                              | **Cursor**           | 失敗ログを貼り付けて「Fix」                        | ⬤ Diff を目視しながら安全に適用<br>⬤ テスト実行はローカルなので無料             | GPT-3.5 で**数円** |
| **3. ビジネスロジック追加**<br>（/items CRUD・/users/sync ほか）                   | **Devin**<br>＋Cursor | Devin: エンドポイント骨格<br>Cursor: モデル整形・命名統一 | ⬤ Devin は複数ファイル派生が速い<br>⬤ Cursor で DB スキーマと型を磨く      | Devin 0.05/EP   |
| **4. UI 微調整 & コードリファクタ**<br>（Nuxt UI/Store 名称変更など）                  | **Cursor**           | 選択範囲 → Inline Refactor                 | ⬤ VS Code 上で即 Preview→Commit<br>⬤ 小さな変更は GPT-3.5 で十分 | 数円              |
| **5. Helm/Argo CD マニフェスト生成**                                        | **Devin**            | 「kind/EKS 用 Helm Chart を追加」            | ⬤ YAML 多発＋リンク先生成が得意                                  | 0.1–0.2 ACU     |
| **6. リーンテスト修正サイクル**                                                 | **Cursor**           | Jest/Playwright 失敗ごとに Fix              | ⬤ 失敗ログ→AI→Patch の最短ループ                               | 数円              |

> **キモ**
> 1️⃣ **“スキャフォールドや大規模ファイル生成” は Devin**
> 2️⃣ **“細部調整・テスト駆動” は Cursor**
> こう切り分けると **ACU を最小限** に抑えつつ実装スピードも落ちません。

---

#### 具体的な依頼テンプレ

**Devin への初回 Prompt（例）**

```
Generate a full-stack repository from the attached design doc:
 – /frontend: Nuxt 3 + nuxt-oidc-auth
 – /api: Node.js 22 + Express 5 + keycloak-connect + lowdb
 – /infra: docker-compose.yml with Keycloak 26.2
Include:
  • .env.sample
  • README with startup commands
Run unit tests with `npm test --workspaces` (skip if none)
```

**Cursor での一発修正（例）**

1. `./frontend/pages/index.vue` の一部を選択 → `⌘I` → “Extract to composable”.
2. ターミナルで `npm run test` → FAIL → ログを Chat パネルへ貼り付け → “Fix the failing test”.

---

### まとめ

* **最初から最後まで Devin に丸投げ**すると ACU が跳ねがち。
* **Cursor だけでゼロから組む**とマルチファイル生成が煩雑。
* **⇒ “骨組み==Devin / 仕上げ==Cursor”** がコスパ最適。

残り **7 ACU 強** でも、この分担なら十分 PoC を完遂できます。
次のステップや具体的な Devin/Cursor の使い分けで迷ったら、また聞いてください！
