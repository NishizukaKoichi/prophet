# Prophet

## 目的

サブスクで"魔力（MP）"を蓄え、**記事の解放**／**マーケット購入**に消費する。足りなければ会員限定アイテム「**エーテル**」で不足分だけを補う。  
UI は**常にひとつのCTA**、計算と権限制御は**サーバ側で厳密**に行う。

---

## コア要件

### プラットフォーム

- **基盤**：Cloudflare Workers（Next.js App Router / OpenNext）
    
- **認証**：Magic Link（メール・単回・短TTL・レート制限）
    
- **入口UI**：存在機能のみの 3 ボタン（Articles / Market / Apps）※封印UIは出さない
    

### 決済・通貨

- **課金方式**：Stripe **Subscription（サブスクのみ）**
    
- **月次付与**：請求確定（`invoice.paid`）ごとに **+999 MP**（種別：free）  
    → **失効なし**
    
- **トップアップ**：会員限定アイテム **エーテル** 1 本 **$3 → +333 MP**（種別：paid）  
    → **失効なし**／複数本まとめて購入可（Checkout quantity）
    

### MP/XP/レベル

- **消費順序**：`free → paid`（ユーザー有利）
    
- **XP付与**：MP**消費時のみ**
    
    - 記事：`XP = 消費MP`
        
    - Market：`XP = round_half_up(消費MP × 1.5)` → **整数式推奨**：`XP = (3×消費MP + 1) // 2`
        
- **レベル**：1〜**100（MAX）**
    
    - **各Lvで次Lvに必要なXP**（ポケモン中速 Lv³ を 0.3 倍）  
        `reqXP(L) = round_half_up(0.3 × ((L+1)^3 − L^3))`  
        ※浮動誤差回避の**整数式**：`reqXP(L) = round_half_up((9L² + 9L + 3) / 10)`
        
    - **加算方式（ポケモン式）**：レベルごとのゲージに加算／満了で繰り越し。**Lv100 到達後は XP 加算停止（ゲージ 0 固定）**
        
- **所持上限（ウォレットキャップ）**：`cap = level × 1000`（Lv100＝100,000）
    

### メンバーシップ状態と可視化

- `ACTIVE`（購読中）：月 +999／**エーテル購入 可**／バッジ**カラー**（銅：Lv1–33、銀：34–66、金：67–100）
    
- `NONE`（解約中）：月 +999 **停止**／**エーテル購入 不可**（再加入誘導）／バッジ**グレー（停止中）**  
    ※どちらの状態でも **残MPの消費は可**。資産・レベル・履歴は保持
    

---

## 体験（読者視点）

1. **ログイン**：メールのマジックリンク
    
2. **記事ページ**：リード下に**「N MP で解放」**（CTA は常に 1 つ）
    
    - 残高十分 → 即全開
        
    - 残高不足 → モーダル
        
        - `ACTIVE`：**エーテル ×{本数}（{本数×333}MP / ${本数×3}）購入しますか？**  
            → 同意後はサーバが **「注ぐ →（残 0）→ 飲む → 注ぐ…」** を自動実行。**余り MP は残る**
            
        - `NONE`：**再加入して月 999MP を受け取りますか？**（エーテル購入は不可）
            
3. **Market**：同様のフロー。購入後は即ダウンロード（R2 署名 URL）
    
4. **/members**：残高（free/paid）、**cap と残容量**、履歴、レベルゲージ（Lv100 は MAX）、**バッジ常時表示**、エーテル購入（`ACTIVE`のみ）
    
5. **Apps**：SSO 起動のみ（MP/XP 非連動、アプリ内は別管理）
    

---

## ビジネス・ルール（不変）

- **勝手課金なし**：不足時は必ず確認モーダル→同意後に決済
    
- **余り MP**：保持（free/paid とも失効なし）
    
- **上限厳守**：サーバ側で `totalMP >= cap` のとき **注入禁止**  
    ※通常フローは「**消費→残 0 → 注入**」なので上限超過に至らない
    
