# Lo-Fi ギャラリー LP 実装仕様書 v2
**対象: Claude Code（Firebase統合・拡張可能構成）**

---

## 1. 実装要件

| 項目 | 値 |
|------|-----|
| 出力形式 | **単一HTMLファイル**（HTML + CSS + JS インライン） |
| 外部依存 | **Firebase JS SDK（CDN）のみ許可** |
| バックエンド | Firebase（Firestore + Storage）無料枠 |
| サムネイル | Firebase Storage参照 → 未登録時はCSSプレースホルダー表示 |
| DLボタン | Firebase Storage URL経由DL → 未設定時はUI表示のみ |
| 対応言語 | 日本語（初期表示）/ 英語 |
| レスポンシブ | 必須（モバイル〜デスクトップ） |

---

## 2. アーキテクチャ概要

```
┌─────────────────────────────────────────────────────┐
│                    単一 HTML ファイル                  │
│                                                     │
│  ┌───────────┐  ┌────────────┐  ┌───────────────┐   │
│  │  UI層      │  │  データ層   │  │  Firebase層    │  │
│  │ (HTML/CSS) │←→│ (JS State) │←→│ (Firestore/   │  │
│  │            │  │            │  │  Storage)     │  │
│  └───────────┘  └────────────┘  └───────────────┘   │
│                       ↕                              │
│              ┌─────────────────┐                     │
│              │ フォールバック    │                     │
│              │ (ローカルデータ) │                     │
│              └─────────────────┘                     │
└─────────────────────────────────────────────────────┘
```

### 動作モード（自動判定）

| モード | 条件 | 動作 |
|--------|------|------|
| **Firebase接続モード** | Firebase初期化成功 + Firestoreからデータ取得可 | Firestore/Storageから全データ取得 |
| **フォールバックモード** | Firebase未設定 or 接続失敗 or データ0件 | HTML内のローカルデータで表示 |

**重要**: どちらのモードでもUIは完全に同一。ユーザーには違いが見えない。

---

## 3. Firebase 構成

### 3.1 使用サービス（無料枠のみ）

| サービス | 用途 | 無料枠上限 |
|----------|------|-----------|
| Firestore | パッケージデータ管理 | 1GiB保存 / 50K読取/日 |
| Storage | サムネイル画像 + DLファイル保存 | 5GB保存 / 1GB DL/日 |
| Hosting | （任意）HTML配信 | 10GB保存 / 360MB/日 |

### 3.2 Firebase設定（HTMLに埋め込み）

```javascript
// ======================================================
// Firebase 設定（ここを書き換えるだけで接続先変更）
// ======================================================
const FIREBASE_CONFIG = {
  apiKey: "",            // ← 自分のキーを入力
  authDomain: "",        // ← 自分のドメインを入力
  projectId: "",         // ← 自分のIDを入力
  storageBucket: "",     // ← 自分のバケットを入力
  messagingSenderId: "", // ← 自分のIDを入力
  appId: ""              // ← 自分のIDを入力
};
// 上記が空 or 無効 → 自動でフォールバックモードになる
// ======================================================
```

### 3.3 Firestore コレクション設計

**コレクション名**: `packages`

```
/packages/{packageId}
```

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| `id` | string | ○ | 一意ID（ドキュメントIDと同一） |
| `title_ja` | string | ○ | 日本語タイトル |
| `title_en` | string | ○ | 英語タイトル |
| `desc_ja` | string | ○ | 日本語説明 |
| `desc_en` | string | ○ | 英語説明 |
| `genres` | array\<string\> | ○ | ジャンルID配列（例: `["rain","study"]`） |
| `durationMinutes` | number | ○ | 再生時間（分）※現在は全て60 |
| `size` | string | ○ | ファイルサイズ表示（例: `"1.2GB"`） |
| `badge` | string | − | `"popular"` / `"new"` / `null` |
| `thumbnailPath` | string | − | Storage内パス（例: `"thumbnails/rain-lofi.jpg"`） |
| `downloadPath` | string | − | Storage内パス（例: `"downloads/rain-lofi.zip"`） |
| `order` | number | − | 表示順（昇順ソート、未設定は末尾） |
| `visible` | boolean | − | `false`で非表示（未設定は`true`扱い） |
| `createdAt` | timestamp | − | 作成日時 |

