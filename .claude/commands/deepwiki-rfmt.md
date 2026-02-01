rfmtプロジェクトのDeepWiki知識ベースを参照して、スライド作成やプロジェクト理解を支援してください。

## rfmtプロジェクト概要

rfmtは**Rustコアを備えたRubyコードフォーマッター**です。Ruby（フロントエンド）とRust（コアエンジン）のハイブリッドアーキテクチャを採用し、高速かつ一貫したRubyコードフォーマットを実現します。

- **リポジトリ**: https://github.com/fs0414/rfmt
- **DeepWiki**: https://deepwiki.com/fs0414/rfmt

---

## 設計原則

- **意見的（Opinionated）**: 最小限の設定で一貫したスタイルを強制
- **冪等性（Idempotent）**: 同じ入力に対して常に同じ出力を生成
- **コメント保護**: コメントの配置とフォーマットを維持
- **高速処理**: Rustコアにより優れたパフォーマンスを実現

---

## アーキテクチャ: 3層構造

```
Ruby レイヤー（フロントエンド）
  ├─ CLI インターフェース（Rfmt::CLI）
  ├─ Prism パーサー統合（PrismBridge）
  ├─ AST シリアル化（JSON変換）
  └─ 設定管理（Rfmt::Config）

FFI 境界（Magnus + rb-sys）

Rust レイヤー（コア）
  ├─ AST 構造体（Node, NodeType, Location）
  ├─ Emitter（フォーマットエンジン）
  └─ Config構造体（設定適用）
```

### データフロー

1. **解析フェーズ**: Prism が Ruby ソースを AST に変換
2. **変換フェーズ**: PrismBridge が Prism ノードを内部表現に変換
3. **シリアル化**: AST とコメントを JSON に変換
4. **FFI 転送**: Magnus 経由でデータ転送
5. **フォーマット処理**: Rust Emitter が形式化されたコードを生成
6. **コメント挿入**: 適切な位置にコメントを再挿入

---

## Ruby層の詳細

### CLI（Rfmt::CLI）

- エントリーポイント: `exe/rfmt`
- コマンド: `format`、`check`、`init`
- ファイルの並列処理を調整
- 終了コード: 0（正常）、1（エラー）、2（フォーマット必要）

### PrismBridge

- Prism.parse() で Ruby AST を生成
- PrismNodeExtractor でメタデータ抽出（クラス名、メソッド名、パラメータ数等）
- convert_node() で rfmt 内部表現に変換
- JSON.generate() で FFI 転送用にシリアライズ

### 設定管理（Rfmt::Config）

設定ファイル検出順序:
1. カレントディレクトリ（`.rfmt.yml`、`.rfmt.yaml`、`rfmt.yml`、`rfmt.yaml`）
2. 親ディレクトリ（ルートまで走査）
3. ホームディレクトリ（`~/.rfmt.yml`）
4. ビルトインデフォルト

設定オプション:

| オプション | デフォルト | 範囲 |
|-----------|----------|------|
| `line_length` | 100 | 40-500 |
| `indent_width` | 2 | 1-8 |
| `indent_style` | spaces | spaces/tabs |
| `quote_style` | double | double/single/consistent |

### キャッシュシステム（Rfmt::Cache）

- SHA256ハッシュでファイル変更を追跡
- `.rfmt_cache.json` に永続化
- 変更されていないファイルをスキップ
- 大規模プロジェクトでも172-191msの一定実行時間を維持

---

## Rust層の詳細

### AST構造

Node構造体の主要フィールド:
- `node_type`: NodeType列挙体（80以上のバリアント）
- `location`: ソース内の行・列・バイトオフセット
- `children`: 子ノードのベクトル
- `metadata`: ノード固有のデータ（HashMap）
- `comments`: 関連するコメント
- `formatting`: インデント情報

### ノード型分類（80以上）

| カテゴリ | 例 |
|---------|-----|
| 構造 | ProgramNode、StatementsNode |
| 定義 | ClassNode、ModuleNode、DefNode |
| 制御フロー | IfNode、CaseNode、WhileNode、ForNode |
| 例外処理 | BeginNode、RescueNode、EnsureNode |
| 式・呼び出し | CallNode、LambdaNode、BlockNode |
| リテラル | StringNode、IntegerNode、ArrayNode |
| 変数 | LocalVariableReadNode、InstanceVariableWriteNode |

### Emitter（フォーマットエンジン）

2つの戦略を組み合わせ:

1. **構造化フォーマット**: クラス、メソッド、条件分岐など → ASTメタデータから構文を再構築
2. **ソース抽出フォールバック**: 複雑な構文（パラメータ、ラムダ等）→ バイトオフセットで元のテキストを保持

---

## コメント保護メカニズム

3段階プロセス:

1. **収集**: AST全体から再帰的にすべてのコメントを収集
2. **位置追跡**: 行番号でソート、3つの位置タイプに分類
   - Leading: ノード前の行
   - Trailing: コードと同じ行の末尾
   - Inner: ノード内部
3. **戦略的再挿入**:
   - `emit_comments_before()`: 行番号マッチングでノード前に挿入
   - `emit_trailing_comments()`: 同一行に配置
   - 未出力コメント: 終了時に追加（消失防止）

---

## FFI境界メカニズム

Magnus フレームワークが安全な Ruby-Rust 相互運用を実現:

- **型変換**: JSONシリアライゼーションによる言語非依存のデータ転送
- **メモリ安全性**: Ruby オブジェクトライフタイム管理
- **エラー処理**: Rust エラーを Ruby 例外に変換
- **文字列エンコーディング**: UTF-8対応

---

## パフォーマンス

ベンチマーク結果:
- 単一ファイル: 約191ms（RuboCopの7.2倍高速）
- フルプロジェクト: RuboCopの25.4倍高速
- ファイル数に関わらず一定の処理時間（172-191ms）
- キャッシュにより変更なしファイルをスキップ

---

## ファイル構成

```
rfmt/
├─ exe/rfmt                     # CLI 実行ファイル
├─ lib/rfmt/
│  ├─ cli.rb                    # コマンド処理
│  ├─ prism_bridge.rb           # AST シリアル化
│  ├─ config.rb                 # 設定管理
│  └─ cache.rb                  # ファイルキャッシュ
└─ ext/rfmt/src/
   ├─ ast/mod.rs                # AST 構造定義
   ├─ emitter/mod.rs            # フォーマットエンジン
   └─ config/mod.rs             # Rust 設定構造体
```

---

## 技術スタック

| 技術 | 用途 |
|------|------|
| Ruby 3.0+ | フロントエンド、CLI |
| Prism | Ruby パーサー gem |
| Magnus + rb-sys | Ruby-Rust FFI フレームワーク |
| Rust 1.70+ | コアエンジン（ソースビルド時） |
| serde_json | JSON 逆シリアル化 |

対応プラットフォーム: Linux x86_64、macOS ARM64、macOS x86_64、Windows x64

---

## インストール・使用方法

```bash
# RubyGems（推奨）
gem install rfmt

# Bundler
bundle add rfmt

# ソースビルド
git clone https://github.com/fs0414/rfmt.git
cd rfmt && bundle exec rake compile
```

```bash
rfmt lib/user.rb          # ファイルフォーマット
rfmt lib/user.rb --diff   # 差分確認
rfmt check lib/user.rb    # チェックのみ
rfmt init                 # 設定初期化
```

Ruby API:
```ruby
require 'rfmt'
formatted = Rfmt.format("class Foo;end")
```

---

## 実行手順

### ユーザーがスライド作成を依頼した場合

1. **上記の知識ベースを参照**して正確な情報を使用
2. **`design_system/ai-guide.md`を参照**してテンプレートとコンポーネントを選択
3. **`workflows/slide-generation.md`を参照**して変換ルールに従う
4. スライドを生成し、ユーザーに確認してもらう

### ユーザーがrfmtについて質問した場合

1. 上記の知識ベースから正確な情報を提供
2. 不明な点がある場合は `https://deepwiki.com/fs0414/rfmt` をWebFetchで参照
3. ソースコードの詳細が必要な場合は `https://github.com/fs0414/rfmt` を参照

### DeepWikiの情報を更新したい場合

以下のページをWebFetchで最新情報を取得:
- `https://deepwiki.com/fs0414/rfmt` （トップ）
- `https://deepwiki.com/fs0414/rfmt/1-overview` （概要）
- `https://deepwiki.com/fs0414/rfmt/2-architecture` （アーキテクチャ）
- `https://deepwiki.com/fs0414/rfmt/3-ruby-layer` （Ruby層）
- `https://deepwiki.com/fs0414/rfmt/4-rust-core` （Rust層）
- `https://deepwiki.com/fs0414/rfmt/5-comment-preservation` （コメント保護）
- `https://deepwiki.com/fs0414/rfmt/6-configuration-system` （設定システム）
- `https://deepwiki.com/fs0414/rfmt/7-installation-and-usage` （インストール・使用方法）

---

## 重要な注意事項

- **入力された情報のみを使用**: DeepWikiや上記知識ベースにない情報を捏造しない
- **抽象的表現・誇張表現を追加しない**: 提供された技術情報を正確に転記
- **既存コンポーネントを優先**: 手動HTMLよりCoverSlide、TwoColumnLayout等を使用
- **コードブロックには言語指定**: `ruby`、`rust`、`yaml`、`bash` を適切に指定