- **再加入**：`NONE`→`ACTIVE` 化でエーテル購入が再開、月 +999 の再付与は**次回の `invoice.paid`**から
    

---

## 自動注ぎ込み（不足時のサーバ処理）

### 方針

- **ドライラン → 確認モーダル → 一括コミット**
    
- Checkout は**必要最小本数をまとめて quantity 指定**（1 回の同意で完了）
    

### ドライラン（UI 表示用）

- 入力：`required`（必要 MP），`bal`（現在 MP），`cap`（Lv×1000）
    
- 計算：不足量 `need = max(required - bal, 0)`  
    **推奨本数** `k = ceil(need / 333)`，合計 MP/金額，見込み余り MP
    
- `NONE` の場合は**再加入モーダル**に切り替え
    

### 本処理（Durable Object 内・原子的）

```ts
// 前提: membership==='ACTIVE' か、残高内で完結できること
// 1) ある分だけ注ぎ切って 0 に（予約）→ XP 加算（倍率適用・四捨五入）
// 2) Checkout 成功で +333MP × quantity（paid; 予約加算）
// 3) 再度「注ぐ」を繰り返し、必要量を満たしたら
// 4) 予約した ledger / unlock(purchase) / level を一括コミット（失敗時は全ロールバック）
```

---

## 数式・アルゴリズム（実装そのまま）

### 必要 XP（整数式・誤差なし）

```ts
// ROUND_HALF_UP 相当で実装（例：0.5 は切り上げ）
function reqXP(L:number){ return Math.round((9*L*L + 9*L + 3)/10); }
```

### XP 付与（Lv100 停止・ポケモン式繰り越し）

```ts
function awardXp(level:number, currentXp:number, baseXp:number){
  if (level >= 100) return { level:100, currentXp:0, added:0 };
  let xp = currentXp + baseXp, L = level, added = baseXp;
  while (L < 100 && xp >= reqXP(L)) { xp -= reqXP(L); L++; }
  if (L >= 100) { L = 100; xp = 0; }
  return { level:L, currentXp:xp, added };
}
```

### Market の倍率（整数で丸め）

```ts
// 消費MP→XP（Market は 1.5 倍、四捨五入）
const xpFromMarket = (spentMP:number) => Math.floor((3*spentMP + 1)/2);
```

### ウォレット上限

```ts
const walletCap = (level:number) => level * 1000;
```

---

## データモデル（最小）

```sql
users(id uuid pk, email text unique, created_at timestamptz);

profiles(
  user_id uuid pk references users(id),
  membership text not null default 'NONE', -- 'ACTIVE'|'NONE'
  level int not null default 1,            -- 1..100
  current_xp int not null default 0,       -- 0..reqXP(level)-1
  updated_at timestamptz
);

mp_wallets(
  user_id uuid pk references users(id),
  free_balance int not null default 0,
  paid_balance int not null default 0
);

mp_ledger( -- 唯一の真実
  id uuid pk, user_id uuid,
  kind text check (kind in ('grant','buy','spend','adjust')),
  mp_kind text null check (mp_kind in ('free','paid')), -- spend時null
  amount int not null,           -- +/-
  balance_after int not null,
  ref_type text, ref_id text,    -- 'post'|'product'|'system'
  created_at timestamptz, meta jsonb
);

mp_orders(
  id uuid pk, user_id uuid,
  pack text default 'ether', price_cents int not null default 300,
  grant int not null default 333,
  stripe_payment_intent text unique, created_at timestamptz
);

posts(id uuid pk, slug text unique, title text, date timestamptz,
      body_mdx text, mp_cost int not null default 15);

post_unlocks(
  user_id uuid, post_id uuid, unlocked_at timestamptz,
  method text default 'mp', primary key(user_id, post_id)
);

products(id uuid pk, name text, mp_cost int not null,
         asset_key text, visible bool default true);

purchases(
  user_id uuid, product_id uuid, created_at timestamptz,
  primary key(user_id, product_id)
);

apps(id uuid pk, name text, manifest_url text, icon_key text,
     visible bool default true, created_at timestamptz);

audit_logs(user_id uuid, action text, meta jsonb, ts timestamptz);
```

