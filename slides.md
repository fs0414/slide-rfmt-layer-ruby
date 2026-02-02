---
theme: default
background: false
class: text-center
highlighter: shiki
lineNumbers: true
shikiConfig:
  theme: 'nord'
drawings:
  persist: false
transition: slide-left
title: rfmt Ruby Layer
mdc: true
fonts:
  sans: 'Roboto'
  serif: 'Roboto Slab'
  mono: 'Fira Code'
info: |
  rfmt Ruby Layer — Rubyの高速コードフォーマッタ rfmt のRuby層の設計と実装について
seoMeta:
  ogTitle: rfmt Ruby Layer
  ogDescription: Rubyのコードフォーマッタ rfmt のRuby layerの設計と実装について
  ogImage: https://fs0414.github.io/slide-rfmt-layer-ruby/og.png
  ogUrl: https://fs0414.github.io/slide-rfmt-layer-ruby/
  twitterCard: summary_large_image
  twitterTitle: rfmt Ruby Layer
  twitterDescription: Rubyのコードフォーマッタ rfmt のRuby layerの設計と実装について
  twitterImage: https://fs0414.github.io/slide-rfmt-layer-ruby/og.png
---

<CoverSlide
  title="rfmt Ruby Layer"
  author="fujitani sora"
/>

---

<div style="padding: 0 8%">

## <span style="color: oklch(0.7 0.15 215)">fujitani sora</span>

<div class="grid grid-cols-[1fr_1fr] items-start gap-8">
  <div>
    <div class="flex flex-col gap-3 text-lg font-semibold">
      <div class="flex items-center gap-2">
        <carbon-person class="text-lg" />
        <span>2001（24）</span>
      </div>
      <div class="flex items-center gap-2">
        <carbon-building class="text-lg" />
        <span>toridori inc engineer</span>
      </div>
      <div class="flex items-center gap-2">
        <carbon-logo-x class="text-lg" />
        <a href="https://x.com/_fs0414">@_fs0414</a>
      </div>
      <div class="flex items-center gap-2">
        <carbon-logo-github class="text-lg" />
        <a href="https://github.com/fs0414">github.com/fs0414</a>
      </div>
      <div class="flex items-center gap-2">
        <carbon-globe class="text-lg" />
        <a href="https://sorafujitani.me/">sorafujitani.me</a>
      </div>
    </div>
  </div>
  <div class="flex justify-center" style="margin-top: -1.5rem">
    <CenteredImage
      src="https://raw.githubusercontent.com/fs0414/imgs/main/fs0414_dot_image.png"
      alt="プロフィール画像"
      width="280px"
    />
  </div>
</div>

</div>

---

# TOC

<div style="display: flex; justify-content: center; margin-top: 2rem;">
<div style="display: flex; flex-direction: column; gap: 1.4rem; text-align: left;">
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.7 0.1 215); font-weight: 600;">PrismBridge</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.7 0.1 215); font-weight: 600;">Foreign Function Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.7 0.1 215); font-weight: 600;">Command Line Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.7 0.1 215); font-weight: 600;">Configuration & Cache</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.7 0.1 215); font-weight: 600;">Native Extension & Editor Integration</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.7 0.1 215); font-weight: 600;">E2E test</p>
</div>
</div>

---

# rfmt

- 高速なRuby code formatter
- RubyGems Rust Extentionを使用し、Code ModuleをRustで実装
  - https://bundler.io/blog/2023/01/31/rust-gem-skeleton.html

- GitHub: https://github.com/fs0414/rfmt
  - いまStar数55くらい
- RubyGems: https://rubygems.org/gems/rfmt
  - 資料書いてる時で9945 installs
- Zenn: https://zenn.dev/soramarjr/articles/0b2464bc09b643

---

# Architecture

