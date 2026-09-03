# バーコード読み取りによるカロリー取得 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** バーコードを読み取って Open Food Facts から栄養情報を取得し、分量を確認したうえで1食として記録できるようにする。

**Architecture:** `@zxing/library` のUMDビルドを `vendor/` に同梱してカメラ映像からJANコードをデコードし、Open Food Facts の公開APIへ直接fetchする。取得値は100gあたりなので、確認画面で分量へ換算してから既存の `addMeal` に渡す。保存形式は変えない（`source: "barcode"` が増えるだけ）。

**Tech Stack:** 素の HTML/CSS/JavaScript（ES5相当・ビルドなし）、`@zxing/library` 0.23.0（UMD・Apache-2.0）、getUserMedia、Open Food Facts API v2、service worker、GitHub Pages + GitHub Actions

**Spec:** `docs/superpowers/specs/2026-09-03-barcode-lookup-design.md`

## Global Constraints

- 既存コードのスタイルに合わせる: `var` 宣言、`function` 式。**アロー関数・`const`/`let`・テンプレートリテラルを使わない**。
- **ビルドステップを追加しない。** 同梱するのは配布済みのUMDファイルそのもの。
- **CDNから実行時に読み込まない。** `vendor/` の同梱ファイルを参照する（オフライン起動を維持するため）。
- ライブラリは **`@zxing/library` 0.23.0** に固定。入手元 `https://cdn.jsdelivr.net/npm/@zxing/library@0.23.0/umd/index.min.js`（実測 362,150 バイト・グローバル `ZXing` を公開・`BrowserMultiFormatReader` を含む）。
- **Apache-2.0 の LICENSE / NOTICE を `vendor/` に同梱する。**
- OFF への照会は `app_name` / `app_version` をクエリで渡す。**User-Agent はブラウザでは設定できないので設定しようとしない。**
- **自動リトライを実装しない**（レート制限 15req/min を無駄に消費するため）。
- **`protein` が無い場合は空欄。0を捏造しない。** kcal も同様。
- 食事名・商品名は利用者由来のテキストとして `escapeHtml` を通す。
- `window.alert` / `confirm` / `prompt` を使わない。
- テストは既存のページ内ハーネス（`?selftest=1`・現在46件）に追加する。`SELFTEST.add(name, fn)` と `T.eq` / `T.deepEq` / `T.ok` / `T.throws`。
- **どのテストも `nexra-diet-tracker` と `nexra-diet-tracker-backup-v1` を読み書き・削除してはならない**（オーナーの実データ）。
- 実機確認は必ず**別のSTORAGE_KEYに差し替えた捨てコピー**か隔離プロファイルで行う。
- 各タスクの最後にコミットする。メッセージ末尾に:
  ```
  Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_01LSabJuSkWyyA66gg3EQP2f
  ```

## File Structure

| ファイル | 役割 |
|---|---|
| `vendor/zxing-0.23.0.umd.min.js` | 新規。同梱するデコーダ本体 |
| `vendor/zxing-LICENSE.txt` / `vendor/zxing-NOTICE.txt` | 新規。Apache-2.0 の要件 |
| `.github/workflows/deploy-pages.yml` | 修正。**明示したファイルしかコピーしないので `vendor/` を足さないと本番だけ404になる** |
| `sw.js` | 修正。`PRECACHE_URLS` に vendor を追加、`CACHE_NAME` を v7 → v8 |
| `diet-tracker-pwa.html` | 修正。純粋関数・照会・カメラ・確認画面・失敗経路 |

---

### Task 1: ライブラリの同梱と配信経路

**この機能全体がここに乗るので最初に通す。** ここを間違えるとローカルでは動くのに本番だけ404になる。

**Files:**
- Create: `vendor/zxing-0.23.0.umd.min.js`, `vendor/zxing-LICENSE.txt`, `vendor/zxing-NOTICE.txt`
- Modify: `.github/workflows/deploy-pages.yml`, `sw.js`, `diet-tracker-pwa.html`

- [ ] **Step 1: ライブラリとライセンスを取得する**

