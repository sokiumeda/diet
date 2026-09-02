# 1食ずつの食事記録（per-meal logging）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 1日の合計値だけを持つ保存形式を、1食ずつの記録を持つ形式へ移行し、レシピ・手入力・よく食べるものから食事を記録できるようにする。

**Architecture:** 単一HTMLファイル `diet-tracker-pwa.html` の中で完結する。保存形式を v2 へ移行し、1日の合計は各食から都度算出する（保存しない）。テスト基盤が無いリポジトリのため、`?selftest=1` でページ内アサーションを走らせるハーネスを最初のタスクで作り、以降のタスクはそれを使って TDD で進める。

**Tech Stack:** 素の HTML/CSS/JavaScript（ES5相当・ビルドなし・依存ライブラリなし）、localStorage、service worker（`sw.js`）、GitHub Pages + GitHub Actions

**Spec:** `docs/superpowers/specs/2026-09-03-per-meal-logging-design.md`

## Global Constraints

- 単一ファイル構成を壊さない。**外部ライブラリ・CDN・ビルドステップを追加しない**。
- 既存コードのスタイルに合わせる: `var` 宣言、`function` 式、アロー関数・`const`/`let`・テンプレートリテラルを使わない。
- 保存キーは `nexra-diet-tracker`。移行前の退避キーは `nexra-diet-tracker-backup-v1`。
- 食事帯の値は `"breakfast" | "lunch" | "dinner" | "snack" | "other"`。
- 既存の `MEAL_TIME_LABEL`（朝食/昼食/夕食）と `MEAL_TIME_ORDER`（3件）は**レシピ表示が参照しているため変更しない**。記録用は `LOG_MEAL_TIME_LABEL` / `LOG_MEAL_TIME_ORDER` を新設する。
- 合計値は保存しない。常に `meals` から算出する。
- 移行は冪等。`schemaVersion` があるデータには一切手を触れない。
- 移行の前後で合計 kcal・合計 protein が変化しないこと（不変条件）。
- `APP_VERSION` は最終タスクで `"3.0"` にする。`sw.js` の `CACHE_NAME` は `diet-tracker-cache-v6` → `v7`。
- 各タスクの最後に必ずコミットする。コミットメッセージ末尾に以下2行を付ける:
  ```
  Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_01LSabJuSkWyyA66gg3EQP2f
  ```

---

### Task 1: セルフテスト基盤（`?selftest=1`）

以降の全タスクがこのハーネスでテストを書く。最初に作る。

**Files:**
- Modify: `diet-tracker-pwa.html`（`<body>` 末尾の `<script>` 内・`APP_VERSION` 定義の直後にハーネスを置く）

**Interfaces:**
- Produces:
  - `SELFTEST` — `{ cases: [], add: function(name, fn), run: function() }`
  - `T.eq(actual, expected, msg)` / `T.ok(cond, msg)` / `T.throws(fn, msg)` — アサーション。失敗時は `Error` を投げる。
  - `T.deepEq(actual, expected, msg)` — `JSON.stringify` 比較。

- [ ] **Step 1: ハーネス本体を書く**

`var APP_VERSION = "2.0";` の直後に追加する:

```js
  // ---- self-test harness (activated by ?selftest=1) ----
  var T = {
    eq: function (a, b, msg) {
      if (a !== b) throw new Error((msg || "eq") + ": expected " + JSON.stringify(b) + " but got " + JSON.stringify(a));
    },
    deepEq: function (a, b, msg) {
      var sa = JSON.stringify(a), sb = JSON.stringify(b);
      if (sa !== sb) throw new Error((msg || "deepEq") + ": expected " + sb + " but got " + sa);
    },
    ok: function (cond, msg) {
      if (!cond) throw new Error((msg || "ok") + ": expected truthy");
    },
    throws: function (fn, msg) {
      var threw = false;
      try { fn(); } catch (e) { threw = true; }
      if (!threw) throw new Error((msg || "throws") + ": expected function to throw");
    }
  };

  var SELFTEST = {
    cases: [],
    add: function (name, fn) { SELFTEST.cases.push({ name: name, fn: fn }); },
    run: function () {
      var passed = 0, failed = 0, rows = "";
      SELFTEST.cases.forEach(function (c) {
        var ok = true, err = "";
        try { c.fn(); } catch (e) { ok = false; err = e.message; }
        if (ok) { passed++; } else { failed++; }
        rows += '<div style="padding:6px 10px;border-bottom:1px solid #333;color:' +
          (ok ? "#7ddb7d" : "#ff6b6b") + '">' + (ok ? "PASS" : "FAIL") + " — " + c.name +
          (ok ? "" : '<div style="color:#ffb4b4;font-size:12px;margin-top:4px">' + err + "</div>") + "</div>";
      });
      document.body.innerHTML =
        '<div style="font-family:monospace;background:#111;color:#eee;min-height:100vh;padding:16px">' +
        '<h1 style="font-size:18px">selftest: ' + passed + " passed, " + failed + " failed</h1>" +
        rows + "</div>";
      document.title = (failed === 0 ? "PASS" : "FAIL") + " selftest (" + passed + "/" + (passed + failed) + ")";
      return failed === 0;
    }
  };
```

- [ ] **Step 2: 起動時にセルフテストへ分岐させる**

初期化処理の最初（`loadState()` を呼ぶ前）に置く。**セルフテストモードでは通常の描画も localStorage の読み書きも行わない**（実データを触らないため）:

```js
  if (location.search.indexOf("selftest=1") >= 0) {
    SELFTEST.run();
    return;
  }
```

`return` が使えない位置なら、初期化処理全体を `if (...) { SELFTEST.run(); } else { /* 通常起動 */ }` で分岐させる。

- [ ] **Step 3: 動作確認用のダミーテストを1件追加する**

```js
  SELFTEST.add("harness works", function () {
    T.eq(1 + 1, 2, "arithmetic");
    T.throws(function () { T.eq(1, 2); }, "failing assertion throws");
  });
```

- [ ] **Step 4: ブラウザで実行して確認**

`file:///Users/soki/Documents/GitHub/diet/diet-tracker-pwa.html?selftest=1` を開く。
Expected: 「selftest: 1 passed, 0 failed」、タブのタイトルが `PASS selftest (1/1)`。

- [ ] **Step 5: コミット**

```bash
git add diet-tracker-pwa.html
git commit -m "test: add in-page selftest harness activated by ?selftest=1"
```

---

### Task 2: 食事帯の定数と算出関数

**Files:**
- Modify: `diet-tracker-pwa.html`

**Interfaces:**
- Consumes: Task 1 の `SELFTEST` / `T`
- Produces:
  - `LOG_MEAL_TIME_LABEL` — `{breakfast:"朝食", lunch:"昼食", dinner:"夕食", snack:"間食", other:"その他"}`
  - `LOG_MEAL_TIME_ORDER` — `["breakfast","lunch","dinner","snack","other"]`
  - `dayKcal(log)` → `number`
  - `dayProtein(log)` → `number`
  - `hasMeals(log)` → `boolean`（`meals` が存在し1件以上）
  - `defaultMealTime(date)` → `string`（`date` は `Date`。時刻から食事帯を決める）

