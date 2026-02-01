# rfmt Ruby Layer

---

## rfmt 全体アーキテクチャ

### レイヤー構成

```
+--------------------------------------------------+
|           Ruby Layer                              |
|  CLI / Configuration / Cache / Editor Integration |
|  PrismBridge (パース + AST変換)                    |
+--------------------------------------------------+
|           FFI Bridge (Magnus)                     |
|  format_code(source, json) / parse_to_json / ...  |
+--------------------------------------------------+
|           Rust Layer                              |
|  AST処理 / Emitter (コード生成) / Config読込       |
+--------------------------------------------------+
```

### データフロー

```
Ruby source code
  |
  v  [Ruby] Prism.parse(source)
Prism AST (Ruby objects)
  |
  v  [Ruby] PrismBridge.serialize_ast_with_comments
JSON string
  |
  v  [Rust] PrismAdapter.parse_json → Internal AST
  v  [Rust] Emitter.emit
Formatted Ruby code (String)
  |
  v  [Ruby] 結果をファイル書き込み or 標準出力
```

---

## Ruby Layer に何が実装されているか

### 1. PrismBridge - パース & AST変換

- ファイル: `lib/rfmt/prism_bridge.rb`, `lib/rfmt/prism_node_extractor.rb`
- 役割: Ruby Prism パーサーを呼び出し、ASTをRust側が受け取れるJSON形式に変換
- 80以上のPrismノードタイプに対応した子ノード抽出ロジック
- 各ノードからメタデータを抽出 (クラス名、メソッド名、パラメータ数、etc.)
- ロケーション情報の抽出 (行番号、カラム、オフセット)
- コメント情報の収集とシリアライズ

### 2. CLI - コマンドラインインターフェース

- ファイル: `lib/rfmt/cli.rb`
- Thor ベースのCLI
- コマンド: format / check / version / config / cache / init
- 並列処理の自動判定 (ファイル数・サイズに基づくヒューリスティクス)
- diff表示 (unified / side_by_side / color)
- プログレス表示

### 3. Configuration - 設定管理

- ファイル: `lib/rfmt/configuration.rb`
- YAML設定ファイルの探索・読込・バリデーション
- ファイルglobパターンによるinclude/exclude
- デフォルト設定とのマージ

### 4. Cache - キャッシュシステム

- ファイル: `lib/rfmt/cache.rb`
- mtime (ファイル更新日時) ベースの変更検知
- `~/.cache/rfmt/cache.json` にJSONで永続化
- clear / prune / stats 操作

### 5. NativeExtensionLoader - ネイティブ拡張ロード

- ファイル: `lib/rfmt/native_extension_loader.rb`
- Ruby 3.0〜3.3+ のバージョン別パス対応
- ロード失敗時の詳細エラーメッセージとワークアラウンド提示

### 6. Ruby LSP Integration - エディタ連携

- ファイル: `lib/ruby_lsp/rfmt/addon.rb`, `lib/ruby_lsp/rfmt/formatter_runner.rb`
- Ruby LSP の Addon として登録
- format-on-save でRfmt.formatを呼び出し

### 7. エントリポイント & エラー定義

- ファイル: `lib/rfmt.rb`
- `Rfmt.format(source)` : 全体のオーケストレーション
- エラー階層: `Error` > `RfmtError` > `ValidationError`
- `Rfmt::Config` モジュール: 設定ファイルinit/find/load

---

## なぜRubyに実装しているか

### Prism パーサーとの親和性

- PrismはRuby標準のパーサーGem
- Ruby側で `Prism.parse(source)` を呼ぶのが最も自然
- Rustから Prism を呼ぶには Ruby VM を経由するFFIが必要で複雑になる
- Ruby側でPrism ASTを走査し、Rust側が処理しやすいJSON形式に正規化する役割

### エコシステム活用

- CLI: Thor gem で宣言的なコマンド定義
- 設定: YAML パース、Dir.glob によるファイル探索が Ruby の得意分野
- diff表示: diffy / diff-lcs gem の活用
- 並列処理: parallel gem (プロセスベース)

### Ruby LSP との統合要件

- Ruby LSP の Addon は Ruby で書く必要がある
- `FormatterRunner` インターフェースに準拠する必要がある

### Gem としての配布

- `rfmt.gemspec` + `ext/rfmt/extconf.rb` による標準的なnative extension gem構造
- `gem install rfmt` で Rust 拡張含めてビルド&インストール
- rb_sys / magnus による Ruby-Rust FFI の標準的なパターン

### パフォーマンスが不要な部分の切り分け

- ファイルI/O、設定読込、キャッシュ管理は低頻度・軽量処理
- ボトルネックになるAST処理・コード生成のみRustに委譲
- Ruby で十分な速度が出る処理を無理にRustで書かない判断