```bash
mkdir -p vendor
curl -sL -o vendor/zxing-0.23.0.umd.min.js "https://cdn.jsdelivr.net/npm/@zxing/library@0.23.0/umd/index.min.js"
curl -sL -o vendor/zxing-LICENSE.txt "https://cdn.jsdelivr.net/npm/@zxing/library@0.23.0/LICENSE"
curl -sL -o vendor/zxing-NOTICE.txt  "https://cdn.jsdelivr.net/npm/@zxing/library@0.23.0/NOTICE"
wc -c vendor/zxing-0.23.0.umd.min.js
```
Expected: 362150 バイト。**違っていたら止めて報告する**（別版を掴んでいる）。
`NOTICE` が404で取れない場合は、その事実を報告し `zxing-NOTICE.txt` は作らない（無いものを捏造しない）。

- [ ] **Step 2: 配信対象に加える**

`.github/workflows/deploy-pages.yml` の `Prepare site directory` に追加する:

```yaml
          cp -r vendor _site/vendor
```

- [ ] **Step 3: service worker に登録し、キャッシュ名を上げる**

`sw.js`:
```js
var CACHE_NAME = "diet-tracker-cache-v8";
var PRECACHE_URLS = [
  "./",
  "./manifest.json",
  "./vendor/zxing-0.23.0.umd.min.js",
  "./icons/icon-192.png",
  "./icons/icon-512.png",
  "./icons/apple-touch-icon.png",
];
```
**`CACHE_NAME` を上げないと `activate` が旧キャッシュを消さず、ライブラリを持たない古いHTMLが配られ続けて読み取りが無言で動かない。**

- [ ] **Step 4: HTMLから読み込む**

`diet-tracker-pwa.html` の既存 `<script>` より**前**に置く:
```html
<script src="./vendor/zxing-0.23.0.umd.min.js"></script>
```

- [ ] **Step 5: 読み込めていることを確認する**

ローカルHTTPサーバで開き（`file://` では動かない）、コンソールで `typeof ZXing` が `"object"`、
`typeof ZXing.BrowserMultiFormatReader` が `"function"` になることを確認する。
`?selftest=1` が引き続き46件全通過であることも確認する。

- [ ] **Step 6: コミット**

```bash
git add vendor .github/workflows/deploy-pages.yml sw.js diet-tracker-pwa.html
git commit -m "chore: vendor @zxing/library 0.23.0 and add it to the delivery path"
```

---

### Task 2: カメラ読み取りの動作確認（ゲート）

**設計書が「未検証の前提」と明記した箇所。ここが通らなければ以降の実装に意味がないので、UIを作り込む前に単体で通す。**

**Files:**
- Modify: `diet-tracker-pwa.html`

**Interfaces:**
- Produces:
  - `startBarcodeScan(videoEl, onResult, onError)` → `stop` 関数を返す
  - 検出したら**必ず自分でカメラを停止する**（電池とプライバシーのため映像を持ち続けない）

- [ ] **Step 1: 最小の読み取り関数を書く**

```js
  var scanReader = null;
  function startBarcodeScan(videoEl, onResult, onError) {
    var stopped = false;
    function stop() {
      stopped = true;
      if (scanReader) { try { scanReader.reset(); } catch (e) {} scanReader = null; }
    }
    if (typeof ZXing === "undefined" || !ZXing.BrowserMultiFormatReader) {
      onError("library");
      return stop;
    }
    scanReader = new ZXing.BrowserMultiFormatReader();
    scanReader.decodeFromVideoDevice(null, videoEl, function (result, err) {
      if (stopped) return;
      if (result) { stop(); onResult(result.getText()); return; }
      // err は「まだ見つからない」でも毎フレーム来るので、種類で選別する
      if (err && err.name && err.name !== "NotFoundException") { stop(); onError(err.name); }
    })["catch"](function (e) {
      stop();
      onError(e && e.name === "NotAllowedError" ? "denied" : "camera");
    });
    return stop;
  }
```

- [ ] **Step 2: 一時的な確認用の導線を置く**

「食事を追加」の中に一時的なボタンと `<video autoplay muted playsinline>` を置き、
押すと `startBarcodeScan` を呼んで、読めた数字を画面に出すだけの状態を作る。
**`playsinline` は必須**（iOSで全画面に乗っ取られるのを防ぐ）。