### 3.4 Storage フォルダ構造

```
gs://<bucket>/
├── thumbnails/          ← サムネイル画像
│   ├── rain-lofi-work.jpg
│   ├── fire-lofi-healing.jpg
│   └── ...
└── downloads/           ← DL用ファイル
    ├── rain-lofi-work.zip
    ├── fire-lofi-healing.zip
    └── ...
```

### 3.5 Firestore セキュリティルール

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // packagesコレクション: 誰でも読み取り可、書き込み不可
    match /packages/{packageId} {
      allow read: if true;
      allow write: if false;  // Firebaseコンソールから直接管理
    }
  }
}
```

### 3.6 Storage セキュリティルール

```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    // サムネイル: 誰でも読み取り可
    match /thumbnails/{fileName} {
      allow read: if true;
      allow write: if false;
    }
    // ダウンロード: 誰でも読み取り可
    match /downloads/{fileName} {
      allow read: if true;
      allow write: if false;
    }
  }
}
```

---

## 4. データ取得フロー

### JS実装パターン

```javascript
// ======================================================
// データ取得（Firebase優先 → フォールバック）
// ======================================================
async function loadPackages() {
  // 1. Firebase接続を試行
  if (isFirebaseConfigured()) {
    try {
      const snapshot = await getDocs(
        query(collection(db, "packages"), orderBy("order"))
      );
      if (!snapshot.empty) {
        return snapshot.docs
          .map(doc => doc.data())
          .filter(pkg => pkg.visible !== false);
      }
    } catch (e) {
      console.warn("Firestore取得失敗、フォールバックへ", e);
    }
  }

  // 2. フォールバック: ローカルデータを返す
  return LOCAL_PACKAGES;
}
```

### サムネイル解決ロジック

```javascript
// ======================================================
// サムネイルURL解決（Storage優先 → プレースホルダー）
// ======================================================
async function resolveThumbnail(pkg) {
  if (pkg.thumbnailPath && isFirebaseConfigured()) {
    try {
      return await getDownloadURL(ref(storage, pkg.thumbnailPath));
    } catch (e) {
      // ファイル未アップロード → プレースホルダーに落ちる
    }
  }
  return null; // → CSSプレースホルダー表示
}
```

### ダウンロード処理

```javascript
// ======================================================
// ダウンロード（Storage URL取得 → ブラウザDL）
// ======================================================
async function handleDownload(pkg) {
  if (pkg.downloadPath && isFirebaseConfigured()) {
    try {
      const url = await getDownloadURL(ref(storage, pkg.downloadPath));
      const a = document.createElement("a");
      a.href = url;
      a.download = "";
      a.click();
      return;
    } catch (e) {
      console.warn("DLファイル未設定:", pkg.id);
    }
  }
  // フォールバック: 準備中メッセージ
  alert(currentLang === "ja"
    ? "準備中です。もうしばらくお待ちください。"
    : "Coming soon. Please wait."
  );
}
```

---

## 5. 画面構成（上→下の順）

```
┌──────────────────────────────────────────┐
│  ヘッダー（ロゴ + 言語トグル）              │
├──────────────────────────────────────────┤
│  検索バー（中央配置）                      │
├──────────────────────────────────────────┤
│  ジャンルフィルター（横並びボタン）          │
├──────────────────────────────────────────┤
│  ローディング表示（Firebase読込中のみ）     │
│  または                                   │
│  パッケージカード グリッド（4列）           │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐       │
│  │Card │ │Card │ │Card │ │Card │       │
│  └─────┘ └─────┘ └─────┘ └─────┘       │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐       │
│  │Card │ │Card │ │Card │ │Card │       │
│  └─────┘ └─────┘ └─────┘ └─────┘       │
└──────────────────────────────────────────┘
```

---

## 6. ヘッダー

| 要素 | 仕様 |
|------|------|
| ロゴテキスト | `Lo-Fi Gallery` 左寄せ |
| 言語トグル | 右端配置、`JA` / `EN` 切替ボタン |
| 切替動作 | ページ遷移なし、全テキスト即時切替 |

---

## 7. 検索バー

| 項目 | 値 |
|------|-----|
| 配置 | 中央、幅 `max-width: 600px` |
| 形状 | 角丸（`border-radius: 24px`） |
| プレースホルダー JA | `キーワードで検索` |
| プレースホルダー EN | `Search by keyword` |
| 検索対象 | パッケージタイトル、説明文、ジャンルタグ（現在の表示言語で照合） |
| 検索タイミング | リアルタイム（入力ごとにフィルタ） |

---

## 8. ジャンルフィルター

### 配置
- 検索バー直下、中央寄せ横並び
- 横スクロールなし（折り返し可）

### ジャンル定義（6種・固定）

| ID | JA表示 | EN表示 | プレースホルダー配色 | 絵文字 |
|----|--------|--------|---------------------|--------|
| `rain` | 雨 | Rain | `#6B8FAD` | 🌧️ |
| `fire` | 火 | Fire | `#C07040` | 🔥 |
| `anime` | アニメ | Anime | `#8B6BAD` | 🎬 |
| `study` | 勉強 | Study | `#7AAD6B` | 📚 |
| `night` | 夜 | Night | `#4A5580` | 🌙 |
| `chill` | チル | Chill | `#AD8B6B` | ☕ |