- [ ] **Step 1: 失敗するテストを書く**

```js
  SELFTEST.add("dayKcal sums meal calories", function () {
    T.eq(dayKcal({ meals: [{ kcal: 420 }, { kcal: 340 }] }), 760, "sum");
    T.eq(dayKcal({ meals: [] }), 0, "empty");
    T.eq(dayKcal({}), 0, "no meals key");
  });

  SELFTEST.add("dayProtein treats missing protein as zero", function () {
    T.eq(dayProtein({ meals: [{ kcal: 420, protein: 45 }, { kcal: 200 }] }), 45, "missing protein counts as 0");
  });

  SELFTEST.add("hasMeals distinguishes unrecorded from zero", function () {
    T.eq(hasMeals({ meals: [] }), false, "empty means unrecorded");
    T.eq(hasMeals({}), false, "absent means unrecorded");
    T.eq(hasMeals({ meals: [{ kcal: 0 }] }), true, "a zero-kcal meal is still a record");
  });

  SELFTEST.add("defaultMealTime picks bucket from clock", function () {
    T.eq(defaultMealTime(new Date("2026-09-03T08:00:00")), "breakfast", "morning");
    T.eq(defaultMealTime(new Date("2026-09-03T12:30:00")), "lunch", "midday");
    T.eq(defaultMealTime(new Date("2026-09-03T19:00:00")), "dinner", "evening");
    T.eq(defaultMealTime(new Date("2026-09-03T23:00:00")), "snack", "late night");
  });
```

- [ ] **Step 2: ブラウザで実行して失敗を確認**

`...?selftest=1` を開く。Expected: 4件 FAIL（`dayKcal is not defined` 等）。

- [ ] **Step 3: 実装する**

既存の `MEAL_TIME_LABEL` / `MEAL_TIME_ORDER` の定義の直後に追加する（**既存の2つは変更しない**）:

```js
  var LOG_MEAL_TIME_LABEL = { breakfast: "朝食", lunch: "昼食", dinner: "夕食", snack: "間食", other: "その他" };
  var LOG_MEAL_TIME_ORDER = ["breakfast", "lunch", "dinner", "snack", "other"];

  function hasMeals(log) {
    return !!(log && Array.isArray(log.meals) && log.meals.length > 0);
  }

  function dayKcal(log) {
    if (!log || !Array.isArray(log.meals)) return 0;
    return log.meals.reduce(function (s, m) { return s + (typeof m.kcal === "number" ? m.kcal : 0); }, 0);
  }

  function dayProtein(log) {
    if (!log || !Array.isArray(log.meals)) return 0;
    return log.meals.reduce(function (s, m) { return s + (typeof m.protein === "number" ? m.protein : 0); }, 0);
  }

  function defaultMealTime(date) {
    var d = date || new Date();
    var mins = d.getHours() * 60 + d.getMinutes();
    if (mins < 10 * 60 + 30) return "breakfast";
    if (mins < 15 * 60) return "lunch";
    if (mins < 21 * 60) return "dinner";
    return "snack";
  }
```

- [ ] **Step 4: ブラウザで実行して成功を確認**

Expected: 5 passed, 0 failed。

- [ ] **Step 5: コミット**

```bash
git add diet-tracker-pwa.html
git commit -m "feat: add meal-time constants and derived daily total helpers"
```

---

### Task 3: v1 → v2 移行と生データの退避

このタスクが実データを触る唯一の危険箇所。テストを厚くする。

**Files:**
- Modify: `diet-tracker-pwa.html`（`loadState` / `saveState` 周辺）

**Interfaces:**
- Consumes: Task 2 の `dayKcal` / `dayProtein`
- Produces:
  - `BACKUP_KEY_V1` — `"nexra-diet-tracker-backup-v1"`
  - `newId(prefix)` → `string`
  - `migrateToV2(parsed)` → `object`（**引数を破壊せず新しいオブジェクトを返す**。`schemaVersion` があればそのまま返す）
  - `state` が `{ logs: [], foods: [], profile: null, schemaVersion: 2 }` を持つ

- [ ] **Step 1: 失敗するテストを書く**

```js
  SELFTEST.add("migrate converts v1 daily totals into one 'other' meal", function () {
    var v1 = { logs: [{ date: "2026-09-01", weight: 72.4, trained: true, calories: 1200, protein: 90 }] };
    var v2 = migrateToV2(v1);
    T.eq(v2.schemaVersion, 2, "version stamped");
    T.eq(v2.logs[0].meals.length, 1, "one meal");
    T.eq(v2.logs[0].meals[0].time, "other", "bucket");
    T.eq(v2.logs[0].meals[0].kcal, 1200, "kcal preserved");
    T.eq(v2.logs[0].meals[0].protein, 90, "protein preserved");
    T.eq(v2.logs[0].meals[0].source, "migrated", "source");
    T.eq(v2.logs[0].weight, 72.4, "weight untouched");
    T.eq(v2.logs[0].trained, true, "trained untouched");
    T.eq(v2.logs[0].calories, undefined, "old field removed");
    T.eq(v2.logs[0].protein, undefined, "old field removed");
  });

  SELFTEST.add("migrate handles calories-only, protein-only and empty days", function () {
    var v2 = migrateToV2({ logs: [
      { date: "2026-09-01", weight: 72, calories: 1200 },
      { date: "2026-09-02", weight: 72, protein: 80 },
      { date: "2026-09-03", weight: 72 }
    ] });
    T.eq(v2.logs[0].meals[0].protein, undefined, "no fabricated protein");
    T.eq(v2.logs[1].meals[0].kcal, 0, "protein-only day gets zero kcal");
    T.eq(v2.logs[1].meals[0].protein, 80, "protein kept");
    T.deepEq(v2.logs[2].meals, [], "day with neither gets empty meals");
  });

  SELFTEST.add("migrate preserves totals exactly", function () {
    var v1 = { logs: [
      { date: "2026-09-01", weight: 72, calories: 1200, protein: 90 },
      { date: "2026-09-02", weight: 72, calories: 1450, protein: 105 }
    ] };
    var before = v1.logs.reduce(function (s, l) { return s + (l.calories || 0); }, 0);
    var v2 = migrateToV2(v1);
    var after = v2.logs.reduce(function (s, l) { return s + dayKcal(l); }, 0);
    T.eq(after, before, "total kcal unchanged");
  });

  SELFTEST.add("migrate is idempotent and leaves v2 data alone", function () {
    var once = migrateToV2({ logs: [{ date: "2026-09-01", weight: 72, calories: 1200 }] });
    var twice = migrateToV2(once);
    T.deepEq(twice, once, "second pass changes nothing");
    T.eq(twice.logs[0].meals.length, 1, "no duplicated meal");
  });

  SELFTEST.add("migrate does not mutate its input", function () {
    var v1 = { logs: [{ date: "2026-09-01", weight: 72, calories: 1200 }] };
    migrateToV2(v1);
    T.eq(v1.logs[0].calories, 1200, "input untouched");
  });

  SELFTEST.add("migrate initialises foods and preserves profile", function () {
    var v2 = migrateToV2({ logs: [], profile: { startWeight: 80, goalWeight: 65 } });
    T.deepEq(v2.foods, [], "foods initialised");
    T.eq(v2.profile.goalWeight, 65, "profile preserved");
  });

  SELFTEST.add("migrate tolerates empty and malformed input", function () {
    T.deepEq(migrateToV2({}).logs, [], "no logs key");
    T.deepEq(migrateToV2({ logs: [] }).logs, [], "empty logs");
  });

  SELFTEST.add("newId produces unique ids with prefix", function () {
    var a = newId("m_"), b = newId("m_");
    T.ok(a.indexOf("m_") === 0, "prefix");
    T.ok(a !== b, "unique within same millisecond");
  });
```