---

## どのように実装されているか

### Ruby-Rust 間のFFI境界

```ruby
# Ruby側 (lib/rfmt.rb)
def self.format(source)
  prism_json = PrismBridge.parse(source)  # Ruby: パース+JSON化
  format_code(source, prism_json)          # Rust: フォーマット実行
end
```

```rust
// Rust側 (ext/rfmt/src/lib.rs)
fn format_ruby_code(ruby: &Ruby, source: String, json: String) -> Result<String, Error> {
    let parser = PrismAdapter::new();
    let ast = parser.parse(&json)?;          // JSONからInternal ASTへ
    let mut emitter = Emitter::with_source(config, source);
    emitter.emit(&ast)                       // フォーマット済みコード生成
}
```

- Magnus (magnus crate) によるRuby-Rust FFI
- `Rfmt` モジュールに `format_code`, `parse_to_json`, `rust_version` を定義
- RubyのString ↔ RustのString は Magnus が自動変換

### PrismBridge の AST変換パイプライン

```
Prism::ProgramNode (Ruby object)
  |
  v  convert_node(node)
{
  node_type: "program_node",          # snake_case変換
  location: { start_line, ... },      # 位置情報
  children: [ ... ],                  # 再帰的に子ノード変換
  metadata: { name: "Foo", ... },     # ノード固有の情報
  comments: [],                       # コメント情報
  formatting: { multiline: true, ... } # フォーマットヒント
}
```

- 80+ ノードタイプの case文による網羅的な子ノード抽出
- `extract_metadata` でノード固有情報 (クラス名、メソッド名、パラメータ、三項演算子判定など) を抽出
- `extract_location` でheredocのclosing_locまで考慮した正確な位置情報
- `PrismNodeExtractor` モジュールで `respond_to?` ベースの安全なアクセス

### CLI の処理フロー

```
rfmt [FILES]
  |
  v  Configuration.discover (YAML読込)
  v  files_to_format (glob展開)
  v  Cache.needs_formatting? (mtimeチェック)
  v  should_use_parallel? (ファイル数/サイズ判定)
  |
  +-- sequential: files.map { format_single_file }
  +-- parallel:   Parallel.map(files, in_processes: N) { format_single_file }
  |
  v  format_single_file
     File.read → Rfmt.format(source) → 結果比較
  |
  v  handle_results (書き込み / diff表示 / check結果)
  v  Cache.save
```

### ネイティブ拡張のビルド & ロード

```
ext/rfmt/extconf.rb
  → create_rust_makefile('rfmt/rfmt')
  → cargo build --release
  → rfmt.bundle (macOS) / rfmt.so (Linux)

NativeExtensionLoader
  → Ruby 3.3+: lib/rfmt/3.3/rfmt.bundle
  → fallback:  lib/rfmt/rfmt.bundle
```

---

## Ruby Layer のファイル一覧

| ファイル | 行数概算 | 役割 |
|---------|---------|------|
| `lib/rfmt.rb` | 170行 | エントリポイント、Rfmt.format、Config モジュール |
| `lib/rfmt/prism_bridge.rb` | 470行 | Prism AST → JSON 変換 (コア) |
| `lib/rfmt/prism_node_extractor.rb` | 115行 | ノード情報の安全な抽出ヘルパー |
| `lib/rfmt/cli.rb` | 450行 | Thor ベース CLI |
| `lib/rfmt/configuration.rb` | 95行 | YAML設定管理 |
| `lib/rfmt/cache.rb` | 112行 | mtime ベースキャッシュ |
| `lib/rfmt/native_extension_loader.rb` | 138行 | ネイティブ拡張ロード |
| `lib/rfmt/version.rb` | 5行 | バージョン定義 |
| `lib/ruby_lsp/rfmt/addon.rb` | 20行 | Ruby LSP アドオン登録 |
| `lib/ruby_lsp/rfmt/formatter_runner.rb` | 26行 | LSP フォーマッタ実行 |
| `exe/rfmt` | 24行 | CLI エントリポイント |
| `exe/rfmt_fast` | 12行 | 高速起動 CLI |

---

## まとめ: Ruby Layer の設計思想

- **境界の明確さ**: Ruby = パース + I/O + ユーザーインターフェース、Rust = AST処理 + コード生成
- **Prism活用**: Rubyの公式パーサーをRuby側で呼び、JSONでRustに橋渡し
- **Gemエコシステム**: Thor, diffy, parallel, ruby_lsp など成熟したGemを活用
- **実用性重視**: パフォーマンスが必要な所だけRust、それ以外はRubyの生産性を活かす
