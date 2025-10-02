# ストレージ構造の詳細

## 📁 ディレクトリ構造

### 開発環境（ローカルストレージ）

```
album-video-main/
└── api/
    ├── .storage/                         # ← ここに全ての写真が保存される
    │   ├── album_clx001/                 # アルバム1のディレクトリ
    │   │   ├── a1b2c3d4-...uuid....jpg   # 写真1
    │   │   ├── e5f6g7h8-...uuid....png   # 写真2
    │   │   ├── i9j0k1l2-...uuid....heic  # 写真3
    │   │   └── m3n4o5p6-...uuid....jpg   # 写真4
    │   │
    │   ├── album_clx002/                 # アルバム2のディレクトリ
    │   │   ├── q7r8s9t0-...uuid....jpg
    │   │   └── u1v2w3x4-...uuid....png
    │   │
    │   └── slideshows/                   # 生成された動画
    │       ├── album_clx001/
    │       │   └── job_abc123.mp4
    │       └── album_clx002/
    │           └── job_def456.mp4
    │
    ├── src/
    ├── prisma/
    └── package.json
```

### 本番環境（S3ストレージ）

```
s3://album-media-bucket/
├── album_clx001/
│   ├── a1b2c3d4-e5f6-7890-abcd-ef1234567890.jpg
│   ├── e5f6g7h8-i9j0-1234-5678-90abcdef1234.png
│   └── i9j0k1l2-m3n4-5678-9012-3456789abcde.heic
├── album_clx002/
│   ├── q7r8s9t0-u1v2-3456-7890-abcdef123456.jpg
│   └── u1v2w3x4-y5z6-7890-1234-567890abcdef.png
└── slideshows/
    ├── album_clx001/
    │   └── job_abc123.mp4
    └── album_clx002/
        └── job_def456.mp4
```

## 🗂️ ファイル命名規則

### パターン

```
{albumId}/{uuid}{extension}
```

### 例

```
album_clx001/a1b2c3d4-e5f6-7890-abcd-ef1234567890.jpg
│           │                                       │
│           │                                       └─ 拡張子（元のファイルから）
│           └───────────────────────────────────────── UUID v4（ランダム生成）
└───────────────────────────────────────────────────── アルバムID
```

### 具体例

**アップロードされたファイル**: `family_photo.jpg`

**保存される場所**:
- ローカル: `.storage/album_abc123/f1e2d3c4-b5a6-7890-1234-567890abcdef.jpg`
- S3: `s3://bucket/album_abc123/f1e2d3c4-b5a6-7890-1234-567890abcdef.jpg`

**データベースに記録される情報**:
```json
{
  "id": "media_xyz789",
  "albumId": "album_abc123",
  "storageKey": "local:album_abc123/f1e2d3c4-b5a6-7890-1234-567890abcdef.jpg",
  "mime": "image/jpeg",
  "caption": null,
  "tags": []
}
```

## 🔄 アップロード時の処理フロー

### ステップ1: ファイル受信
```typescript
// クライアントからのリクエスト
POST /media/upload
Content-Type: multipart/form-data

file: [binary data]
albumId: "album_abc123"
caption: "家族写真"
tags: "family,2024"
```

### ステップ2: ファイル名生成
```typescript
// services/media.service.ts

const ext = path.extname(input.file.originalname);  // ".jpg"
const key = `${album.id}/${randomUUID()}${ext}`;
// → "album_abc123/f1e2d3c4-b5a6-7890-1234-567890abcdef.jpg"
```

### ステップ3: ディレクトリ作成
```typescript
// libs/storage.ts

const file = path.join(localDir, key);
// → ".storage/album_abc123/f1e2d3c4-b5a6-7890-1234-567890abcdef.jpg"

const dir = path.dirname(file);
// → ".storage/album_abc123"

await fs.mkdir(dir, { recursive: true });
// → ディレクトリが存在しなければ自動作成
```

### ステップ4: ファイル書き込み
```typescript
await fs.writeFile(file, buffer);
// → バイナリデータをファイルに書き込み
```