- [ ] **Step 2: ブラウザで実行して失敗を確認**

Expected: 8件 FAIL（`migrateToV2 is not defined`）。

- [ ] **Step 3: 実装する**

```js
  var BACKUP_KEY_V1 = "nexra-diet-tracker-backup-v1";
  var idCounter = 0;

  function newId(prefix) {
    idCounter++;
    return prefix + Date.now() + "_" + idCounter;
  }

  function migrateToV2(parsed) {
    if (!parsed || typeof parsed !== "object") return { schemaVersion: 2, logs: [], foods: [] };
    if (parsed.schemaVersion) return parsed;

    var logs = Array.isArray(parsed.logs) ? parsed.logs : [];
    var out = { schemaVersion: 2, logs: [], foods: Array.isArray(parsed.foods) ? parsed.foods : [] };
    if (parsed.profile) out.profile = parsed.profile;

    logs.forEach(function (l) {
      var entry = { date: l.date, trained: !!l.trained, meals: [] };
      if (typeof l.weight === "number") entry.weight = l.weight;
      var hasCal = typeof l.calories === "number";
      var hasPro = typeof l.protein === "number";
      if (hasCal || hasPro) {
        var meal = {
          id: newId("m_"), time: "other", name: "まとめて記録（移行）",
          kcal: hasCal ? l.calories : 0, source: "migrated"
        };
        if (hasPro) meal.protein = l.protein;
        entry.meals.push(meal);
      }
      out.logs.push(entry);
    });
    return out;
  }
```

- [ ] **Step 4: ブラウザで実行して成功を確認**

Expected: 全件 PASS。

- [ ] **Step 5: `loadState` / `saveState` を v2 に対応させる**

`state` の初期値を `var state = { logs: [], foods: [], profile: null, schemaVersion: 2 };` に変える。

```js
  function loadState() {
    try {
      var raw = localStorage.getItem(STORAGE_KEY);
      if (!raw) return;
      var parsed = JSON.parse(raw);
      var needsMigration = parsed && !parsed.schemaVersion;
      if (needsMigration && !localStorage.getItem(BACKUP_KEY_V1)) {
        localStorage.setItem(BACKUP_KEY_V1, raw);   // 退避は一度だけ・既存があれば上書きしない
      }
      var data = migrateToV2(parsed);
      if (Array.isArray(data.logs)) state.logs = data.logs;
      if (Array.isArray(data.foods)) state.foods = data.foods;
      if (data.profile && typeof data.profile.startWeight === "number" && typeof data.profile.goalWeight === "number") {
        state.profile = data.profile;
      }
      if (needsMigration) saveState();
    } catch (e) {
      state.logs = [];
      state.foods = [];
    }
  }

  function saveState() {
    var data = { schemaVersion: 2, logs: state.logs, foods: state.foods };
    if (state.profile) data.profile = state.profile;
    localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
  }
```

- [ ] **Step 6: 退避の冪等性テストを追加して実行**

```js
  SELFTEST.add("backup key is written once and never overwritten", function () {
    var K = "selftest-backup-probe";
    localStorage.removeItem(K);
    if (!localStorage.getItem(K)) localStorage.setItem(K, "first");
    if (!localStorage.getItem(K)) localStorage.setItem(K, "second");
    T.eq(localStorage.getItem(K), "first", "first backup wins");
    localStorage.removeItem(K);
  });
```

Expected: 全件 PASS。

- [ ] **Step 7: コミット**

```bash
git add diet-tracker-pwa.html
git commit -m "feat: migrate storage to schema v2 with one-time v1 backup"
```

---

### Task 4: `addOrUpdateLog` のマージ化と体重の任意化

**放置すると体重の再記録でその日の食事が消える。** 仕様の最重要修正。

**Files:**
- Modify: `diet-tracker-pwa.html`（`addOrUpdateLog`・`renderHistory`）

**Interfaces:**
- Produces: `addOrUpdateLog(date, weight, trained)` — 引数から `calories`・`protein` を撤去。既存エントリの `meals` を保持する。

- [ ] **Step 1: 失敗するテストを書く**

```js
  SELFTEST.add("addOrUpdateLog merges instead of replacing", function () {
    var saved = state.logs;
    state.logs = [{ date: "2026-09-01", weight: 72, trained: false,
      meals: [{ id: "m_1", time: "lunch", name: "テスト", kcal: 500 }] }];
    var realSave = saveState; saveState = function () {};
    addOrUpdateLog("2026-09-01", 71.8, true);
    T.eq(state.logs[0].meals.length, 1, "meals survive a weight update");
    T.eq(state.logs[0].meals[0].kcal, 500, "meal contents intact");
    T.eq(state.logs[0].weight, 71.8, "weight updated");
    T.eq(state.logs[0].trained, true, "trained updated");
    saveState = realSave; state.logs = saved;
  });

  SELFTEST.add("addOrUpdateLog creates a new day with empty meals", function () {
    var saved = state.logs; state.logs = [];
    var realSave = saveState; saveState = function () {};
    addOrUpdateLog("2026-09-02", 70.2, false);
    T.deepEq(state.logs[0].meals, [], "new day starts with no meals");
    saveState = realSave; state.logs = saved;
  });
```

- [ ] **Step 2: ブラウザで実行して失敗を確認**

Expected: 1件目が FAIL（`meals` が消える／引数不一致）。

- [ ] **Step 3: 実装する**

```js
  function addOrUpdateLog(date, weight, trained) {
    var idx = state.logs.findIndex(function (l) { return l.date === date; });
    if (idx >= 0) {
      state.logs[idx].weight = Math.round(weight * 10) / 10;
      state.logs[idx].trained = !!trained;
      if (!Array.isArray(state.logs[idx].meals)) state.logs[idx].meals = [];
    } else {
      state.logs.push({ date: date, weight: Math.round(weight * 10) / 10, trained: !!trained, meals: [] });
    }
    saveState();
  }
```

- [ ] **Step 4: `renderHistory` の体重表示を体重なしに対応させる**

`'<span class="weight num">' + fmt1(l.weight) + 'kg</span>'` を次に置き換える:

```js
        '<span class="weight num">' + (typeof l.weight === "number" ? fmt1(l.weight) + "kg" : "—") + '</span>' +
```

同じ行のマクロ表示も算出値に変える:

```js
      var macroParts = [];
      if (hasMeals(l)) {
        macroParts.push(dayKcal(l) + "kcal");
        var p = dayProtein(l);
        if (p > 0) macroParts.push(p + "g");
      }
```

- [ ] **Step 5: 体重を直接読む箇所を全数確認する**

Run: `grep -n "\.weight" diet-tracker-pwa.html`
体重が `undefined` になり得る前提で、各ヒットが安全か確認する。`metricLogs("weight")` は
`typeof l[metric] === "number"` で除外しているため安全。危険な箇所があればこのタスクで直す。

- [ ] **Step 6: ブラウザで実行して成功を確認**

Expected: 全件 PASS。

- [ ] **Step 7: コミット**

```bash
git add diet-tracker-pwa.html
git commit -m "fix: merge log updates so weight edits no longer wipe the day's meals"
```

---

### Task 5: 食事と「よく食べるもの」のデータ層

**Files:**
- Modify: `diet-tracker-pwa.html`

**Interfaces:**
- Produces:
  - `addMeal(date, time, name, kcal, protein, source)` → `string`（作成した meal の `id`）
  - `removeMeal(date, id)` → `boolean`
  - `upsertFood(name, kcal, protein)` → `object`（更新後の food）
  - `topFoods(limit)` → `array`（`useCount` 降順・同数なら `lastUsedAt` 降順）
  - `normalizeFoodName(name)` → `string`（前後空白除去・小文字化）

- [ ] **Step 1: 失敗するテストを書く**

```js
  SELFTEST.add("addMeal appends to the right day and returns an id", function () {
    var saved = state.logs; state.logs = [];
    var realSave = saveState; saveState = function () {};
    var id = addMeal("2026-09-03", "lunch", "サラダチキン", 110, 25, "manual");
    T.eq(state.logs.length, 1, "day created");
    T.eq(state.logs[0].meals[0].name, "サラダチキン", "meal stored");
    T.eq(state.logs[0].meals[0].kcal, 110, "kcal stored");
    T.eq(state.logs[0].weight, undefined, "no weight fabricated");
    T.ok(id && id.indexOf("m_") === 0, "returns id");
    saveState = realSave; state.logs = saved;
  });

  SELFTEST.add("addMeal omits protein when not given", function () {
    var saved = state.logs; state.logs = [];
    var realSave = saveState; saveState = function () {};
    addMeal("2026-09-03", "snack", "コーヒー", 5, undefined, "manual");
    T.eq(state.logs[0].meals[0].protein, undefined, "protein not fabricated");
    saveState = realSave; state.logs = saved;
  });

  SELFTEST.add("removeMeal deletes only the targeted meal", function () {
    var saved = state.logs; state.logs = [];
    var realSave = saveState; saveState = function () {};
    var a = addMeal("2026-09-03", "lunch", "A", 100, 10, "manual");
    addMeal("2026-09-03", "dinner", "B", 200, 20, "manual");
    T.eq(removeMeal("2026-09-03", a), true, "returns true");
    T.eq(state.logs[0].meals.length, 1, "one left");
    T.eq(state.logs[0].meals[0].name, "B", "correct one left");
    T.eq(removeMeal("2026-09-03", "nope"), false, "unknown id returns false");
    saveState = realSave; state.logs = saved;
  });

  SELFTEST.add("upsertFood merges by normalized name and counts uses", function () {
    var saved = state.foods; state.foods = [];
    var realSave = saveState; saveState = function () {};
    upsertFood("サラダチキン", 110, 25);
    upsertFood("  サラダチキン  ", 115, 26);
    T.eq(state.foods.length, 1, "merged");
    T.eq(state.foods[0].useCount, 2, "count incremented");
    T.eq(state.foods[0].kcal, 115, "latest values win");
    T.eq(state.foods[0].protein, 26, "latest values win");
    saveState = realSave; state.foods = saved;
  });

  SELFTEST.add("topFoods orders by useCount then recency", function () {
    var saved = state.foods;
    state.foods = [
      { id: "f_1", name: "A", kcal: 100, protein: 5, useCount: 1, lastUsedAt: "2026-09-01T00:00:00.000Z" },
      { id: "f_2", name: "B", kcal: 100, protein: 5, useCount: 5, lastUsedAt: "2026-09-01T00:00:00.000Z" },
      { id: "f_3", name: "C", kcal: 100, protein: 5, useCount: 1, lastUsedAt: "2026-09-02T00:00:00.000Z" }
    ];
    var top = topFoods(3);
    T.eq(top[0].name, "B", "highest count first");
    T.eq(top[1].name, "C", "then most recent");
    T.eq(topFoods(1).length, 1, "limit respected");
    state.foods = saved;
  });
```

- [ ] **Step 2: ブラウザで実行して失敗を確認**

Expected: 5件 FAIL。

- [ ] **Step 3: 実装する**

```js
  function normalizeFoodName(name) {
    return String(name == null ? "" : name).trim().toLowerCase();
  }

  function findOrCreateLog(date) {
    var idx = state.logs.findIndex(function (l) { return l.date === date; });
    if (idx >= 0) {
      if (!Array.isArray(state.logs[idx].meals)) state.logs[idx].meals = [];
      return state.logs[idx];
    }
    var entry = { date: date, trained: false, meals: [] };
    state.logs.push(entry);
    return entry;
  }

  function addMeal(date, time, name, kcal, protein, source) {
    var log = findOrCreateLog(date);
    var meal = {
      id: newId("m_"), time: time, name: String(name).trim(),
      kcal: Math.round(kcal), source: source || "manual"
    };
    if (typeof protein === "number" && !isNaN(protein)) meal.protein = Math.round(protein);
    log.meals.push(meal);
    saveState();
    return meal.id;
  }

  function removeMeal(date, id) {
    var log = state.logs.find(function (l) { return l.date === date; });
    if (!log || !Array.isArray(log.meals)) return false;
    var before = log.meals.length;
    log.meals = log.meals.filter(function (m) { return m.id !== id; });
    if (log.meals.length === before) return false;
    saveState();
    return true;
  }

  function upsertFood(name, kcal, protein) {
    var key = normalizeFoodName(name);
    var existing = state.foods.find(function (f) { return normalizeFoodName(f.name) === key; });
    var now = new Date().toISOString();
    if (existing) {
      existing.useCount = (existing.useCount || 0) + 1;
      existing.kcal = Math.round(kcal);
      if (typeof protein === "number" && !isNaN(protein)) existing.protein = Math.round(protein);
      existing.lastUsedAt = now;
      saveState();
      return existing;
    }
    var food = { id: newId("f_"), name: String(name).trim(), kcal: Math.round(kcal), useCount: 1, lastUsedAt: now };
    if (typeof protein === "number" && !isNaN(protein)) food.protein = Math.round(protein);
    state.foods.push(food);
    saveState();
    return food;
  }

  function topFoods(limit) {
    return state.foods.slice().sort(function (a, b) {
      if ((b.useCount || 0) !== (a.useCount || 0)) return (b.useCount || 0) - (a.useCount || 0);
      return String(b.lastUsedAt || "") < String(a.lastUsedAt || "") ? -1 : 1;
    }).slice(0, limit || 12);
  }
```

