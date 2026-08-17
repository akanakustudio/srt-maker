# SRT MAKER — MV字幕作成ツール

音源とSRT字幕から、歌詞テロップ入りのミュージックビデオをブラウザ上でプレビュー・書き出しできるツールです。ビルド不要の単一HTMLファイル（[src/index.html](src/index.html)）として実装されており、npmや外部サーバーへの依存はありません。

## 構成

- [src/index.html](src/index.html) — アプリ本体（HTML/CSS/JSを1ファイルに同梱、three.js等のライブラリも埋め込み済み）
- [doc/feature-roadmap.md](doc/feature-roadmap.md) — 機能追加ロードマップ（新機能は実装前にここへ記載する運用）
- [doc/video-export-spec.md](doc/video-export-spec.md) — 動画出力（プレビュー描画とcanvas書き出し）の仕様書
- [.github/workflows/static.yml](.github/workflows/static.yml) — GitHub Pagesへの自動デプロイ設定

ビルドツール・パッケージマネージャは使用していません。`src/index.html` を編集し、そのままブラウザで開けば動作します。

## デバッグ

ビルド手順がないため、以下のいずれかの方法でそのまま動作確認できます。

### 1. ファイルを直接開く

`src/index.html` をブラウザで直接開くだけで動作します（ES Modulesを使用していないため `file://` でも動作します）。

```bash
open src/index.html
```

### 2. ローカルサーバーで開く（推奨）

音声/動画ファイルの読み込みやワーカー系の挙動をより実環境に近い形で確認したい場合は、簡易HTTPサーバー経由で開くことを推奨します。

```bash
cd src
python3 -m http.server 8000
# http://localhost:8000 をブラウザで開く
```

### デバッグの基本方針

- ロジックはすべて `src/index.html` 内の `<script>` にあるため、ブラウザの開発者ツール（コンソール・Sourcesパネル）でそのままデバッグできます。
- プレビュー描画（CSS）と動画書き出し（`<canvas>` + `MediaRecorder`）は別経路の実装です。書き出し結果とプレビューの見た目がずれる場合は [doc/video-export-spec.md](doc/video-export-spec.md) を参照してください。
- UI操作の自動検証にはPlaywrightでの手動シナリオ実行を用いています（リポジトリにテストコードとしては同梱していません）。新機能の検証手順・不具合の原因調査ログは [doc/feature-roadmap.md](doc/feature-roadmap.md) の各項目に記録されています。

## デプロイ

`main` ブランチへのpushをトリガーに、GitHub Actions（[.github/workflows/static.yml](.github/workflows/static.yml)）が自動的にGitHub Pagesへデプロイします。

- ワークフロー: `Deploy static content to Pages`
- 公開ディレクトリ: `./src`
- デプロイ方式: `peaceiris/actions-gh-pages@v3` が `src` の内容を `gh-pages` ブランチへ直接pushし、GitHub Pagesはその `gh-pages` ブランチから配信します
- 手動実行: GitHub Actionsの「Actions」タブから `workflow_dispatch` で手動トリガーも可能です

デプロイに際して追加のビルドステップは存在しません。`src/index.html` の内容がそのまま公開されます。

## 新機能を追加する場合

新機能を実装する前に、まず [doc/feature-roadmap.md](doc/feature-roadmap.md) に項目を追加し、優先度・規模感・依存関係を整理してから着手する運用になっています。詳細は同ファイル冒頭の「運用ルール」を参照してください。