### 動作
- 複数選択可（AND検索）
- 選択中はボタンをアクティブスタイルに変更
- 再クリックで選択解除
- 検索バーとの併用可（AND条件）

---

## 9. パッケージカード

### グリッドレイアウト

| 画面幅 | 列数 |
|--------|------|
| 1200px以上 | 4列 |
| 768px〜1199px | 3列 |
| 480px〜767px | 2列 |
| 479px以下 | 1列 |

- `gap: 24px`
- 最大幅 `1280px` 中央配置

### カード構造（上→下の順、厳守）

```
┌─────────────────────────┐
│ [バッジ]                 │
│                         │
│   サムネイル（1:1）       │
│   実画像 or             │
│   プレースホルダー       │
│                         │
├─────────────────────────┤
│ タイトル（1行）           │
│ ⏱ 60分  💾 1.2GB        │
│ [雨] [勉強]              │
│ ┌─────────────────────┐ │
│ │   ダウンロード        │ │
│ └─────────────────────┘ │
└─────────────────────────┘
```

### カードスタイル

| 項目 | 値 |
|------|-----|
| 比率 | カード全体は自動高さ、サムネ部分のみ1:1 |
| 角丸 | `border-radius: 12px` |
| 背景 | `#FFFFFF` |
| 影 | `box-shadow: 0 2px 8px rgba(0,0,0,0.08)` |
| hover | `box-shadow: 0 4px 16px rgba(0,0,0,0.12)` |

---

## 10. カード内要素の詳細

### 10.1 サムネイル

**表示優先順位**:
1. Firebase Storageの実画像（`thumbnailPath` が存在 + URL取得成功）
2. CSSプレースホルダー（ジャンル配色グラデーション + 絵文字）

```
実画像あり → <img src="Storage URL" />
実画像なし → <div class="placeholder"> + 絵文字 + グラデーション背景
```

- プレースホルダーは先頭ジャンルの配色を使用
- 画像読込中はプレースホルダーを表示、読込完了で差し替え

### 10.2 ステータスバッジ

| 項目 | 値 |
|------|-----|
| 位置 | サムネイル左上、`position: absolute` |
| 種類 | `popular`（人気/Popular）または `new`（新作/New） |
| popular色 | 背景 `#E85D5D` 白文字 |
| new色 | 背景 `#5DAD5D` 白文字 |
| 形状 | 角丸 `border-radius: 6px`、`padding: 2px 10px` |
| font-size | `12px`、`font-weight: 600` |
| 表示 | 1カードにつき最大1種。未指定なら非表示 |