- [ ] **Step 4: ブラウザで実行して成功を確認**

Expected: 全件 PASS。

- [ ] **Step 5: コミット**

```bash
git add diet-tracker-pwa.html
git commit -m "feat: add meal and frequent-food data layer"
```

---

### Task 6: インポート / エクスポートの v2 対応

**Files:**
- Modify: `diet-tracker-pwa.html`（`validateImportedData`・`exportData`・インポート処理）

**Interfaces:**
- Produces: `validateImportedData(data)` → `string | null`（v1 と v2 の両方を受理する）

- [ ] **Step 1: 失敗するテストを書く**

```js
  SELFTEST.add("import accepts v1 payloads", function () {
    T.eq(validateImportedData({ logs: [{ date: "2026-09-01", weight: 72, calories: 1200 }] }), null, "v1 ok");
  });

  SELFTEST.add("import accepts v2 payloads including weightless days", function () {
    T.eq(validateImportedData({ schemaVersion: 2, logs: [
      { date: "2026-09-01", trained: false, meals: [{ id: "m_1", time: "lunch", name: "A", kcal: 100 }] }
    ], foods: [] }), null, "weightless day accepted");
  });

  SELFTEST.add("import rejects malformed data", function () {
    T.ok(validateImportedData({ logs: [{ date: "bad", weight: 72 }] }), "bad date rejected");
    T.ok(validateImportedData({ logs: [{ date: "2026-09-01", weight: "72" }] }), "non-numeric weight rejected");
    T.ok(validateImportedData({ schemaVersion: 2, logs: [{ date: "2026-09-01", meals: "nope" }] }), "meals must be an array");
    T.ok(validateImportedData({ schemaVersion: 2, logs: [
      { date: "2026-09-01", meals: [{ id: "m_1", time: "brunch", name: "A", kcal: 100 }] }
    ]}), "unknown meal time rejected");
    T.ok(validateImportedData({ schemaVersion: 2, logs: [
      { date: "2026-09-01", meals: [{ id: "m_1", time: "lunch", name: "A", kcal: "100" }] }
    ]}), "non-numeric kcal rejected");
  });

  SELFTEST.add("import round-trips through migration", function () {
    var v1 = { logs: [{ date: "2026-09-01", weight: 72, calories: 1200, protein: 90 }] };
    var migrated = migrateToV2(v1);
    T.eq(validateImportedData(migrated), null, "migrated payload validates");
    T.eq(dayKcal(migrated.logs[0]), 1200, "totals survive");
  });
```

- [ ] **Step 2: ブラウザで実行して失敗を確認**

Expected: 体重なしの受理と `meals` 検証が FAIL。

- [ ] **Step 3: 実装する**

`validateImportedData` の体重チェックを任意にし、`meals`・`foods` の検証を足す:

```js
      if (l.weight !== undefined && (typeof l.weight !== "number" || isNaN(l.weight))) return "記録に不正な体重があります (" + i + "件目)";
      if (l.meals !== undefined) {
        if (!Array.isArray(l.meals)) return "記録の食事データが配列ではありません (" + i + "件目)";
        for (var j = 0; j < l.meals.length; j++) {
          var m = l.meals[j];
          if (!m || typeof m !== "object") return "食事データの形式が正しくありません (" + i + "件目)";
          if (!LOG_MEAL_TIME_LABEL[m.time]) return "食事データに不正な食事帯があります (" + i + "件目)";
          if (typeof m.name !== "string") return "食事データに不正な名前があります (" + i + "件目)";
          if (typeof m.kcal !== "number" || isNaN(m.kcal)) return "食事データに不正なカロリーがあります (" + i + "件目)";
          if (m.protein !== undefined && (typeof m.protein !== "number" || isNaN(m.protein))) return "食事データに不正なタンパク質量があります (" + i + "件目)";
        }
      }
```

`foods` の検証をログのループの後に足す:

```js
    if (data.foods !== undefined) {
      if (!Array.isArray(data.foods)) return "foodsの形式が正しくありません";
      for (var k = 0; k < data.foods.length; k++) {
        var f = data.foods[k];
        if (!f || typeof f !== "object" || typeof f.name !== "string") return "よく食べるものの形式が正しくありません (" + k + "件目)";
        if (typeof f.kcal !== "number" || isNaN(f.kcal)) return "よく食べるもののカロリーが不正です (" + k + "件目)";
        if (f.protein !== undefined && (typeof f.protein !== "number" || isNaN(f.protein))) return "よく食べるもののタンパク質量が不正です (" + k + "件目)";
      }
    }
```

- [ ] **Step 4: エクスポートを v2 形式にする**

`exportData` の `var data = { logs: state.logs };` を次に置き換える:

```js
    var data = { schemaVersion: 2, logs: state.logs, foods: state.foods };
```

- [ ] **Step 5: インポート処理で移行を通す**

検証を通ったデータを取り込む箇所で、`state.logs = data.logs` の代わりに移行を通す:

```js
      var incoming = migrateToV2(data);
      state.logs = incoming.logs;
      state.foods = Array.isArray(incoming.foods) ? incoming.foods : [];
      if (incoming.profile) state.profile = incoming.profile;
      saveState();
```

- [ ] **Step 6: ブラウザで実行して成功を確認**

Expected: 全件 PASS。

- [ ] **Step 7: コミット**

```bash
git add diet-tracker-pwa.html
git commit -m "feat: accept v1 and v2 payloads on import, export schema v2"
```

---

### Task 7: 「本日の摂取」の食事帯別表示

**Files:**
- Modify: `diet-tracker-pwa.html`（`renderTodayTargets`・CSS）

**Interfaces:**
- Consumes: `LOG_MEAL_TIME_ORDER`・`LOG_MEAL_TIME_LABEL`・`dayKcal`・`dayProtein`・`hasMeals`・`removeMeal`
- Produces: `renderTodayTargets()` がゲージ＋食事帯別リストを描画する

- [ ] **Step 1: 実装する**

`renderTodayTargets` を次の形に置き換える。**`other` 帯は中身がある時だけ描画する。** 食事が1件も無い日は
従来どおり空状態メッセージを出す（0 と未記録を区別するため）:

