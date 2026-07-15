# ComfyUI CLI workspace

mise で Python と `comfy-cli` を管理し、ComfyUI をローカルで実行するためのワークスペースです。

## 初期セットアップ

```bash
mise trust
mise install
mise run setup-comfyui
```

`mise run setup-comfyui` は `.venv/` に `comfy-cli` をインストールし、`workspace/` に Apple Silicon 向けの ComfyUI 安定版と ComfyUI-Manager を構築します。

## 起動と停止

フォアグラウンドで起動する場合：

```bash
mise run launch
```

バックグラウンドで起動する場合：

```bash
mise run start
mise run stop
```

起動後は通常、ブラウザで <http://127.0.0.1:8188> を開きます。

初回の画像生成には checkpoint モデルが必要です。Claude Code に「画像を生成して」と依頼して自動導入するか、[公式ガイド](https://docs.comfy.org/ja/get_started/first_generation)の手順で `workspace/models/checkpoints/` にモデルを配置してください。

## 環境確認

```bash
mise run doctor
mise tasks
```

## ディレクトリ

- `.venv/`: `comfy-cli` と ComfyUI 実行用 (torch など) の Python 仮想環境
- `workspace/`: ComfyUI 本体、カスタムノード、モデル、生成物
- `workflows/`: ワークフロー JSON (API 形式) の置き場。`text-to-image.json` は SD 1.5 の最小構成
- `.claude/`: Claude Code のプロジェクト設定 (`settings.json`) とスキル (`skills/`)
- `override.txt`: `--fast-deps` が生成する依存関係の上書き定義。ComfyUI 本体の依存関係をカスタムノードなどの依存関係より優先するために使われる
- `requirements.compiled`: `--fast-deps` が `uv pip compile` で解決した、バージョン固定済みの Python 依存関係一覧

`.venv/` と `workspace/` はサイズが大きく、マシン固有の内容を含みます。`override.txt` と `requirements.compiled` はインストール時に再生成され、生成元の絶対パスを含みます。そのため、いずれも Git 管理対象外です。

## Claude Code 連携

このワークスペースは Claude Code で運用・活用できるよう整備されています。

- `AGENTS.md`: AI エージェント向け運用ガイド (英語)。`CLAUDE.md` は `AGENTS.md` への symlink
- `.claude/settings.json`: ComfyUI の起動・停止、ワークフロー実行、モデルダウンロード、カスタムノード追加、ローカル API アクセス、docs.comfy.org の参照を確認プロンプトなしで実行するための permissions 設定。初回起動時にこのフォルダを信頼するかの確認が1回必要
- `.claude/skills/`: プロジェクトスキル4種
  - `comfyui-workflow`: ワークフロー JSON (API 形式) の作成・編集
  - `comfyui-run`: 生成の実行と結果画像の取得
  - `comfyui-assets`: モデルのダウンロードとカスタムノード管理
  - `comfyui-docs`: docs.comfy.org/ja の参照導線 (URL マップ)

例えば「幻想的な風景の画像を生成して」と依頼すると、モデル導入 → サーバー起動 → ワークフロー作成 → 実行 → `workspace/output/` の生成結果報告までを Claude Code が自走します。破壊的な操作 (モデル削除、カスタムノード削除、`workspace/` の削除など) は引き続き確認プロンプトの対象です。