### ステップ5: DB記録
```typescript
const media = await prisma.media.create({
  data: {
    albumId: "album_abc123",
    ownerId: "user_xyz",
    storageKey: "local:album_abc123/f1e2d3c4-...",
    mime: "image/jpeg",
    caption: "家族写真",
    tags: ["family", "2024"]
  }
});
```

## 📊 データベースとファイルの対応

### データベース（MySQL）

| id | albumId | storageKey | mime | caption |
|----|---------|------------|------|---------|
| media_1 | album_001 | local:album_001/uuid1.jpg | image/jpeg | 海の写真 |
| media_2 | album_001 | local:album_001/uuid2.png | image/png | 山の写真 |
| media_3 | album_002 | local:album_002/uuid3.jpg | image/jpeg | 街の写真 |

### ファイルシステム

```
.storage/
├── album_001/
│   ├── uuid1.jpg  ← media_1
│   └── uuid2.png  ← media_2
└── album_002/
    └── uuid3.jpg  ← media_3
```

## 🎯 なぜこの構造なのか？

### 1. アルバムごとにディレクトリを分ける理由

✅ **整理しやすい**: アルバム単位でファイルを管理
✅ **削除が簡単**: アルバム削除時にディレクトリごと削除可能
✅ **バックアップが容易**: アルバム単位でバックアップ
✅ **パフォーマンス**: 1ディレクトリに大量ファイルを入れない

### 2. UUIDを使う理由

✅ **衝突防止**: 同名ファイルでも問題なし
✅ **セキュリティ**: ファイル名から内容を推測できない
✅ **URL安全**: 特殊文字を含まない
✅ **グローバル一意性**: システム全体で一意

### 3. 拡張子を保持する理由

✅ **ファイルタイプ判別**: 拡張子でファイル種別が分かる
✅ **互換性**: 他のツールで開きやすい
✅ **デバッグ**: 開発時に内容が推測しやすい

## 🔐 セキュリティ対策

### 1. パストラバーサル攻撃の防止

```typescript
// ❌ 危険な実装例
const key = req.body.filename;  // "../../../etc/passwd"
await fs.writeFile(key, buffer);  // システムファイルを上書き！

// ✅ 安全な実装（実際のコード）
const key = `${album.id}/${randomUUID()}${ext}`;
// → "album_001/uuid.jpg" のみ（../ は含まれない）
```

### 2. ファイルサイズ制限

```typescript
const upload = multer({ 
  limits: { fileSize: 50 * 1024 * 1024 }  // 50MB制限
});
```

### 3. MIME タイプ検証（今後の拡張）

```typescript
// 実装例（必要に応じて追加）
const allowedTypes = ['image/jpeg', 'image/png', 'image/heic'];
if (!allowedTypes.includes(file.mimetype)) {
  throw createError(400, 'Invalid file type');
}
```

## 📈 容量管理

### ディスク使用量の確認

```bash
# ローカルストレージの使用量
du -sh api/.storage/

# アルバムごとの使用量
du -sh api/.storage/*/
```

### クリーンアップ

```bash
# 特定アルバムのファイル削除
rm -rf api/.storage/album_xxx/

# 孤立ファイルの検出（DBに記録がないファイル）
# → 専用スクリプトの作成を推奨
```

## 🚀 パフォーマンス最適化

### 1. サムネイル生成（今後の拡張）

```
.storage/
└── album_001/
    ├── uuid1.jpg          # オリジナル（5MB）
    ├── uuid1_thumb.jpg    # サムネイル（100KB）
    └── uuid1_medium.jpg   # 中サイズ（500KB）
```

### 2. CDN配信（本番環境）

```
CloudFront CDN
    ↓
S3 Bucket
    ↓
オリジン: album-media/album_001/uuid.jpg
```

## 📝 まとめ

| 項目 | 説明 |
|-----|------|
| **保存場所** | `.storage/` または S3 |
| **ディレクトリ** | アルバムごとに分離 |
| **ファイル名** | UUID + 拡張子 |
| **管理方法** | DBとファイルシステムの2層管理 |
| **セキュリティ** | UUID、サイズ制限、権限チェック |

この構造により、**写真は安全に、整理された状態で保存**されます！