### 10.3 タイトル

- 1行、`overflow: hidden; text-overflow: ellipsis; white-space: nowrap`
- `font-size: 15px`、`font-weight: 600`
- i18n対応（`title_ja` / `title_en`）

### 10.4 情報行

- `font-size: 13px`、`color: #888`
- 形式: `⏱ 60分  💾 1.2GB`（EN: `⏱ 60 min  💾 1.2GB`）
- 再生時間は `durationMinutes` フィールドから取得
- 容量は `size` フィールドから取得

### 10.5 ジャンルタグ

- 角丸ピル型（`border-radius: 12px`）
- `font-size: 12px`
- 背景: 各ジャンル配色を `opacity: 0.15` で使用
- 文字色: 各ジャンル配色そのまま
- `padding: 2px 10px`
- 複数並列可

### 10.6 ダウンロードボタン

| 項目 | 値 |
|------|-----|
| テキスト JA | `ダウンロード` |
| テキスト EN | `Download` |
| 幅 | `width: 100%` |
| 背景色 | `#5B9BD5` |
| hover | `#4A8AC4` |
| 文字色 | `#FFFFFF` |
| 角丸 | `border-radius: 20px` |
| font-size | `14px`、`font-weight: 600` |
| 動作 | `handleDownload(pkg)` を呼び出す（セクション4参照） |

---

## 11. ローディング・状態表示

| 状態 | 表示 |
|------|------|
| Firebase読込中 | カードエリアに小さなスピナー or 「読み込み中...」 |
| フォールバック動作中 | 何も表示しない（ユーザーには見せない） |
| フィルタ結果0件 | JA: `該当するパッケージが見つかりません` / EN: `No packages found` |

---

## 12. フォールバック用ローカルデータ（8件）

Firebase未接続時に表示するデータ。HTML内のJSに直接埋め込む。

```javascript
const LOCAL_PACKAGES = [
  {
    id: "rain-lofi-work",
    title_ja: "雨音×Lo-Fi 作業用",
    title_en: "Rain × Lo-Fi Work",
    desc_ja: "雨の日の作業BGMに最適なLo-Fiテンプレ",
    desc_en: "Lo-Fi template perfect for rainy day work sessions",
    genres: ["rain", "study"],
    durationMinutes: 60,
    size: "1.2GB",
    badge: "popular",
    thumbnailPath: "",
    downloadPath: "",
    order: 1
  },
  {
    id: "fire-lofi-healing",
    title_ja: "焚き火×Lo-Fi 癒し系",
    title_en: "Bonfire × Lo-Fi Healing",
    desc_ja: "焚き火の揺らぎで心を落ち着けるテンプレ",
    desc_en: "Calming template with flickering bonfire ambience",
    genres: ["fire", "chill"],
    durationMinutes: 60,
    size: "900MB",
    badge: "new",
    thumbnailPath: "",
    downloadPath: "",
    order: 2
  },
  {
    id: "night-city-lofi",
    title_ja: "夜の街×Lo-Fi 散歩用",
    title_en: "Night City × Lo-Fi Walk",
    desc_ja: "夜の街並みを歩くような映像テンプレ",
    desc_en: "Template with nighttime city stroll vibes",
    genres: ["night", "anime"],
    durationMinutes: 60,
    size: "1.5GB",
    badge: "popular",
    thumbnailPath: "",
    downloadPath: "",
    order: 3
  },
  {
    id: "cafe-lofi-reading",
    title_ja: "カフェ×Lo-Fi 読書用",
    title_en: "Café × Lo-Fi Reading",
    desc_ja: "カフェの静かな空間で読書に集中するテンプレ",
    desc_en: "Template for focused reading in a quiet café setting",
    genres: ["rain", "study"],
    durationMinutes: 60,
    size: "1.1GB",
    badge: "new",
    thumbnailPath: "",
    downloadPath: "",
    order: 4
  },
  {
    id: "sakura-lofi-spring",
    title_ja: "桜×Lo-Fi 春の作業用",
    title_en: "Sakura × Lo-Fi Spring",
    desc_ja: "桜舞う春の風景で作業がはかどるテンプレ",
    desc_en: "Spring sakura scenery to boost your productivity",
    genres: ["chill", "study"],
    durationMinutes: 60,
    size: "1.3GB",
    badge: "popular",
    thumbnailPath: "",
    downloadPath: "",
    order: 5
  },
  {
    id: "forest-lofi-nature",
    title_ja: "森林×Lo-Fi 自然音",
    title_en: "Forest × Lo-Fi Nature",
    desc_ja: "森の中にいるような自然音テンプレ",
    desc_en: "Immersive forest nature sounds template",
    genres: ["rain", "chill"],
    durationMinutes: 60,
    size: "1.0GB",
    badge: "new",
    thumbnailPath: "",
    downloadPath: "",
    order: 6
  },
  {
    id: "ocean-lofi-waves",
    title_ja: "海×Lo-Fi 波の音",
    title_en: "Ocean × Lo-Fi Waves",
    desc_ja: "波の音で集中力を高めるテンプレ",
    desc_en: "Ocean wave sounds for deep concentration",
    genres: ["chill", "night"],
    durationMinutes: 60,
    size: "1.4GB",
    badge: "popular",
    thumbnailPath: "",
    downloadPath: "",
    order: 7
  },
  {
    id: "bedroom-lofi-sleep",
    title_ja: "ベッドルーム×Lo-Fi 睡眠用",
    title_en: "Bedroom × Lo-Fi Sleep",
    desc_ja: "穏やかな寝室の映像で眠りにつくテンプレ",
    desc_en: "Gentle bedroom ambience for falling asleep",
    genres: ["night", "chill"],
    durationMinutes: 60,
    size: "1.1GB",
    badge: "new",
    thumbnailPath: "",
    downloadPath: "",
    order: 8
  }
];
```