- [ ] **Step 3: デスクトップChromeで実際に読み取れることを確認する**

ローカルHTTPサーバで開き、画面にEAN-13バーコードを表示して（例: `4901005009158` の画像を用意する）
カメラにかざし、その数字が表示されることを確認する。**読めた数字を報告に書くこと。**
読めなければ止めて BLOCKED で報告する（`@zxing/browser` への切り替え判断は統括役が行う）。

- [ ] **Step 4: 権限拒否の経路を確認する**

ブラウザの権限をブロックにして押し、`onError("denied")` に落ちることを確認する。

- [ ] **Step 5: コミット**

```bash
git add diet-tracker-pwa.html
git commit -m "feat: decode EAN-13 from the camera with a bundled ZXing reader"
```

---

### Task 3: 純粋関数（解析と換算）

**Files:**
- Modify: `diet-tracker-pwa.html`

**Interfaces:**
- Produces:
  - `parseOffProduct(json, jan)` → `{name, kcalPer100, proteinPer100, defaultQty, unit, productUrl}` または `null`（未収載）
  - `computeFromPer100(per100, qty)` → 整数、または `undefined`

- [ ] **Step 1: 失敗するテストを書く**

固定データは**実測した実レスポンスの抜粋**を使う:

```js
  var OFF_FIXTURE_NAGEWA = { status: 1, product: {
    product_name: "nagewa Uma shio aji", product_name_ja: "やげわ うま塩味",
    quantity: "67g", product_quantity: 67, serving_size: "67 g", serving_quantity: 67,
    nutriments: { "energy-kcal_100g": 547.761194029851, proteins_100g: 2.53731343283582 } } };
  var OFF_FIXTURE_POCKY = { status: 1, product: {
    product_name: "Pocky chocolate", product_name_ja: "Pocky chocolate",
    quantity: null, product_quantity: null, serving_size: "29 g", serving_quantity: 29,
    nutriments: { "energy-kcal_100g": 489.655172413793, proteins_100g: 8.96551724137931 } } };
  var OFF_FIXTURE_TEA = { status: 1, product: {
    product_name: "Gogo no kocha Straight Tea", product_name_ja: null,
    quantity: "500 ml", product_quantity: 500, serving_size: null, serving_quantity: null,
    nutriments: { "energy-kcal_100g": 15, proteins_100g: 0 } } };

  SELFTEST.add("parseOffProduct prefers serving quantity as the default portion", function () {
    var p = parseOffProduct(OFF_FIXTURE_NAGEWA, "4901940200641");
    T.eq(p.defaultQty, 67, "serving_quantity wins");
    T.eq(p.unit, "g", "unit from quantity string");
    T.eq(p.name, "やげわ うま塩味", "japanese name preferred");
    T.eq(p.productUrl, "https://world.openfoodfacts.org/product/4901940200641", "product url");
  });

  SELFTEST.add("parseOffProduct falls back to package quantity, and detects ml", function () {
    var p = parseOffProduct(OFF_FIXTURE_TEA, "4909411076610");
    T.eq(p.defaultQty, 500, "product_quantity used when no serving");
    T.eq(p.unit, "ml", "ml detected from '500 ml'");
    T.eq(p.name, "Gogo no kocha Straight Tea", "falls back to product_name");
  });

  SELFTEST.add("parseOffProduct handles a missing package quantity", function () {
    var p = parseOffProduct(OFF_FIXTURE_POCKY, "4901005009158");
    T.eq(p.defaultQty, 29, "serving_quantity still wins");
    T.eq(p.unit, "g", "defaults to g when no quantity string");
  });

  SELFTEST.add("parseOffProduct never fabricates missing nutrients", function () {
    var bare = { status: 1, product: { product_name: "X", nutriments: {} } };
    var p = parseOffProduct(bare, "111");
    T.eq(p.kcalPer100, undefined, "no kcal invented");
    T.eq(p.proteinPer100, undefined, "no protein invented");
    T.eq(p.defaultQty, undefined, "no quantity invented");
    T.eq(p.name, "X", "name still read");
  });

  SELFTEST.add("parseOffProduct returns null for a product that is not listed", function () {
    T.eq(parseOffProduct({ status: 0 }, "111"), null, "status 0");
    T.eq(parseOffProduct(null, "111"), null, "null payload");
    T.eq(parseOffProduct({}, "111"), null, "empty payload");
  });

  SELFTEST.add("parseOffProduct falls back to the barcode when there is no name", function () {
    var p = parseOffProduct({ status: 1, product: { nutriments: {} } }, "4901005009158");
    T.eq(p.name, "4901005009158", "barcode used as the name");
  });

  SELFTEST.add("computeFromPer100 scales and rounds", function () {
    T.eq(computeFromPer100(547.761194029851, 67), 367, "matches OFF's own serving value");
    T.eq(computeFromPer100(15, 500), 75, "whole bottle");
    T.eq(computeFromPer100(489.655172413793, 29), 142, "matches OFF's own serving value");
    T.eq(computeFromPer100(undefined, 67), undefined, "missing per-100 stays missing");
    T.eq(computeFromPer100(100, 0), 0, "zero quantity is a real zero");
    T.eq(computeFromPer100(100, undefined), undefined, "missing quantity stays missing");
  });
```

