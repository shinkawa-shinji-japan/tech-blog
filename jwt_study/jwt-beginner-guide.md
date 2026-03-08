# JWT（JSON Web Token）完全理解ガイド — 初心者向け

**このガイドの役割**: JWT の**全体の流れ**（何に使うか・どんな形か・改ざんをどう防ぐか・実装の注意）を解説する。  
**用語の定義・アルゴリズムの詳細・Base64 などの手順**は **[jwt-glossary.md](./jwt-glossary.md)**（用語集）にまとめてある。知りたい用語があれば用語集を参照すること。

---

## 1. JWT とは何か（全体像）

JWT は **「誰であるか」「何をしてよいか」を安全に運ぶためのトークン** です。発行元が正しいことと、中身が書き換えられていないことを、**署名**で示します。

- 規格: RFC 7519（→ 用語集「RFC 7519」）
- 形式: ヘッダー・ペイロード・署名の 3 部分を Base64URL でエンコードし、`.` でつないだ文字列（→ 用語集「Base64 / Base64URL」）
- 主な用途: ログイン後の**認証・認可**（API のアクセス権証明）（→ 用語集「認証・認可」）

**トークン**とは、ここでは「権利や状態を表すデータの切れはし」のこと。詳しくは用語集「トークン」を参照。

### なぜトークン（JWT）で済ませるか

従来はサーバーがセッション ID を覚えておく方式が多かったが、サーバーが状態を持つと台数増加時に扱いづらい。JWT は **トークン自体に必要な情報（誰か・期限など）を入れておく** ので、サーバーは「このトークンが正しいか」だけ検証すればよく、**ステートレス**でスケールしやすい（→ 用語集「ステートレス」）。

---

## 2. JWT の形 — 3 つのブロック

JWT は **`ヘッダー.ペイロード.署名`** の 3 部分がピリオドで区切られた 1 本の文字列です。

```
ヘッダー.ペイロード.署名
xxxxx.yyyyy.zzzzz
```

- **ヘッダー**: トークン種別（JWT）と署名アルゴリズム（例: HS256）の宣言。JSON を Base64URL したもの（→ 用語集「ヘッダー」）。
- **ペイロード**: 誰のトークンか・有効期限・権限など。JSON を Base64URL したもの。**暗号化はされていない**ので、パスワードなどを入れない（→ 用語集「ペイロード」）。
- **署名**: ヘッダーとペイロードの組み合わせが、秘密を知る者だけによって作られたことを示す部分。改ざん検知の要（→ 用語集「署名」）。

API に渡すときは **`Authorization: Bearer <トークン>`** という HTTP ヘッダーを使う（→ 用語集「Bearer」「Authorization」）。

---

## 3. 改ざんを検知する仕組み

### 考え方

1. **発行時**: サーバーが「ヘッダー + ペイロード」を**秘密鍵**で署名し、その結果を Base64URL してトークンの 3 つ目のブロックにする。
2. **検証時**: 受け取ったヘッダーとペイロードから、**同じ鍵・同じアルゴリズム**で署名を再計算し、トークンに付いている署名と**完全一致**するか比べる。
3. **一致** → 改ざんされていない。**不一致** → 拒否する。

攻撃者は秘密を知らないと正しい署名を作れないため、中身だけ書き換えたトークンは検証で弾かれる。署名の計算ロジック（HMAC-SHA256 など）は用語集「HMAC-SHA256」を参照。

### 流れの図

```
【発行時】
  ヘッダー + ペイロード + 秘密鍵
       ↓
  署名アルゴリズム（例: HMAC-SHA256）
       ↓
  署名 → トークン「ヘッダー.ペイロード.署名」として返す

【検証時】
  受け取ったトークン → ヘッダー / ペイロード / 署名 に分割
       ↓
  ヘッダー + ペイロード + 正しい鍵 で署名を再計算
       ↓
  再計算した署名 === トークンに含まれる署名 ?
      YES → 改ざんなし（検証OK）
      NO  → 改ざんあり（拒否）
```

### コード例（Node.js）— 発行・検証

HS256（共有秘密で署名）で、ログイン時にトークン発行・保護 API で検証する例。

```bash
npm init -y && npm install express jsonwebtoken
```