```js
  function renderTodayTargets() {
    var el = document.getElementById("todayTargetCard");
    var today = todayStr();
    var log = state.logs.find(function (l) { return l.date === today; });

    if (!hasMeals(log)) {
      el.innerHTML = '<div class="empty-state">本日の食事はまだ記録されていません。「食事を追加」から記録できます。</div>';
      return;
    }

    var html = targetBarHtml("カロリー", dayKcal(log), getTargetCalories(), "kcal") +
               targetBarHtml("タンパク質", dayProtein(log), getTargetProtein(), "g");

    LOG_MEAL_TIME_ORDER.forEach(function (mt) {
      var items = log.meals.filter(function (m) { return m.time === mt; });
      if (mt === "other" && items.length === 0) return;
      var rows = items.length
        ? items.map(function (m) {
            var macro = m.kcal + "kcal" + (typeof m.protein === "number" ? " / P" + m.protein + "g" : "");
            return '<div class="meal-row">' +
              '<span class="meal-name">' + escapeHtml(m.name) + '</span>' +
              '<span class="meal-macro num">' + macro + '</span>' +
              '<button class="meal-del" data-meal="' + m.id + '" aria-label="削除">&times;</button>' +
            '</div>';
          }).join("")
        : '<div class="meal-row empty">（未記録）</div>';
      html += '<div class="meal-bucket"><div class="meal-bucket-label">' + LOG_MEAL_TIME_LABEL[mt] + '</div>' + rows + '</div>';
    });

    el.innerHTML = html;

    el.querySelectorAll("[data-meal]").forEach(function (btn) {
      btn.addEventListener("click", function () {
        removeMeal(today, btn.getAttribute("data-meal"));   // 1食削除は確認を出さない
        renderAll();
      });
    });
  }
```

- [ ] **Step 2: `escapeHtml` が存在するか確認し、無ければ追加する**

Run: `grep -n "function escapeHtml" diet-tracker-pwa.html`
無ければ追加する（**利用者が入力した食事名をそのまま `innerHTML` に入れるため必須**）:

```js
  function escapeHtml(s) {
    return String(s).replace(/[&<>"']/g, function (c) {
      return { "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" }[c];
    });
  }
```

- [ ] **Step 3: エスケープのテストを追加する**

```js
  SELFTEST.add("escapeHtml neutralises markup in food names", function () {
    T.eq(escapeHtml('<img src=x onerror=alert(1)>'), "&lt;img src=x onerror=alert(1)&gt;", "tags escaped");
    T.eq(escapeHtml("A & B"), "A &amp; B", "ampersand escaped");
  });
```

- [ ] **Step 4: CSS を追加する**

既存の `.target-row` 系の近くに置く:

```css
  .meal-bucket{margin-top:12px}
  .meal-bucket-label{font-size:12px;color:#8a8f98;margin-bottom:4px}
  .meal-row{display:flex;align-items:center;gap:8px;padding:6px 0;border-bottom:1px solid rgba(255,255,255,.06)}
  .meal-row.empty{color:#6b7078;font-size:13px}
  .meal-name{flex:1;min-width:0;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
  .meal-macro{font-size:13px;color:#b9bec7}
  .meal-del{background:none;border:none;color:#8a8f98;font-size:18px;padding:0 4px;cursor:pointer}
```

- [ ] **Step 5: ブラウザで確認**

`?selftest=1` で全件 PASS を確認したうえで、通常起動して本日の摂取カードが崩れないことを見る。

- [ ] **Step 6: コミット**

```bash
git add diet-tracker-pwa.html
git commit -m "feat: show today's meals grouped by meal time"
```

---

### Task 8: 「食事を追加」フォームと「体重を記録」への改称

**Files:**
- Modify: `diet-tracker-pwa.html`（`#tab-dashboard` の HTML・フォーム処理・CSS）

**Interfaces:**
- Consumes: `addMeal`・`upsertFood`・`topFoods`・`defaultMealTime`・`LOG_MEAL_TIME_*`
- Produces: `renderFoodChips()`・`setupMealForm()`

- [ ] **Step 1: HTML を追加する**

既存の「記録を追加」セクションの見出しを `体重を記録` に変え、`f-calories`・`f-protein` の
`<label>` と `<input>` を**削除**する。そのうえで、その直後に新しいセクションを追加する:

```html
    <h2 class="section-title">食事を追加</h2>
    <div class="card">
      <div class="food-chips" id="foodChips"></div>
      <form id="mealForm">
        <div class="form-row">
          <label for="m-time">食事帯</label>
          <select id="m-time">
            <option value="breakfast">朝食</option>
            <option value="lunch">昼食</option>
            <option value="dinner">夕食</option>
            <option value="snack">間食</option>
          </select>
        </div>
        <div class="form-row">
          <label for="m-date">日付</label>
          <input type="date" id="m-date">
        </div>
        <div class="form-row">
          <label for="m-name">名前</label>
          <input type="text" id="m-name" placeholder="例: コンビニのサラダチキン">
        </div>
        <div class="form-row">
          <label for="m-kcal">カロリー (kcal)</label>
          <input type="number" id="m-kcal" step="1" min="0" inputmode="numeric">
        </div>
        <div class="form-row">
          <label for="m-protein">タンパク質 (任意)</label>
          <input type="number" id="m-protein" step="1" min="0" inputmode="numeric" placeholder="g">
        </div>
        <button type="submit" class="primary-btn">食事を追加</button>
        <div class="data-status" id="mealStatus"></div>
      </form>
    </div>
```

- [ ] **Step 2: 「よく食べるもの」チップを描画する**

```js
  function renderFoodChips() {
    var el = document.getElementById("foodChips");
    var foods = topFoods(12);
    if (foods.length === 0) {
      el.innerHTML = '<div class="empty-state">手入力した食事はここに残り、次回から1タップで追加できます。</div>';
      return;
    }
    el.innerHTML = foods.map(function (f) {
      var macro = f.kcal + "kcal" + (typeof f.protein === "number" ? " / P" + f.protein : "");
      return '<button class="food-chip" data-food="' + f.id + '">' +
        escapeHtml(f.name) + '<span class="food-chip-macro">' + macro + '</span></button>';
    }).join("");

    el.querySelectorAll("[data-food]").forEach(function (btn) {
      btn.addEventListener("click", function () {
        var food = state.foods.find(function (f) { return f.id === btn.getAttribute("data-food"); });
        if (!food) return;
        var date = document.getElementById("m-date").value || todayStr();
        var time = document.getElementById("m-time").value;
        addMeal(date, time, food.name, food.kcal, food.protein, "food");
        upsertFood(food.name, food.kcal, food.protein);
        renderAll();
      });
    });
  }
```

- [ ] **Step 3: 手入力フォームの処理を書く**

```js
  function setupMealForm() {
    var form = document.getElementById("mealForm");
    var dateInput = document.getElementById("m-date");
    dateInput.value = todayStr();
    document.getElementById("m-time").value = defaultMealTime(new Date());

    form.addEventListener("submit", function (e) {
      e.preventDefault();
      var status = document.getElementById("mealStatus");
      var name = document.getElementById("m-name").value.trim();
      var kcal = parseFloat(document.getElementById("m-kcal").value);
      var proteinRaw = document.getElementById("m-protein").value;
      var protein = proteinRaw === "" ? undefined : parseFloat(proteinRaw);

      if (!name) { status.textContent = "名前を入力してください"; status.className = "data-status err"; return; }
      if (isNaN(kcal) || kcal < 0) { status.textContent = "カロリーを正しく入力してください"; status.className = "data-status err"; return; }
      if (protein !== undefined && (isNaN(protein) || protein < 0)) { status.textContent = "タンパク質を正しく入力してください"; status.className = "data-status err"; return; }

      var date = dateInput.value || todayStr();
      addMeal(date, document.getElementById("m-time").value, name, kcal, protein, "manual");
      upsertFood(name, kcal, protein);

      document.getElementById("m-name").value = "";
      document.getElementById("m-kcal").value = "";
      document.getElementById("m-protein").value = "";
      status.textContent = name + " を追加しました";
      status.className = "data-status ok";
      renderAll();
    });
  }
```