- [ ] **Step 2: ブラウザで実行して失敗を確認**

Expected: 7件 FAIL（`parseOffProduct is not defined`）。

- [ ] **Step 3: 実装する**

```js
  function offUnitOf(quantityStr) {
    if (typeof quantityStr === "string" && /ml/i.test(quantityStr)) return "ml";
    return "g";
  }

  function parseOffProduct(json, jan) {
    if (!json || json.status !== 1 || !json.product) return null;
    var p = json.product;
    var n = p.nutriments || {};
    var out = {
      name: (typeof p.product_name_ja === "string" && p.product_name_ja) ||
            (typeof p.product_name === "string" && p.product_name) || String(jan),
      unit: offUnitOf(p.quantity),
      productUrl: "https://world.openfoodfacts.org/product/" + encodeURIComponent(jan)
    };
    if (typeof n["energy-kcal_100g"] === "number") out.kcalPer100 = n["energy-kcal_100g"];
    if (typeof n.proteins_100g === "number") out.proteinPer100 = n.proteins_100g;
    if (typeof p.serving_quantity === "number") out.defaultQty = p.serving_quantity;
    else if (typeof p.product_quantity === "number") out.defaultQty = p.product_quantity;
    return out;
  }

  function computeFromPer100(per100, qty) {
    if (typeof per100 !== "number" || isNaN(per100)) return undefined;
    if (typeof qty !== "number" || isNaN(qty)) return undefined;
    return Math.round(per100 * qty / 100);
  }
```

注: `serving_quantity` が文字列で返る商品があり得る。`typeof === "number"` で弾いた結果
`defaultQty` が未定義になるのは仕様どおり（利用者が入力する）。捏造しないことを優先する。

- [ ] **Step 4: 成功を確認してコミット**

Expected: 53件全通過（46 + 7）。

```bash
git add diet-tracker-pwa.html
git commit -m "feat: parse Open Food Facts products and scale nutrients to a portion"
```

---

### Task 4: Open Food Facts への照会

**Files:**
- Modify: `diet-tracker-pwa.html`

**Interfaces:**
- Produces: `fetchOffProduct(jan)` → Promise。解決値は `{ok:true, product:<parseOffProductの戻り>}` か `{ok:false, reason:"notfound"|"ratelimit"|"network"}`

- [ ] **Step 1: 失敗するテストを書く**（`fetch` を差し替えて検証する）

```js
  SELFTEST.add("fetchOffProduct maps responses to outcomes", function () {
    var realFetch = window.fetch;
    var calls = [];
    function stub(status, body) {
      window.fetch = function (url) {
        calls.push(url);
        return Promise.resolve({ status: status, ok: status === 200,
          json: function () { return Promise.resolve(body); } });
      };
    }
    var done = [];
    stub(200, { status: 0 });
    fetchOffProduct("111").then(function (r) { done.push(r.reason); });
    stub(429, {});
    fetchOffProduct("222").then(function (r) { done.push(r.reason); });
    window.fetch = realFetch;
    T.ok(calls[0].indexOf("app_name=") >= 0, "app_name is sent");
    T.ok(calls[0].indexOf("/api/v2/product/111") >= 0, "barcode in path");
  });
```