<TwoColumnLayout>
  <template #left>
    <div style="padding-top: 10%;">
      <ul>
        <li><strong>Ruby Layer</strong>
          <ul>
            <li>CLI, LSP Integration</li>
            <li>Config</li>
            <li>Cache</li>
            <li>PrismBridge Module によるAST Parse</li>
          </ul>
        </li>
        <li><strong>FFI Boundary</strong>
          <ul>
            <li>Magnus + rb_sys で Ruby ↔ Rust をJSON経由で接続</li>
          </ul>
        </li>
        <li><strong>Rust Layer</strong>
          <ul>
            <li>Emitterによる具体的なformat処理</li>
          </ul>
        </li>
      </ul>
      <br/>
      <h5>今日はこのRuby layerの全体的な話</h5>
    </div>
  </template>
  <template #right>
    <img src="/architecture.png" alt="rfmt アーキテクチャ図" style="width: 100%;" />
  </template>
</TwoColumnLayout>

---

# format実行フロー

<div class="text-2xl leading-loose">

1. Ruby source code を受け取る
2. Prism でパースして **AST** を取得
3. PrismBridge で AST を走査し、Rust が処理できる **JSON** に変換
4. JSON を FFI 経由で Rust に渡す
5. Rust から返ってきた結果をファイル書き込み or 標準出力

</div>

---

# TOC

<div style="display: flex; justify-content: center; margin-top: 2rem;">
<div style="display: flex; flex-direction: column; gap: 1.4rem; text-align: left;">
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.9 0.12 215); font-weight: 700;">PrismBridge</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Foreign Function Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Command Line Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Configuration & Cache</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Native Extension & Editor Integration</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">E2E test</p>
</div>
</div>

---

# PrismBridge

`lib/rfmt/prism_bridge.rb` / `lib/rfmt/prism_node_extractor.rb`

- source codeを入力にPrism parserを呼び出し、ASTを受け取る
- 各ノードからRfmt内部が利用したいドメインモデルに合わせたメタデータを抽出
    - クラス名、メソッド名、パラメータ数、etc.
    - ロケーション情報 (行番号、カラム、オフセット)
- コメント情報の収集とシリアライズ

**PrismBridgeの中間層を入れることで、Prismとの依存を吸収し、Parserを差し替えられる設計**

---

# PrismとのIntegration

- PrismはRuby標準のパーサーGem
- Ruby側でPrismを呼ぶのが最も自然
- RustからPrismを呼ぶにはRuby VMを経由するFFIが必要で複雑になる
- Ruby側でPrism ASTを走査し、Rust側が処理しやすい形式への事前処理を行う役割

`lib/rfmt/prism_bridge.rb`

```ruby
def self.parse(source)
  result = Prism.parse(source)
  handle_parse_errors(result) if result.failure?
  serialize_ast_with_comments(result)
end
```

---

# AST変換 — 具体例

<TwoColumnLayout>
  <template #left>

このRubyコードを入力すると…

```ruby
def greet(name)
  puts "Hello, #{name}"
end
```

  </template>
  <template #right>

PrismBridge がこのような JSON に変換する

```json
{
  "node_type": "def_node",
  "metadata": {
    "name": "greet",
    "parameters_count": "1"
  },
  "children": [
    { "node_type": "required_parameter_node" },
    { "node_type": "statements_node" }
  ]
}
```

  </template>
</TwoColumnLayout>

Rust 側はこの JSON を受け取り、AST として再構築してフォーマットを行う

---

# TOC

<div style="display: flex; justify-content: center; margin-top: 2rem;">
<div style="display: flex; flex-direction: column; gap: 1.4rem; text-align: left;">
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">PrismBridge</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.9 0.12 215); font-weight: 700;">Foreign Function Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Command Line Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Configuration & Cache</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Native Extension & Editor Integration</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">E2E test</p>
</div>
</div>

---

# Magnus — Ruby bindings for Rust

- Rust で Ruby の拡張 gem を書くためのライブラリ
- Rust の関数を Ruby のメソッドとして公開できる
- Ruby ↔ Rust 間の型変換を自動で処理
- 引数のバリデーションやエラーハンドリングも Ruby の慣習に沿って動作
- rfmt では Magnus 経由で Rust のフォーマッタを Ruby から呼び出している

---

# Ruby-Rust 間のFFI境界

<TwoColumnLayout>
  <template #left>

<br><br><br>

```ruby
# Ruby側 (lib/rfmt.rb)
def self.format(source)
  prism_json = PrismBridge.parse(source)
  format_code(source, prism_json)
end
```

  </template>
  <template #right>

