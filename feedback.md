# テンプレート修正フィードバック

## 1. スライド背景のグラデーション修正

### 問題
- 左上に弧状のグラデーションが不自然に見切れていた

### 原因
- `.slidev-page` の `radial-gradient` が左上起点 (`at 0% 0%`) の楕円 (`80% 50%`) で、色の遷移が急すぎた

### 修正内容 (`style.css`)
```css
/* Before */
radial-gradient(ellipse 80% 50% at 0% 0%, oklch(0.6 0.2 220 / 0.35) 0%, transparent 50%)

/* After */
radial-gradient(ellipse 150% 40% at 50% 0%, oklch(0.5 0.15 220 / 0.35) 0%, oklch(0.5 0.15 220 / 0.1) 40%, transparent 70%)
```

- 起点を上辺中央 (`at 50% 0%`) に変更し、上部全体に横長に広がるように
- 楕円を `150% 40%` にして横に広く縦は浅く
- 中間色停止を追加 (`40%` 地点) して滑らかにフェード
- 彩度を `0.2` → `0.15` に抑えて落ち着いた青みに

---

## 2. `slide-gradient-bg` クラスの削除

### 問題
- `class: slide-gradient-bg` を適用したスライドでのみ、左上のグラデーションが不自然に表示されていた

### 原因
- Slidev の `class:` 指定は `.slidev-layout` (子要素) に適用される
- `.slide-gradient-bg` はほぼ不透明な背景 (`color-mix(in oklch, var(--color-primary) 5%, var(--color-bg))`) を設定していた
- これが親 `.slidev-page` の radial-gradient を覆い隠し、135deg 対角の隙間から下層グラデーションが弧状に透けて見えていた

### 修正内容
- `style.css` から `.slide-gradient-bg` クラス定義を削除
- `.slidev-page` の全スライド共通グラデーションが既にあるため不要

### テンプレート反映時の注意
- `slides.md` 等で使用している `class: slide-gradient-bg` の指定は残っていても無害だが、不要なので削除推奨
- デザインシステムドキュメント（下記セクション3参照）からも `slide-gradient-bg` の記述を削除する

---

## 3. 要修正: デザインシステム・AI参照ドキュメント

上記の変更に伴い、以下のドキュメントを現状のスライドデザインに合わせて更新する必要がある。

### 3-1. `slide-gradient-bg` の記述削除

クラスが廃止されたため、各ドキュメントから記述・使用例を削除する。

| ファイル | 修正内容 |
|---------|---------|
| `design_system/ai-guide.md` | テンプレート例の `class: slide-gradient-bg` を削除、早見表からエントリ削除 |
| `design_system/utility-classes.md` | `slide-gradient-bg` セクション全体を削除 |
| `design_system/components.md` | `slide-gradient-bg` セクション・使用例を削除 |
| `design_system/layouts.md` | セクション区切りの例から `class: slide-gradient-bg` を削除、説明文を修正 |
| `design_system/examples.md` | Before/After例から `class: slide-gradient-bg` を削除 |
| `content/README.md` | セクション区切りの説明から `slide-gradient-bg` を削除 |
| `workflows/slide-generation.md` | テンプレート例から `class: slide-gradient-bg` を削除 |
| `.claude/commands/gen-slide.md` | セクション区切りの指示を修正 |
| `README.md` | 使用例から `class: slide-gradient-bg` を削除 |

### 3-2. 背景グラデーションの仕様記述を更新

グラデーションの仕様が変わったため、記述を現状に合わせる。

| ファイル | 修正内容 |
|---------|---------|
| `design_system/tokens.md` | 背景グラデーションの定義値を新しい値に更新 |
| `design_system/ai-guide.md` | 背景に関する説明を「上辺中央から横長に広がる青グラデーション」に修正 |

### 3-3. セクション区切りスライドのパターン修正

`slide-gradient-bg` が不要になったため、セクション区切りの推奨パターンを更新する。

**Before:**
```
layout: center
class: slide-gradient-bg
```

**After:**
```
layout: center
```

背景グラデーションは `.slidev-page` に全スライド共通で適用されるため、追加クラスは不要。

| ファイル | 修正内容 |
|---------|---------|
| `design_system/ai-guide.md` | セクション区切りテンプレートから `class: slide-gradient-bg` を削除 |
| `design_system/layouts.md` | セクション区切りパターンの説明を修正 |
| `CLAUDE.md` | 判断フローチャートの「セクション区切り → layout: center + gradient-bg」を修正 |
