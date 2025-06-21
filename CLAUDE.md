# PDF Vision Rename - Claude Code Instructions

## 🚨 重要な実装ルール

### ❌ 絶対にやってはいけないこと
1. **ダウンロード機能の使用禁止**
   - `a.download = filename` でのダウンロードは厳禁
   - `URL.createObjectURL()` → `a.click()` でのファイル保存は禁止
   - 新しいファイルを作成してダウンロードすることは一切しない

2. **元ファイルを変更しない処理**
   - 元のPDFファイルはそのまま残す処理は禁止
   - コピーを作成する処理は禁止

### ✅ 必ず守るべき実装方針

#### 1. 直接リネーム機能
```javascript
// 正しい実装例
if ('move' in currentFileHandle) {
    // Chrome 125+ のネイティブmove()メソッドを使用
    await currentFileHandle.move(newFilename);
} else if (currentDirectoryHandle) {
    // フォールバック: コピー→削除方式
    const file = await currentFileHandle.getFile();
    const newHandle = await currentDirectoryHandle.getFileHandle(newFilename, { create: true });
    const writableStream = await newHandle.createWritable();
    await writableStream.write(file);
    await writableStream.close();
    
    // 元のファイルを削除（これが重要！）
    await currentDirectoryHandle.removeEntry(currentFile.name);
}
```

#### 2. フォルダ選択の権限設定
```javascript
// 必ずreadwrite権限で取得
const directoryHandle = await window.showDirectoryPicker({ mode: 'readwrite' });
currentDirectoryHandle = directoryHandle;
```

#### 3. PDFプレビュー表示（左側パネル）
- `pdf.js`ライブラリを使用
- `<canvas>`要素に第1ページを描画
- 左側パネルに表示して、右側にメタデータフォーム配置

```javascript
// PDF表示の実装例
if (file.type === 'application/pdf') {
    const pdf = await pdfjsLib.getDocument({data: arrayBuffer}).promise;
    const page = await pdf.getPage(1);
    const viewport = page.getViewport({scale: 1});
    
    canvas.width = viewport.width;
    canvas.height = viewport.height;
    
    await page.render({
        canvasContext: canvas.getContext('2d'),
        viewport: viewport
    }).promise;
}
```

## 🎯 アプリケーションの要件

### 基本機能
1. **フォルダ選択機能**
   - File System Access APIでフォルダを選択
   - PDF/画像ファイルを自動検出
   - 一括処理モード

2. **AI解析機能**
   - OpenAI Vision APIでPDF/画像を解析
   - 日付、会社名、サービス名、金額を抽出
   - JSONフォーマットでレスポンス取得

3. **UI構成**
   ```
   ┌─────────────────────────────────────┐
   │           ヘッダー                    │
   ├─────────────┬───────────────────────┤
   │             │                       │
   │   PDFプレビュー │    メタデータフォーム      │
   │   (canvas)   │                       │
   │             │  ┌─ 日付              │
   │             │  ├─ 会社名            │
   │             │  ├─ サービス名         │
   │             │  ├─ 金額              │
   │             │  └─ [リネーム確定]     │
   └─────────────┴───────────────────────┘
   ```

### ファイル命名規則
```
{日付}_{会社名}_{サービス名}_{金額}.pdf
例: 2024-01-15_株式会社サンプル_コンサルティング料_50000.pdf
```

## 🔧 技術仕様

### 対応ブラウザ
- Chrome 125+ (File System Access API + move()メソッド)
- Edge 125+ (Chromiumベース)

### 依存ライブラリ
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
```

### API設定
```javascript
// OpenAI Vision API
const response = await fetch('https://api.openai.com/v1/chat/completions', {
    model: "gpt-4-vision-preview",
    messages: [{
        role: "user",
        content: [{
            type: "text",
            text: "この請求書・領収書から以下の情報を抽出してください。JSONフォーマットで回答してください：{\"date\": \"YYYY-MM-DD\", \"company\": \"会社名\", \"service\": \"サービス・商品名\", \"amount\": \"金額（数字のみ）\"}"
        }, {
            type: "image_url",
            image_url: { url: base64DataUrl }
        }]
    }]
});
```

## 🛠️ 開発・修正時の注意点

### コード修正時のチェックリスト
- [ ] File System Access APIの`mode: 'readwrite'`が設定されているか
- [ ] `move()`メソッドまたはコピー→削除の実装があるか
- [ ] ダウンロード処理（`a.download`）が含まれていないか
- [ ] PDFプレビューが左側に正しく表示されるか
- [ ] メタデータフォームが右側に配置されているか
- [ ] エラーハンドリングが適切に実装されているか

### 既知の問題と対処法
1. **`move()`メソッド未対応ブラウザ**
   - フォールバック: `copy + delete`方式を実装済み

2. **権限エラー**
   - `showDirectoryPicker({ mode: 'readwrite' })`で解決済み

3. **PDF.js読み込みエラー**
   - CDNのバージョン固定で安定化済み

## 📁 プロジェクト構成

```
PDF_Vision_Rename/
├── pdf_vision_final.html      # メインアプリケーション
├── launch_pdf_vision.bat      # Windows起動用バッチファイル
├── .gitignore                 # Git除外設定
└── CLAUDE.md                  # このファイル（Claude Code用指示書）
```

## 🎯 今後の拡張可能性

### 実装可能な追加機能
1. **バッチ処理の進捗表示**
2. **ファイル名テンプレートのカスタマイズ**
3. **OCR精度向上のための前処理**
4. **複数ページPDFの対応**
5. **エクセル/CSV出力機能**

---

**🚨 重要**: このアプリケーションの核心は「直接リネーム」機能です。ダウンロードではなく、ローカルファイルシステム上での真のファイル名変更を実現することが最優先事項です。この要件を満たさない修正は一切受け入れられません。