```rust
// Rust側 (ext/rfmt/src/lib.rs)
#[magnus::init]
fn init(ruby: &Ruby) -> Result<(), Error> {
    let module =
      ruby.define_module("Rfmt")?;

    module.define_singleton_method(
      "format_code",
      function!(format_ruby_code, 2))?;
    module.define_singleton_method(
      "parse_to_json",
      function!(parse_to_json, 1))?;
    module.define_singleton_method(
      "rust_version",
      function!(rust_version, 0))?;
    Ok(())
}
```

  </template>
</TwoColumnLayout>

- Magnus crate による Ruby-Rust FFI
- Ruby から呼べるフォーマット・パース・バージョン取得の3つの関数を公開
- Ruby と Rust 間のデータ型変換は Magnus が自動で処理

---

# TOC

<div style="display: flex; justify-content: center; margin-top: 2rem;">
<div style="display: flex; flex-direction: column; gap: 1.4rem; text-align: left;">
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">PrismBridge</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Foreign Function Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.9 0.12 215); font-weight: 700;">Command Line Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Configuration & Cache</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Native Extension & Editor Integration</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">E2E test</p>
</div>
</div>

---

# CLI の概要

`lib/rfmt/cli.rb`

- Thor ベースのCLI — Thor gem で宣言的なコマンド定義
- コマンド: format / check / version / config / cache / init
- 並列処理の自動判定 (ファイル数・サイズに基づくヒューリスティクス)
  - 余談で、10fileほどであれば並列化しない方が速い
- diff表示: diffy / diff-lcs gem (unified / side_by_side / color)
- progress表示

---

# CLI の処理フロー

1. **設定の読み込み** — プロジェクトごとのルールを尊重するために、YAMLから設定を探索
2. **対象ファイルの決定** — 全ファイルを毎回処理するのは遅いので、キャッシュで変更があったものだけに絞る
3. **並列/逐次の自動判定** — ファイル数・サイズから最速の戦略を選ぶ。少量なら逐次の方が速い
4. **フォーマット実行** — Prism で AST に変換し、Rust に渡して整形
5. **結果の出力** — モードに応じてファイル書き込み / diff表示 / 違反チェックを切り替え
6. **キャッシュ更新** — 次回の実行を速くするために、処理済みファイルの情報を保存

---

# TOC

<div style="display: flex; justify-content: center; margin-top: 2rem;">
<div style="display: flex; flex-direction: column; gap: 1.4rem; text-align: left;">
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">PrismBridge</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Foreign Function Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Command Line Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.9 0.12 215); font-weight: 700;">Configuration & Cache</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Native Extension & Editor Integration</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">E2E test</p>
</div>
</div>

---

# Configuration & Cache

<TwoColumnLayout>
  <template #left>
    <p><strong>Configuration - 設定管理</strong></p>
    <p><code>lib/rfmt/configuration.rb</code></p>
    <ul>
      <li>YAML設定ファイルの探索・読込・バリデーション</li>
      <li>ファイルglobパターンによるinclude/exclude</li>
      <li>デフォルト設定とのマージ</li>
    </ul>
    <p>Ruby の得意分野: YAML パース、Dir.glob によるファイル探索</p>
  </template>
  <template #right>
    <p><strong>Cache - キャッシュシステム</strong></p>
    <p><code>lib/rfmt/cache.rb</code></p>
    <ul>
      <li>mtime (ファイル更新日時) ベースの変更検知</li>
      <li>~/.cache/rfmt/cache.json にJSONで永続化</li>
      <li>clear / prune / stats 操作</li>
    </ul>
    <p>低頻度・軽量処理のためRubyで十分な速度</p>
  </template>
</TwoColumnLayout>

---

# TOC

<div style="display: flex; justify-content: center; margin-top: 2rem;">
<div style="display: flex; flex-direction: column; gap: 1.4rem; text-align: left;">
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">PrismBridge</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Foreign Function Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Command Line Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Configuration & Cache</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.9 0.12 215); font-weight: 700;">Native Extension & Editor Integration</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">E2E test</p>
</div>
</div>

---

# Gem配布 & ネイティブ拡張ロード

`lib/rfmt/native_extension_loader.rb`