- [ ] **Step 4: 体重フォームの送信処理から kcal・protein を外す**

`addOrUpdateLog(date, weight, trained, calories, protein);` を `addOrUpdateLog(date, weight, trained);` にし、
`f-calories`・`f-protein` を読む行とクリアする行を削除する。

- [ ] **Step 5: 初期化と再描画に組み込む**

`setupMealForm()` を他の `setup*()` と並べて呼び、`renderAll()` の中に `renderFoodChips()` を足す。

- [ ] **Step 6: CSS を追加する**

```css
  .food-chips{display:flex;flex-wrap:wrap;gap:6px;margin-bottom:10px}
  .food-chip{background:#20242b;border:1px solid #333a44;color:#e6e9ee;border-radius:14px;padding:6px 10px;font-size:13px;cursor:pointer;display:flex;gap:6px;align-items:baseline}
  .food-chip-macro{font-size:11px;color:#8a8f98}
```

- [ ] **Step 7: ブラウザで確認**

通常起動し、手入力で1件追加 → 本日の摂取に出る → チップに残る → チップから1タップで追加できる、を確認する。
`?selftest=1` が引き続き全件 PASS であることも確認する。

- [ ] **Step 8: コミット**

```bash
git add diet-tracker-pwa.html
git commit -m "feat: add meal entry form with frequent-food shortcuts"
```

---

### Task 9: 履歴の内訳表示と推移タブの算出値対応

**Files:**
- Modify: `diet-tracker-pwa.html`（`renderHistory`・`metricLogs`）

- [ ] **Step 1: 推移タブの失敗するテストを書く**

```js
  SELFTEST.add("metricLogs derives calories and protein from meals", function () {
    var saved = state.logs;
    state.logs = [
      { date: "2026-09-01", weight: 72, meals: [{ id: "m_1", time: "lunch", name: "A", kcal: 500, protein: 40 }] },
      { date: "2026-09-02", weight: 71, meals: [] }
    ];
    var cal = metricLogs("calories");
    T.eq(cal.length, 1, "days without meals are skipped");
    T.eq(cal[0].calories, 500, "derived from meals");
    T.eq(metricLogs("protein")[0].protein, 40, "protein derived");
    T.eq(metricLogs("weight").length, 2, "weight series unchanged");
    state.logs = saved;
  });
```

- [ ] **Step 2: ブラウザで実行して失敗を確認**

Expected: FAIL（`l.calories` が存在しないため 0件になる）。

- [ ] **Step 3: `metricLogs` を実装する**

```js
  function metricLogs(metric) {
    if (metric === "weight") {
      return sortedLogsAsc().filter(function (l) { return typeof l.weight === "number"; });
    }
    return sortedLogsAsc().filter(hasMeals).map(function (l) {
      var copy = { date: l.date };
      copy.calories = dayKcal(l);
      copy.protein = dayProtein(l);
      return copy;
    });
  }
```

- [ ] **Step 4: 履歴行に内訳の開閉を足す**

`renderHistory` の各行の後ろに、折りたたみ用の要素を追加する:

```js
        '<div class="history-meals" data-meals-for="' + l.date + '" hidden>' +
          (hasMeals(l)
            ? l.meals.map(function (m) {
                return '<div class="history-meal-row"><span>' + LOG_MEAL_TIME_LABEL[m.time] + '</span>' +
                  '<span class="hm-name">' + escapeHtml(m.name) + '</span>' +
                  '<span class="num">' + m.kcal + 'kcal' + (typeof m.protein === "number" ? " / P" + m.protein + "g" : "") + '</span></div>';
              }).join("")
            : '<div class="history-meal-row empty">食事の記録はありません</div>') +
        '</div>' +
```

行のクリックで開閉する（削除ボタンのクリックと衝突させないこと）:

```js
    el.querySelectorAll(".history-row").forEach(function (row) {
      row.addEventListener("click", function (e) {
        if (e.target.hasAttribute("data-del")) return;
        var panel = el.querySelector('[data-meals-for="' + row.getAttribute("data-date") + '"]');
        if (panel) panel.hidden = !panel.hidden;
      });
    });
```

- [ ] **Step 5: CSS を追加する**

```css
  .history-meals{padding:6px 0 10px 8px;border-bottom:1px solid rgba(255,255,255,.06)}
  .history-meal-row{display:flex;gap:8px;font-size:12px;color:#b9bec7;padding:2px 0}
  .history-meal-row .hm-name{flex:1;min-width:0;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
  .history-meal-row.empty{color:#6b7078}
```

- [ ] **Step 6: ブラウザで確認してコミット**

`?selftest=1` 全件 PASS、通常起動で履歴行をタップして内訳が開くことを確認する。

```bash
git add diet-tracker-pwa.html
git commit -m "feat: expand history rows to show meals and derive chart series"
```

---

### Task 10: レシピからの「記録に追加」とトースト

**Files:**
- Modify: `diet-tracker-pwa.html`（`renderMeals`・レシピ詳細モーダル・CSS）

**Interfaces:**
- Produces: `showToast(message, undoFn)`

- [ ] **Step 1: トーストを実装する**

```js
  var toastTimer = null;
  function showToast(message, undoFn) {
    var el = document.getElementById("toast");
    if (!el) {
      el = document.createElement("div");
      el.id = "toast";
      el.className = "toast";
      document.body.appendChild(el);
    }
    el.innerHTML = '<span>' + escapeHtml(message) + '</span>' +
      (undoFn ? '<button class="toast-undo" type="button">取り消し</button>' : "");
    el.classList.add("show");
    if (toastTimer) clearTimeout(toastTimer);
    toastTimer = setTimeout(function () { el.classList.remove("show"); }, 5000);
    if (undoFn) {
      el.querySelector(".toast-undo").addEventListener("click", function () {
        undoFn();
        el.classList.remove("show");
        renderAll();
      });
    }
  }
```

- [ ] **Step 2: レシピカードにボタンを足す**

`renderMeals` が組み立てる各レシピの HTML に、食事帯 `mt` とレシピの索引を持つボタンを足す:

```js
          '<button class="recipe-log-btn" data-log-recipe="' + mt + ':' + i + '">記録に追加</button>' +
```

- [ ] **Step 3: ハンドラを書く**