---

## 13. i18n 実装方針

```javascript
const i18n = {
  ja: {
    searchPlaceholder: "キーワードで検索",
    download: "ダウンロード",
    duration: "分",
    loading: "読み込み中...",
    noResults: "該当するパッケージが見つかりません",
    comingSoon: "準備中です。もうしばらくお待ちください。",
    genres: {
      rain: "雨", fire: "火", anime: "アニメ",
      study: "勉強", night: "夜", chill: "チル"
    },
    badges: { popular: "人気", new: "新作" }
  },
  en: {
    searchPlaceholder: "Search by keyword",
    download: "Download",
    duration: "min",
    loading: "Loading...",
    noResults: "No packages found",
    comingSoon: "Coming soon. Please wait.",
    genres: {
      rain: "Rain", fire: "Fire", anime: "Anime",
      study: "Study", night: "Night", chill: "Chill"
    },
    badges: { popular: "Popular", new: "New" }
  }
};
```

---

## 14. フィルタ・検索ロジック

```
表示パッケージ = 全パッケージ
  .filter(pkg => pkg.visible !== false)         // 非表示除外
  .sort((a, b) => (a.order ?? 999) - (b.order ?? 999))  // order昇順
  .filter(ジャンルフィルタ条件)  // 選択ジャンルすべてを含む（AND）
  .filter(検索キーワード条件)    // title / desc / genre表示名に部分一致
```

---

## 15. デザイントークン

```css
:root {
  --bg-main: #F5F3EF;
  --bg-card: #FFFFFF;
  --bg-header: #F5F3EF;
  --text-primary: #2D2D2D;
  --text-secondary: #888888;
  --accent-blue: #5B9BD5;
  --accent-blue-hover: #4A8AC4;
  --search-bg: #FFFFFF;
  --search-border: #E0DDD8;
  --search-border-focus: #5B9BD5;
  --genre-btn-bg: #FFFFFF;
  --genre-btn-border: #E0DDD8;
  --genre-btn-text: #555555;
  --genre-btn-active-bg: #2D2D2D;
  --genre-btn-active-text: #FFFFFF;
  --font-family: -apple-system, BlinkMacSystemFont, "Segoe UI",
                 "Hiragino Sans", "Noto Sans JP", sans-serif;
  --radius-card: 12px;
  --radius-button: 20px;
  --radius-search: 24px;
  --radius-tag: 12px;
  --radius-badge: 6px;
  --grid-gap: 24px;
  --content-max-width: 1280px;
  --section-padding: 24px;
}
```