- rfmt.gemspec + ext/rfmt/extconf.rb による標準的なnative extension gem構造
- gem install rfmt で Rust 拡張含めてビルド&インストール
- rb_sys / magnus による Ruby-Rust FFI の標準的なパターン
- Ruby 3.0〜3.3+ のバージョン別パス対応

**ビルド**: extconf.rb → Rust の Makefile を生成 → Cargo でリリースビルド → 共有ライブラリ (.bundle / .so) を出力

**ロード**: Ruby のバージョンに応じたパスからshared libraryを自動検出して読み込み

---

# Ruby LSP Integration

`lib/ruby_lsp/rfmt/addon.rb` / `lib/ruby_lsp/rfmt/formatter_runner.rb`

- Ruby LSP の Addon として登録
- format-on-save で rfmt を呼び出し
- FormatterRunner インターフェースに準拠

<br>

Ruby LSP の Addon は Ruby で書く必要があるため、Ruby Layer に実装

---

# TOC

<div style="display: flex; justify-content: center; margin-top: 2rem;">
<div style="display: flex; flex-direction: column; gap: 1.4rem; text-align: left;">
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">PrismBridge</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Foreign Function Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Command Line Interface</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Configuration & Cache</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.35 0.03 270);">Native Extension & Editor Integration</p>
<p style="font-size: 1.8rem; margin: 0; color: oklch(0.9 0.12 215); font-weight: 700;">E2E test</p>
</div>
</div>

---

# テスト構成

- **フォーマットテスト** — Rubyコードを入力し、期待する整形結果と比較
  - 条件分岐、ループ、ブロック、rescue、lambda、パターンマッチなど構文ごとに網羅
- **設定テスト** — YAML設定の探索・読込・親ディレクトリからの継承を検証
- **CLIテスト** — コマンド実行の正常系・異常系
- **LSP連携テスト** — Ruby LSP Addon の登録と format-on-save の動作
- **ネイティブ拡張テスト** — Rubyバージョン別のロードパス切り替え

Ruby テストと Rust テスト (`cargo test`) を `rake dev:test_all` で一括実行

---

# テストの具体例

```ruby
it 'formats if with elsif and else' do
  source = "if x > 0\nputs \"positive\"\nelsif x < 0\n..."

  result = Rfmt.format(source)

  # フォーマット後、正しくインデントされていることを検証
  expect(result).to eq(expected)
end
```

入力のRubyコードをフォーマットし、期待するインデントや構造と一致するかを検証

---

# まとめ

- **Rust Extension**: Ruby = parse + I/O + 外部とのIntegration、Rust = 具体的なformattiongや計算負荷の高い処理
- **Prism**: Rubyの標準parserをRuby側で呼び、JSONでRustに渡す
- **Gemエコシステム**: Thor, diffy, parallel, ruby_lsp などのGemを活用

---

<h1 class="text-white text-4xl font-bold mb-8">see you later 👋</h1>

<div class="grid grid-cols-3 gap-6 text-left mx-8">
  <div class="p-4 rounded-lg" style="background: oklch(0.25 0.03 260); border: 1px solid oklch(0.4 0.06 260)">
    <p class="font-bold mb-2">Rails Girls Tokyo #18</p>
    <p class="font-bold">コーチで参加するよ</p>
    <a href="https://railsgirls.com/tokyo-2026-02-13.html">https://railsgirls.com/tokyo-2026-02-13</a>
  </div>
  <div class="p-4 rounded-lg" style="background: oklch(0.25 0.03 260); border: 1px solid oklch(0.4 0.06 260)">
    <p class="font-bold mb-2">PHPerKaigi 2026</p>
    <p class="font-bold">Day1に登壇するよ</p>
    <img src="/phperkaigi.png" alt="PHPerKaigi 2026" class="rounded mb-2" style="width: 100%;" />
  </div>
  <div class="p-4 rounded-lg" style="background: oklch(0.25 0.03 260); border: 1px solid oklch(0.4 0.06 260)">
    <p class="font-bold mb-2">rfmt</p>
    <p class="font-bold">GitHub Starしてね</p>
    <a href="https://github.com/fs0414/rfmt">https://github.com/fs0414/rfmt</a>
  </div>
 </div>