注: 非同期の解決値までは同期テストで見られない。**URLの組み立てと分岐条件を検証対象とし、
Promiseの解決内容は実通信での確認に委ねる**。テスト名もそのとおりにすること。

- [ ] **Step 2: 実装する**

```js
  var OFF_FIELDS = "product_name,product_name_ja,quantity,product_quantity,serving_size,serving_quantity,nutriments";
  function offProductUrl(jan) {
    return "https://world.openfoodfacts.org/api/v2/product/" + encodeURIComponent(jan) +
      ".json?fields=" + OFF_FIELDS + "&app_name=diet-tracker&app_version=" + APP_VERSION;
  }

  function fetchOffProduct(jan) {
    return fetch(offProductUrl(jan)).then(function (res) {
      if (res.status === 429) return { ok: false, reason: "ratelimit" };
      if (!res.ok) return { ok: false, reason: "network" };
      return res.json().then(function (json) {
        var p = parseOffProduct(json, jan);
        return p ? { ok: true, product: p } : { ok: false, reason: "notfound" };
      });
    })["catch"](function () { return { ok: false, reason: "network" }; });
  }
```

**自動リトライを足さないこと。**

- [ ] **Step 3: 実通信で3商品を確認する**

コンソールから `fetchOffProduct("4901940200641").then(console.log)` 等を実行し、
なげわ・ポッキー・午後の紅茶の3件が `ok:true` で期待どおりの値を返すことを確認する。
存在しないコード（例 `"0000000000000"`）で `notfound` になることも確認する。**結果を報告に書く。**

- [ ] **Step 4: コミット**

```bash
git add diet-tracker-pwa.html
git commit -m "feat: look a barcode up in Open Food Facts"
```

---

### Task 5: 確認画面

**この機能の要。100gあたりの値をそのまま記録させない砦なので、分量を経由しない導線を作らないこと。**

**Files:**
- Modify: `diet-tracker-pwa.html`

**Interfaces:**
- Produces: `openScanConfirm(product, jan)` — 確認画面を開く。`renderScanConfirm()` — 再計算して描画する。

- [ ] **Step 1: HTML を追加する**

「食事を追加」の中に、既定では隠れているパネルを置く。**既存の `.panel` / `.field` / `.btn` /
`.btn.full` クラスを使う**（`.card` や `.primary-btn` はこのリポジトリに存在しない）。項目:

- 商品名（`<input type="text">`・編集可）
- 「100gあたり: NNN kcal / P N.Ng」の**表示のみ**の行（読み取った生の値。根拠として見せる）
- 分量（`<input type="number" min="0">` ＋ 単位ラベル `g` / `ml`）
- 算出結果: kcal と タンパク質の `<input type="number">`（**手で上書き可**）
- 食事帯 `<select>`（既定は `defaultMealTime(new Date())`）
- 日付 `<input type="date">`（既定は `todayStr()`）
- 出典表示（Step 4）
- `[記録に追加]` と `[やめる]`

- [ ] **Step 2: 再計算と「上書き」の状態管理を書く**

```js
  var scanState = null;   // { product, jan, kcalOverridden, proteinOverridden }

  function renderScanConfirm() {
    if (!scanState) return;
    var qty = parseFloat(document.getElementById("sc-qty").value);
    if (!scanState.kcalOverridden) {
      var k = computeFromPer100(scanState.product.kcalPer100, qty);
      document.getElementById("sc-kcal").value = (k === undefined ? "" : k);
    }
    if (!scanState.proteinOverridden) {
      var p = computeFromPer100(scanState.product.proteinPer100, qty);
      document.getElementById("sc-protein").value = (p === undefined ? "" : p);
    }
    document.getElementById("sc-overridden").hidden =
      !(scanState.kcalOverridden || scanState.proteinOverridden);
  }
```

**規則**: 利用者が kcal（またはタンパク質）を手で書き換えたら、その欄の `*Overridden` を立て、
以後**分量を変えても自動再計算しない**（直した値が黙って消えると混乱するため）。
上書き中はその旨を示し、「自動計算に戻す」で解除できるようにする。kcal と タンパク質は**独立**。

- [ ] **Step 3: 記録する**