---

## API 契約（最小）

**Auth**

- `POST /api/auth/magic-link` → 送信
    
- `GET /api/auth/callback` → セッション確立
    

**Subscription / Webhooks**

- `POST /api/stripe/subscription-webhook`
    
    - `invoice.paid`：`membership='ACTIVE'` & **grant_free(+999)**
        
    - `customer.subscription.deleted`：`membership='NONE'`（資産保持／以後の付与・トップアップ不可）  
        ※二重付与防止のため `subscription.created` では付与しない
        

**Wallet / Ether**

- `GET /api/mp/wallet` → `{ free, paid, total, cap }`
    
- `GET /api/mp/ledger?limit=50`
    
- `GET /api/mp/store`（`ACTIVE`のみ）→ `{ pack:'ether', grant:333, price_cents:300 }`
    
- `POST /api/mp/buy` → Stripe Checkout（**idempotency_key 必須**）
    
- `POST /api/stripe/mp-webhook` → 署名検証 → **grant_paid(+333)**（DO 直列・冪等）
    

**Articles / Market（自動注ぎ込み）**

- `POST /api/articles/:slug/unlock`
    
- `POST /api/market/:id/purchase`
    
    - ドライラン → モーダル同意 →（必要なら）**エーテル quantity** で Checkout → **一括コミット**
        
    - 成功：`mp_ledger(spend)`，`post_unlocks` or `purchases`，`profiles(level,current_xp)`
        

**Apps**

- `POST /api/apps/:id/sso-token`（JWT RS256：`iss,aud,sub,email,iat,exp≤5m,jti`）  
    ※残高や級位はトークンに載せない
    

---

## UI ポリシー（最小）

- CTA は**常に 1 つ**：足りる→「MPで解放/購入」、不足→「エーテル購入」または「再加入」
    
- 通知は**数値のみ**（例：`-15MP / +15XP`、LvUP 時：`Lv34 → cap 34,000`）
    
- `/members`：残高（free/paid）、**cap/残容量**、履歴、レベルゲージ（Lv100 は MAX）、**バッジ常時表示**、エーテル購入（`ACTIVE`のみ）
    

---

## セキュリティ／整合性／運用

- **Durable Object**：`grant/buy/spend` を直列化。**idempotency_key 必須**  
    Stripe `payment_intent` に DB ユニーク制約
    
- **SSR ガード**：解放済み判定は**サーバ側**。未解放本文/資産は出さない
    
- **R2 署名 URL**：短期限・単回
    
- **Magic Link**：レート制限・使い捨てドメイン抑止・単回TTL
    
- **法務表記**：MP は**通貨でない／払戻し不可**（地域法の例外に従う）
    
- **監査・計測**：台帳に before/after 残高、メトリクス（grant/spend/buy/error_rate/level_dist）
    

---

## 設定例（env）

```
MP_GRANT_MONTHLY=999
ETHER_GRANT=333
ETHER_PRICE_CENTS=300
LEVEL_CAP=100
XP_CURVE_K=0.3
JWT_TTL_MIN=5
```

---

## 受け入れ基準（QA）

1. `invoice.paid` で **+999 free** が 1 回付与（重複なし）
    
2. `ACTIVE` だけ **エーテル $3→+333 paid** を購入可能。`NONE` は再加入誘導
    
3. 不足時は**確認モーダル必須**。同意後、サーバが**自動注ぎ込み**を完走し、余りは残る
    
4. **cap = level×1000** を超える注入は**拒否**（通常フローは「消費→残 0 → 注入」で安全）
    
5. XP は**消費MP×倍率**で加算（Market は整数式で四捨五入）。**Lv100 で停止**
    
6. バッジは**常時表示**（ACTIVE＝カラー／NONE＝グレー「停止中」）
    
7. すべての書き込みは**DO 直列＋冪等**。Stripe Intent 重複でも二重記帳しない
    
8. UI は CTA 1 つ、/members で**残高・上限・履歴・レベル**が正しく見える