---

## 16. Firebase CDN読み込み

```html
<!-- Firebase App + Firestore + Storage（モジュラーSDK v10系） -->
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.14.1/firebase-app.js";
  import { getFirestore, collection, getDocs, query, orderBy }
    from "https://www.gstatic.com/firebasejs/10.14.1/firebase-firestore.js";
  import { getStorage, ref, getDownloadURL }
    from "https://www.gstatic.com/firebasejs/10.14.1/firebase-storage.js";

  // ... アプリ初期化・データ取得ロジック
</script>
```

---

## 17. 運用手順（Firebase管理）

### パッケージ追加

1. Firebaseコンソール → Firestore → `packages` コレクション → ドキュメント追加
2. 全フィールドをセクション3.3の通り入力
3. サムネイル画像を Storage `thumbnails/` にアップロード
4. DLファイルを Storage `downloads/` にアップロード
5. Firestoreの `thumbnailPath` / `downloadPath` に対応パスを記入
6. ページリロードで反映

### パッケージ非表示

- Firestoreで `visible: false` を設定するだけ（削除不要）

### 表示順変更

- `order` フィールドの数値を変更

### サムネイル差し替え

- Storage上の同じパスに上書きアップロード → 即反映（CDNキャッシュの遅延あり）

---

## 18. 実装チェックリスト

### 基本動作
- [ ] 単一HTMLファイルとして完結している
- [ ] Firebase CDNのみ外部依存
- [ ] JA/EN切替が即時反映される
- [ ] ジャンルフィルター複数選択（AND）が動作する
- [ ] 検索バーのリアルタイムフィルタが動作する
- [ ] 検索 + ジャンルフィルタの併用が動作する
- [ ] 該当なし時に空状態メッセージが表示される
- [ ] レスポンシブ（4列→3列→2列→1列）が動作する
- [ ] デザイントークン（CSS変数）が適用されている
- [ ] hover時のカード影変化が動作する

### Firebase接続モード
- [ ] FIREBASE_CONFIG入力済みでFirestoreからデータ取得できる
- [ ] Storageのサムネイル画像が表示される
- [ ] DLボタンでStorageファイルがダウンロードされる
- [ ] `visible: false` のパッケージが非表示になる
- [ ] `order` 順でソートされる

### フォールバックモード
- [ ] FIREBASE_CONFIG空でもエラーなく表示される
- [ ] ローカルデータ8件でUI完全表示される
- [ ] プレースホルダーサムネイルが表示される
- [ ] DLボタンで「準備中」メッセージが出る
- [ ] コンソールにエラーが出ない

---

## 19. 禁止事項

- Firebase Auth の実装（今回は不要）
- Firestoreへの書き込み処理（読み取り専用）
- 複数ファイル分割
- Firebase以外の外部API呼び出し
- ユーザー投稿・コメント・AI機能

---

## 20. 将来拡張ポイント（今回は実装しない）

| 拡張 | 対応方法 |
|------|---------|
| 有料パッケージ追加 | Firestoreに `price` / `isFree` フィールド追加 |
| 決済連携 | Stripe連携、DLボタンを決済フローに差替え |
| DL数カウント | Cloud Functions でカウンター更新 |
| ユーザー認証 | Firebase Auth 追加 |
| サブスク機能 | Firebase Auth + Stripe Subscription |

---

## 21. 参考：画面イメージ

アップロード画像（`Lo-Fi動画テンプレ.png`）のUIを再現すること。
特に以下の視覚要素を忠実に反映する：

- 明るいベージュ系背景（`#F5F3EF`）
- カード内サムネイルはLo-Fi風イラスト調（プレースホルダーで代用）
- ジャンルボタンは白背景＋ボーダー、選択時は反転
- ダウンロードボタンは青系角丸ピル
- 全体の余白はゆったり