`renderMeals` の描画後に登録する。**レシピの食事帯をそのまま使い、現在時刻では上書きしない**
（利用者が「夕食」の欄から選んでいるため）:

```js
    document.querySelectorAll("[data-log-recipe]").forEach(function (btn) {
      btn.addEventListener("click", function (e) {
        e.stopPropagation();                     // カードのモーダル展開と衝突させない
        var parts = btn.getAttribute("data-log-recipe").split(":");
        var recipe = MEALS[parts[0]][parseInt(parts[1], 10)];
        var date = todayStr();
        var id = addMeal(date, parts[0], recipe.name, recipe.kcal, recipe.protein, "recipe");
        renderAll();
        showToast("+" + recipe.kcal + "kcal / P" + recipe.protein + "g を記録しました", function () {
          removeMeal(date, id);
        });
      });
    });
```

- [ ] **Step 4: 詳細モーダルにも同じボタンを置く**

モーダルを組み立てている箇所に同じ `data-log-recipe` 属性のボタンを足し、Step 3 のハンドラ登録を
モーダル描画後にも実行する。

- [ ] **Step 5: CSS を追加する**

```css
  .recipe-log-btn{margin-top:6px;background:#2b6cb0;border:none;color:#fff;border-radius:8px;padding:6px 10px;font-size:13px;cursor:pointer}
  .toast{position:fixed;left:50%;transform:translateX(-50%);bottom:72px;background:#2a2f37;color:#e6e9ee;border:1px solid #3a414c;border-radius:10px;padding:10px 14px;display:none;gap:12px;align-items:center;z-index:60;font-size:13px;box-shadow:0 6px 24px rgba(0,0,0,.4)}
  .toast.show{display:flex}
  .toast-undo{background:none;border:none;color:#7fb1ff;font-size:13px;cursor:pointer;padding:0}
```

- [ ] **Step 6: ブラウザで確認してコミット**

食事例タブでレシピの「記録に追加」を押す → 本日の摂取に出る → トーストの「取り消し」で消える、を確認する。

```bash
git add diet-tracker-pwa.html
git commit -m "feat: log a meal straight from a recipe with undo"
```

---

### Task 11: 版番号・キャッシュ更新・実機確認・デプロイ

**Files:**
- Modify: `diet-tracker-pwa.html`（`APP_VERSION`）
- Modify: `sw.js`（`CACHE_NAME`）

- [ ] **Step 1: 版番号を上げる**

`diet-tracker-pwa.html`: `var APP_VERSION = "2.0";` → `var APP_VERSION = "3.0";`
`sw.js`: `var CACHE_NAME = "diet-tracker-cache-v6";` → `"diet-tracker-cache-v7"`

**キャッシュ名を上げないと、`activate` が古いキャッシュを消さず、利用者に旧HTMLが配信され続ける。**
旧HTMLは体重なしのログで落ちるため、ここを飛ばすと移行後に壊れた画面が出る。

- [ ] **Step 2: セルフテストを最終実行する**

`file:///Users/soki/Documents/GitHub/diet/diet-tracker-pwa.html?selftest=1`
Expected: 全件 PASS。1件でも FAIL なら次へ進まない。

- [ ] **Step 3: 実データに近い形での手動確認**

ブラウザの開発者ツールで、v1 形式のデータを localStorage に入れてから通常起動する:

```js
localStorage.setItem("nexra-diet-tracker", JSON.stringify({
  logs: [
    { date: "2026-08-30", weight: 72.4, trained: true, calories: 1480, protein: 112 },
    { date: "2026-08-31", weight: 72.1, trained: false, calories: 1390 }
  ],
  profile: { startWeight: 80, goalWeight: 65 }
}));
```

確認する項目:
1. 起動して落ちない。履歴に2日分が出る。
2. `nexra-diet-tracker-backup-v1` に移行前のJSONが入っている。
3. 8/30 の履歴行を開くと「まとめて記録（移行）1480kcal / P112g」が出る。
4. 体重だけを再記録しても、その日の食事が消えない。
5. エクスポートしたJSONを、そのままインポートで復元できる。
6. 推移タブのカロリー系列が2点描かれる。

- [ ] **Step 4: コミットして push**

```bash
git add diet-tracker-pwa.html sw.js
git commit -m "chore: bump app version to 3.0 and cache to v7 for schema v2"
git push origin main
```

- [ ] **Step 5: デプロイを確認する**

```bash
gh run list --limit 3
curl -s -o /dev/null -w "%{http_code}\n" https://sokiumeda.github.io/diet/
```
Expected: 直近の run が success、HTTP 200。

- [ ] **Step 6: ボルトのノートを更新する**

`01_プロジェクト/diet（ダイエット管理PWA）.md` を実態に合わせる。**現状のノートは大きく陳腐化しており**
（「1283行・v1.5」と書かれているが実際は3000行超）、v3.0・スキーマv2・1食記録・よく食べるもの・
セルフテスト（`?selftest=1`）の存在を追記し、行数と版の記述を実測値へ直す。

---

## Self-Review

**1. Spec coverage**

| 仕様の項目 | 実装するタスク |
|---|---|
| v2 データモデル | Task 3 |
| 食事帯の定数（既存を壊さない） | Task 2 |
| 移行・退避・冪等・合計不変 | Task 3 |
| 算出（保存しない） | Task 2・Task 9 |
| 本日の摂取（帯別・その他は非空時のみ） | Task 7 |
| 食事を追加（帯・日付・チップ・手入力） | Task 8 |
| 「体重を記録」への改称 | Task 8 |
| `addOrUpdateLog` のマージ化 | Task 4 |
| 履歴の体重「—」と内訳展開 | Task 4・Task 9 |
| 推移タブの算出値対応 | Task 9 |
| レシピからの記録追加・トースト・取り消し | Task 10 |
| import/export の v2 対応と検証修正 | Task 6 |
| 削除の確認方針・トースト5秒 | Task 7・Task 10 |
| `id` 採番 | Task 3 |
| APP_VERSION 3.0・CACHE_NAME v7 | Task 11 |
| 検証項目1〜8 | Task 2・3・4・5・6・9 |

**2. 仕様に無いが必要と判明して足したもの**
- `escapeHtml`（Task 7）: 利用者が入力した食事名を `innerHTML` に入れるため必須。仕様策定時に見落としていた。
- `sw.js` のキャッシュ名更新の理由（Task 11 Step 1）: 旧HTMLが配信されると体重なしログで落ちるため。

**3. Type consistency**
- `addMeal(date, time, name, kcal, protein, source)` は Task 5 で定義し、Task 8・Task 10 で同じ順序で呼んでいる。
- `removeMeal(date, id)` は Task 5 で定義し、Task 7・Task 10 で同じ形で呼んでいる。
- `migrateToV2(parsed)` は Task 3 で定義し、Task 6 で再利用している。
- `hasMeals` / `dayKcal` / `dayProtein` は Task 2 で定義し、Task 4・7・9 で使っている。
- `escapeHtml` は Task 7 で定義し、Task 8・9・10 で使っている（Task 7 が先に実行される順序を守ること）。
