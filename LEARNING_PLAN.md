# ComfyUI 学習プラン

このプランは、このリポジトリ(mise管理のComfyUIワークスペース)を実際に動かしながらComfyUIを学ぶための段階的ガイドです。
各フェーズに「目標」「やること」「確認ポイント」を用意しています。
Claude Codeに作業を依頼しながら進めても、自分の手でUI操作しながら理解を深めても構いません。

## 進め方のTips

各フェーズに入る前に `mise run doctor` と `curl -s http://127.0.0.1:8188/system_stats` で環境が生きていることを確認してください。
`workspace/` 配下は読み取り専用のground truthです。変更が必要な場合は必ずスキル経由で行ってください。
生成結果は `workspace/output/` に溜まっていきます。定期的に見返すと変化を振り返りやすくなります。
わからない挙動に出会ったら、まず `comfyui-docs` スキルで公式ドキュメント(docs.comfy.org/ja)を確認してください。

## Phase 0: 環境構築とヘルスチェック(目安15分)

目標: 環境が正しく動くことを確認する。

- 初回のみ: `mise trust` → `mise install` → `mise run setup-comfyui`(既にセットアップ済みならスキップ)
- `mise run doctor` でPython/comfy-cliのバージョンとワークスペースパスを確認
- `mise run start` でバックグラウンド起動し、`curl -s http://127.0.0.1:8188/system_stats` で疎通確認
- ブラウザで <http://127.0.0.1:8188> を開き、デフォルトのノードグラフ画面を一目見る

確認ポイント: `system_stats` がJSONを返す。ブラウザでノードグラフが表示される。

## Phase 1: Hello World — 最初の1枚を生成する(目安30分)

目標: 最小構成のtext-to-imageワークフローを動かして1枚生成する。

- checkpointモデルが未導入なら、Claude Codeに「画像を生成して」と依頼すると `comfyui-assets` スキル経由で自動導入される
- `workflows/text-to-image.json`(SD 1.5の最小構成)を、Claude Codeに実行依頼するか、Web UIにドラッグ&ドロップして「Queue Prompt」を押す
- `workspace/output/` に生成されたPNGを確認する

確認ポイント: PNGファイルが1枚出力される。生成にかかる時間感覚をつかむ。

## Phase 2: ノードグラフの基本概念を理解する(目安45分)

目標: `text-to-image.json` を構成する各ノードの役割を自分の言葉で説明できるようになる。

- `comfyui-docs` スキルでdocs.comfy.org/jaの基礎ページを参照する
- 主要ノードを1つずつ確認する: CheckpointLoaderSimple(モデル読み込み) → CLIPTextEncode(プロンプトのエンコード、positive/negativeの2系統) → EmptyLatentImage(キャンバスサイズ) → KSampler(拡散過程のサンプリング) → VAEDecode(潜在空間から画像へのデコード) → SaveImage(保存)
- `workflows/text-to-image.json` をRead toolで開き、JSON構造とUI上のノードグラフを対応させる

確認ポイント: 「なぜpositiveとnegativeの2つのCLIPTextEncodeがあるのか」「KSamplerのseed/steps/cfgが何をしているか」を説明できる。

## Phase 3: パラメータを変えて実験する(目安30分)

目標: steps、cfg、sampler_name、seed、denoiseの効果を体感する。

- `comfyui-workflow` スキルで `text-to-image.json` を複製・編集し、steps(10 vs 30)、cfg(3 vs 12)、seed固定/変更をそれぞれ試す
- 同じseedで結果が再現されることを確認する(決定論的生成の理解)

確認ポイント: パラメータ変更が生成結果にどう影響するか、before/afterで比較できる。

## Phase 4: img2img(目安30分)

目標: 既存画像を元にした生成を理解する。

- `workspace/input/` に元画像を配置する(またはAPI経由でアップロード)
- `comfyui-workflow` スキルでimg2imgワークフローを新規作成する(LoadImageノード + VAEEncode + denoise調整)
- denoiseの値による元画像への忠実度の変化を確認する

確認ポイント: denoise=1.0とdenoise=0.3の違いを説明できる。

## Phase 5: LoRAとモデルのカスタマイズ(目安45分)

目標: LoRAを使ったスタイル調整を理解する。

- `comfyui-assets` スキルでLoRAモデルを1つ導入する
- LoraLoaderノードをワークフローに追加し、strength_model/strength_clipを調整する

確認ポイント: LoRA適用前後で画風がどう変わるか比較できる。

## Phase 6: Z-Image-Turboワークフローを動かす(目安20分)

目標: このリポジトリに既にあるより高度なワークフロー(`z-image-turbo.json`)を理解・実行する。

- `workflows/z-image-turbo.json` をRead toolで開き、`text-to-image.json` との差分を確認する(高速サンプラー、ステップ数の違いなど)
- `comfyui-run` スキルで実行し、生成速度と品質のトレードオフを体感する

確認ポイント: turbo系モデル特有の設定(低ステップ数など)の理由を説明できる。

## Phase 7: アップスケールと後処理(目安30分)

目標: 生成画像の高解像度化パイプラインを理解する。

- LatentUpscaleまたはESRGAN系アップスケーラーノードを追加する
- Hires fix的な2段階生成(低解像度生成→アップスケール→denoise低めで再生成)を試す

確認ポイント: 直接高解像度で生成する場合との品質差を比較できる。

## Phase 8: カスタムノードとComfyUI-Manager(目安30分)

目標: エコシステムの広がりを理解する。

- `comfyui-assets` スキルでカスタムノードを1つ導入する(例: ControlNet系)
- ComfyUI-Managerの役割(このリポジトリでは `.venv/` 内のpipパッケージとして統合されている点)を理解する

確認ポイント: node install/updateの操作フローを一通り体験する。

## Phase 9: Claude Codeとの連携を使いこなす(目安20分)

目標: 4つのスキルの使い分けを体で覚える。

- `comfyui-assets`: モデル・カスタムノード管理
- `comfyui-workflow`: ワークフローJSON作成・編集
- `comfyui-run`: 実行と結果取得
- `comfyui-docs`: 公式ドキュメント参照
- 「幻想的な風景の画像を生成して」のような自然言語の依頼から、Claude Codeがどのスキルをどの順で呼ぶかを観察する

確認ポイント: 自分では日本語の指示だけを与え、ワークフローを1から作らせて実行できる。

## Phase 10: 応用・発展(継続的)

- ControlNet(構図・ポーズ指定)
- バッチ生成とseed探索
- 複数モデルの組み合わせ(img2img + LoRA + upscale)
- `workflows/` への新規ワークフローのコミット(このリポジトリのgit管理対象)