```javascript
const express = require('express');
const jwt = require('jsonwebtoken');
const app = express();
app.use(express.json());

const SECRET = 'my-secret-key-at-least-32-chars!!';

// ログイン: JWT 発行
app.post('/login', (req, res) => {
  const payload = {
    sub: (req.body && req.body.userId) || 'user123',
    role: 'user',
    exp: Math.floor(Date.now() / 1000) + 3600,
  };
  const token = jwt.sign(payload, SECRET, { algorithm: 'HS256' });
  return res.json({ token });
});

// 保護API: トークン検証
app.get('/api/me', (req, res) => {
  const auth = req.headers.authorization;
  const token = auth && auth.startsWith('Bearer ') ? auth.slice(7) : null;
  if (!token) return res.status(401).json({ error: 'No token' });
  try {
    const decoded = jwt.verify(token, SECRET, { algorithms: ['HS256'] });
    return res.json({ userId: decoded.sub, role: decoded.role });
  } catch (e) {
    return res.status(401).json({ error: 'Invalid token', message: e.message });
  }
});

app.listen(3000, () => console.log('Server http://localhost:3000'));
```

```bash
# トークン取得 → アクセス
TOKEN=$(curl -s -X POST http://localhost:3000/login -H "Content-Type: application/json" -d '{"userId":"alice"}' | jq -r .token)
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:3000/api/me
```

SECRET の扱い（漏洩リスク）は用語集「SECRET 漏洩」および本文「5. 実装でやってはいけないこと」を参照。

### 攻撃例: ペイロードだけ改ざんするとどうなるか

ペイロードを書き換えて `role: admin` にしても、**署名は秘密がないと再計算できない**ため、元の署名のまま送るとサーバー側で再計算した署名と一致せず **401 Invalid token** になる。改ざん検知が働いている様子を手順付きで確認したい場合は、用語集の「攻撃例: ペイロード改ざんと署名検証での拒否」を参照。

---

## 4. 対称鍵と非対称鍵 — HS256 と RS256

署名に使う「鍵」の持ち方で 2 種類ある。

| 種類     | 例    | 鍵の持ち方                     | 向き |
|----------|-------|--------------------------------|------|
| 対称鍵   | HS256 | 署名も検証も同じ秘密鍵         | サーバー 1 台 or 鍵を共有できる相手だけが検証するとき |
| 非対称鍵 | RS256 | 秘密鍵で署名、公開鍵で検証     | 認証サーバーが 1 か所で署名し、複数 API が公開鍵だけで検証するとき |

アルゴリズム名の意味（HS・RS・256）は用語集「HS256・RS256」を参照。どちらの方式でも、改ざん検知の考え方（「ヘッダー＋ペイロード」から署名を再計算して比較）は同じ。

---

## 5. 実装でやってはいけないこと

- **署名検証をスキップしない**（`verify_signature: false` などは本番で使わない）。
- **ヘッダーの `alg` を盲信しない**。検証側で「HS256 のみ」「RS256 のみ」などアルゴリズムを**固定**する。
- **アルゴリズム混乱攻撃**に注意: RS256 運用時に、攻撃者が `alg` を HS256 に変えて公開鍵で署名し直すと通ってしまう実装がある。対策は上記のアルゴリズム固定（→ 用語集「アルゴリズム混乱攻撃」）。
- **鍵の漏洩・弱い鍵を防ぐ**: SECRET や秘密鍵が漏れると正しい署名を誰でも計算でき、改ざん検知が無力化する。環境変数やシークレット管理で扱う（→ 用語集「SECRET 漏洩」）。

---

## 6. まとめ

- 改ざん検知は **署名** に依存する。署名 = 「ヘッダー + ペイロード」を秘密鍵（とアルゴリズム）で署名した結果。
- 検証 = 受け取ったヘッダー・ペイロードから署名を再計算し、トークンに含まれる署名と一致するか見る。一致 → 改ざんなし、不一致 → 拒否。
- 署名検証を必ず行う・アルゴリズムを固定する・秘密鍵を守る、が不可欠。

---

## 参考

- **[jwt-glossary.md](./jwt-glossary.md)** … 用語集（RFC 7519, Base64/Base64URL, トークン, ヘッダー/ペイロード/署名, Bearer, HS256/RS256, HMAC-SHA256, エンコード・デコード手順など）
- [jwt.io — Introduction](https://jwt.io/introduction)
- [RFC 7519 — JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)
- [RFC 7515 — JSON Web Signature (JWS)](https://www.rfc-editor.org/rfc/rfc7515)