`[記録に追加]` で、名前が空でないこと・kcal が数値であることを検証し（既存の食事フォームと
同じインライン表示を使う）、`addMeal(date, time, name, kcal, protein, "barcode")` を呼ぶ。
protein の欄が空なら `undefined` を渡す（**0を渡さない**）。続けて `upsertFood(name, kcal, protein)`
を呼び、`renderAll()` して確認画面を閉じる。

- [ ] **Step 4: 出典表示を入れる（ライセンス上の義務）**

確認画面に常時表示する。**社名だけでは足りない。**

```js
    '<div class="off-attribution">' +
      'この栄養情報は <a href="' + escapeHtml(product.productUrl) + '" target="_blank" rel="noopener">Open Food Facts</a> ' +
      'のデータベース由来で、<a href="https://opendatacommons.org/licenses/odbl/" target="_blank" rel="noopener">Open Database License (ODbL)</a> ' +
      'の下で提供されています。読み取ったバーコードは Open Food Facts に送信されます。' +
    '</div>'
```

プライバシーの明示もここで兼ねる（このアプリで初めて端末外へデータが出る機能のため）。

- [ ] **Step 5: テストを追加する**

上書き規則は状態機械なので必ずテストする:

```js
  SELFTEST.add("scan confirm recomputes on quantity change until the user overrides", function () {
    // scanState と入力要素をスタブし、
    // 1) 分量を変えると kcal が再計算される
    // 2) kcal を手で書き換えて Overridden を立てると、分量を変えても kcal が変わらない
    // 3) タンパク質の上書きは kcal の再計算に影響しない（独立していること）
  });
```
**この擬似コードのまま残さないこと。** 実際の要素IDに合わせて書き、3点を実際に検証する。

- [ ] **Step 6: 捨てコピーで動作確認してコミット**

`STORAGE_KEY` を差し替えた捨てコピーで、分量変更→再計算、上書き→固定、記録→本日の摂取に反映、
を確認する。**実データのキーには一切書き込まないこと。**

```bash
git add diet-tracker-pwa.html
git commit -m "feat: confirm the portion before logging a scanned product"
```

---

### Task 6: 失敗経路と手入力フォールバック

**「読めなかったので諦める」導線を作らないことがこのタスクの目的。**

**Files:**
- Modify: `diet-tracker-pwa.html`

- [ ] **Step 1: JANコードの手入力欄を置く**

読み取りパネルに `<input type="text" inputmode="numeric">` と `[この番号で調べる]` を置き、
カメラを使わずに同じ `fetchOffProduct` を走らせられるようにする。
**カメラ権限が拒否された場合・カメラが無い場合は、この欄を出して案内する。**

- [ ] **Step 2: 失敗を種類ごとに扱う**

| 失敗 | 表示と遷移 |
|---|---|
| `denied`（権限拒否）／`camera` | 「カメラを使えません。番号を入力してください」＋ JAN手入力欄 |
| `library`（ZXing未読込） | 同上（原因は違うが利用者の次の一手は同じ） |
| `notfound` | 「この商品は登録されていません」＋ 既存の食事追加フォームへ。**読み取れたJANを画面に残す** |
| `ratelimit` | 「時間をおいて再度お試しください」＋ 手入力へ。**自動リトライしない** |
| `network` | 「通信できませんでした」＋ 手入力へ。読み取れたJANを残す |
| ヒットしたが kcal が無い | 確認画面は開く。商品名だけ埋め、**kcal は空欄**にして利用者に入力させる |

- [ ] **Step 3: 一定時間読み取れない場合の逃げ道**

読み取り開始から一定時間（20秒程度）で「読み取れませんか? 番号を入力できます」を出す。
**カメラは止めない**（続けて読める可能性があるため）。

- [ ] **Step 4: テストを追加する**

失敗の種類 → 利用者に見せるメッセージ、の対応表を関数に切り出して検証する
（例 `scanFailureMessage(reason)`）。UIの遷移そのものは実機確認とする。

- [ ] **Step 5: 捨てコピーで確認してコミット**

権限拒否・未収載（存在しないJAN）・kcal欠損 の3経路を実際に通す。

