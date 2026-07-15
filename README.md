# ComfyUI CLI workspace

mise で Python と `comfy-cli` を管理し、ComfyUI をローカルで実行するためのワークスペースです。

## 初期セットアップ

```bash
mise trust
mise install
mise run install
```

`mise run install` は `.venv/` に `comfy-cli` をインストールし、`workspace/` に Apple Silicon 向けの ComfyUI 安定版と ComfyUI-Manager を構築します。

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

## 環境確認

```bash
mise run doctor
mise tasks
```

## ディレクトリ

- `.venv/`: `comfy-cli` 用 Python 仮想環境
- `workspace/`: ComfyUI 本体、専用 Python 環境、カスタムノード、モデル、生成物

これらはサイズが大きく、マシン固有の内容を含むため Git 管理対象外です。