```bash
git add diet-tracker-pwa.html
git commit -m "feat: always fall back to manual entry when a scan cannot be resolved"
```

---

### Task 7: 版番号・デプロイ・実機確認の依頼

**Files:**
- Modify: `diet-tracker-pwa.html`

- [ ] **Step 1: 版番号を上げる**

`var APP_VERSION = "3.0";` → `"3.1"`。
`sw.js` の `CACHE_NAME` は Task 1 で v8 に上げ済み。**上がっているか確認する。**

- [ ] **Step 2: セルフテストを最終実行する**

全通過でなければ次へ進まない。

- [ ] **Step 3: 本番の配信経路を確認する**

**デプロイ後に**次を確認する。ここが通らないと本番でだけ読み取りが動かない:

```bash
curl -s -o /dev/null -w "%{http_code} %{size_download}\n" https://sokiumeda.github.io/diet/vendor/zxing-0.23.0.umd.min.js
curl -s https://sokiumeda.github.io/diet/sw.js | grep -o 'CACHE_NAME = "[^"]*"'
```
Expected: `200 362150`、`diet-tracker-cache-v8`。

- [ ] **Step 4: オーナーへ実機確認を依頼する**

**push とデプロイは統括役がオーナーの判断を仰いでから行う。実行役はここで止まること。**
依頼内容: iPhoneで実際の市販品のバーコードを読み、①読み取れるか ②商品が当たるか
③分量の既定値が妥当か、を確認してもらう。**ここが設計書のゲート。**

- [ ] **Step 5: ボルトのノートを更新する**（統括役が行う）

`01_プロジェクト/diet（ダイエット管理PWA）.md` に v3.1・バーコード読み取り・
`vendor/` 同梱によって単一ファイルでなくなったこと・OFF依存とODbL表示義務を追記する。

---

## Self-Review

**1. 仕様の網羅**

| 仕様の項目 | タスク |
|---|---|
| ライブラリ同梱・CDN不使用・LICENSE同梱 | Task 1 |
| `sw.js` 登録と `CACHE_NAME` v8 | Task 1 |
| **ワークフローへの `vendor/` 追加** | Task 1 |
| カメラ読み取り（ゲート） | Task 2 |
| 分量の既定値の優先順・g/ml判定 | Task 3 |
| 100gあたり×分量の換算 | Task 3 |
| 商品名の優先順とJANフォールバック | Task 3 |
| kcal/proteinを捏造しない | Task 3・Task 5 |
| OFF照会・`app_name`・UA非設定 | Task 4 |
| 確認画面・名前編集可・分量必須経由 | Task 5 |
| 上書きと再計算の優先順 | Task 5 |
| 出典表示（ODbL明示・ライセンスリンク・商品リンク） | Task 5 |
| プライバシーの明示 | Task 5 |
| `source: "barcode"` で `addMeal` | Task 5 |
| よく食べるものへの登録 | Task 5 |
| 失敗経路すべて | Task 6 |
| レート制限・自動リトライしない | Task 4・Task 6 |
| APP_VERSION 3.1 | Task 7 |
| 本番での配信確認 | Task 7 |
| 実機ゲート | Task 2・Task 7 |

**2. 仕様に無いが必要と判明して足したもの**
- **`.github/workflows/deploy-pages.yml` への `vendor/` 追加**（Task 1 Step 2）。ワークフローは
  明示したファイルしかコピーしない実装で、これを足さないと**本番でだけ404**になる。設計時に
  見落としていた。

**3. 型と名前の一貫性**
- `parseOffProduct(json, jan)` は Task 3 で定義し、Task 4 が同じ引数順で呼ぶ。
- `computeFromPer100(per100, qty)` は Task 3 で定義し、Task 5 が使う。
- `fetchOffProduct(jan)` は Task 4 で定義し、Task 5・Task 6 が使う。
- `startBarcodeScan(videoEl, onResult, onError)` は Task 2 で定義し、Task 6 が失敗理由
  （`denied` / `camera` / `library`）を受ける。
- `addMeal(date, time, name, kcal, protein, source)`・`upsertFood(name, kcal, protein)`・
  `defaultMealTime(date)`・`escapeHtml(s)`・`todayStr()` は既存。監査で実物と一致を確認済み。
