# よっこらしょ対策_絵コンテラフ画 Implementation Plan

**Goal:** 「01_椅子から立つとき「よっこらしょ」対策」動画の撮影前検討用に、台本の実演3ステップ（①基本姿勢／②お辞儀→立ち上がり／③座位膝伸ばし）を ControlNet(OpenPose) で構図固定したラフスケッチ画像3枚を生成する。

**Architecture:** 参照動画2本から各ポーズに最適なフレームを目視選定 → OpenPose骨格検出でControlNet入力化 → SD1.5 + OpenPose ControlNet + プロンプトでラフスケッチ生成、という一方向パイプライン。テスト駆動開発は適用せず、各タスクの「検証」はコマンド実行結果・生成画像の目視確認に置き換える（本リポジトリに自動テストフレームワークは存在しないため）。

**Tech Stack:** ComfyUI 0.28.0 (ローカル, mise管理) / comfy-cli 1.12.0 / Python 3.13.13 (.venv, cv2 5.0.0) / custom_nodes: comfyui_controlnet_aux (OpenposePreprocessor) / checkpoint: v1-5-pruned-emaonly-fp16.safetensors (SD1.5)

**Design Document:** 本セッションの `/brainstorm` で確定した要件（チャット内サマリー、`tmp/台本とプログラム/筋トレ_動画企画案50件/01_椅子から立つとき「よっこらしょ」対策.md` が元台本）

**Recommended Execution:** Sequential（8タスク・単一セッションで完結する線形パイプラインであり、各所で自分の目視判断を挟む必要があるため、`/loop` によるHITLスケジューリングより直接 `executing-plans` で通しで実行する方が適合する）

**Assumptions（要ユーザー確認）:**

- OpenPose ControlNetモデルは `comfyanonymous/ControlNet-v1-1_fp16_safetensors` の `control_v11p_sd15_openpose_fp16.safetensors`（既存の canny モデルと同一シリーズ、HTTP 302で存在確認済み、実ファイル内容は未検証）
- 生成解像度は動画のアスペクト比(16:9, 1920x1080)をSD1.5の得意面積(512×512相当)に縮小した `704x400`（8の倍数）
- ワークフローファイルは `workflows/storyboard-yokkorasho/` フォルダにまとめ、`pose1-basic.json` / `pose2-standup.json` / `pose3-kneeext.json` として保存
- 参照フレーム画像: `workspace/input/yokkorasho_pose{1,2,3}_{basic,standup,kneeext}.png`

---

### Task 1: OpenPose ControlNetモデルの確保

**Files:**

- Create: `workspace/models/controlnet/control_v11p_sd15_openpose_fp16.safetensors`（`comfyui-assets` スキル経由、gitignore対象）

**Step 1: ダウンロード実行**

```bash
.venv/bin/comfy --workspace workspace model download \
  --url https://huggingface.co/comfyanonymous/ControlNet-v1-1_fp16_safetensors/resolve/main/control_v11p_sd15_openpose_fp16.safetensors \
  --relative-path models/controlnet --filename control_v11p_sd15_openpose_fp16.safetensors
```

**Step 2: 反映確認**

```bash
ls workspace/models/controlnet/
curl -s "http://127.0.0.1:8188/object_info/ControlNetLoader" | python3 -c "import json,sys; d=json.load(sys.stdin); print('control_v11p_sd15_openpose_fp16.safetensors' in d['ControlNetLoader']['input']['required']['control_net_name'][0])"
```

Expected: `ls` にファイルが表示され、`object_info` 側の判定が `True`。`False` の場合は `mise run stop && mise run start` でサーバーを再起動してから再確認する。

**Step 3: Commit**

```bash
git status  # workspace/ 配下はgitignore対象のためコミット不要（確認のみ）
```

---

### Task 2: 参照動画から候補フレームを抽出

**Files:**

- Create（scratchpad、リポジトリ非管理）: `<scratchpad>/frame_candidates/extract_frames.py`
- Output（scratchpad）: `<scratchpad>/frame_candidates/{video_stem}/frame_{ms:06d}.png`

**Step 1: 抽出スクリプト作成**

```python
# <scratchpad>/frame_candidates/extract_frames.py
import cv2
import pathlib
import sys

VIDEOS = [
    "tmp/台本とプログラム/筋トレ_動画企画案50件/01_椅子から立つとき「よっこらしょ」対策_動画/動画リンク_IMG_1744.MOV",
    "tmp/台本とプログラム/筋トレ_動画企画案50件/01_椅子から立つとき「よっこらしょ」対策_動画/追加動画リンク_IMG_3718.MOV",
]
OUT_DIR = pathlib.Path(sys.argv[1])
INTERVAL_MS = int(sys.argv[2]) if len(sys.argv) > 2 else 500

for video_path in VIDEOS:
    cap = cv2.VideoCapture(video_path)
    fps = cap.get(cv2.CAP_PROP_FPS)
    frame_count = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
    height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
    duration_ms = frame_count / fps * 1000
    print(f"{video_path}: {width}x{height}, {fps:.2f}fps, {frame_count}frames, {duration_ms:.0f}ms")

    stem = pathlib.Path(video_path).stem
    out_sub = OUT_DIR / stem
    out_sub.mkdir(parents=True, exist_ok=True)

    t_ms = 0
    while t_ms < duration_ms:
        cap.set(cv2.CAP_PROP_POS_MSEC, t_ms)
        ok, frame = cap.read()
        if not ok:
            break
        cv2.imwrite(str(out_sub / f"frame_{t_ms:06d}.png"), frame)
        t_ms += INTERVAL_MS
    cap.release()
```

**Step 2: 実行**

Run: `.venv/bin/python3 <scratchpad>/frame_candidates/extract_frames.py <scratchpad>/frame_candidates 500`
Expected: 標準出力に2本の動画メタデータ（解像度・fps・フレーム数・長さ）が表示され、`<scratchpad>/frame_candidates/{動画リンク_IMG_1744,追加動画リンク_IMG_3718}/` に `frame_*.png` が並ぶ。

**代替経路:** Task3で候補プールに①②③いずれかに適したフレームが見当たらない場合（特に②「しなり」のピークは一瞬で0.5秒刻みでは捉え損ねる可能性がある）、該当タイムスタンプ付近を `INTERVAL_MS=100` で再実行し候補を細かくする。

**Step 3: Commit**

コミット対象なし（scratchpad配下は非管理）。

---

### Task 3: ①②③各ポーズに最適なフレームを選定・保存

**Files:**

- Create: `workspace/input/yokkorasho_pose1_basic.png`
- Create: `workspace/input/yokkorasho_pose2_standup.png`
- Create: `workspace/input/yokkorasho_pose3_kneeext.png`

**Step 1: 目視選定**

Task2の候補フレーム群を Read ツールで確認し、台本の記述と照合して各ポーズに最も近いタイムスタンプを選ぶ:

- ①基本姿勢: 座位で骨盤前傾・膝外側・背筋が伸びている瞬間
- ②お辞儀→立ち上がり: 上体を前に倒し重心が最も前方へ移動している瞬間（「しなり」のピーク）
- ③座位膝伸ばし: 片脚が水平近くまで伸びている瞬間

**Step 2: 選定フレームを保存**

該当タイムスタンプの候補PNGを `workspace/input/yokkorasho_pose{N}_{name}.png` としてコピーする。

```bash
cp "<scratchpad>/frame_candidates/{選定した動画}/frame_{選定ms}.png" workspace/input/yokkorasho_pose1_basic.png
cp "<scratchpad>/frame_candidates/{選定した動画}/frame_{選定ms}.png" workspace/input/yokkorasho_pose2_standup.png
cp "<scratchpad>/frame_candidates/{選定した動画}/frame_{選定ms}.png" workspace/input/yokkorasho_pose3_kneeext.png
```

**Step 3: 確認**

Run: `ls workspace/input/yokkorasho_pose*.png`
Expected: 3ファイルが存在。Read ツールで各画像を開き、台本の説明と矛盾しないことを確認する。

---

### Task 4: OpenPose骨格検出のプレビュー確認

**Files:**

- Create（一時、非git管理）: `<scratchpad>/openpose_preview_pose{1,2,3}.json`（`comfyui-workflow` スキル参照の使い捨てワークフロー）

**Step 1: 検出用ワークフロー作成**

```json
{
  "1": { "class_type": "LoadImage", "inputs": { "image": "yokkorasho_pose1_basic.png" } },
  "2": { "class_type": "OpenposePreprocessor", "inputs": { "image": ["1", 0], "detect_hand": "disable", "detect_body": "enable", "detect_face": "disable", "resolution": 512 } },
  "3": { "class_type": "SaveImage", "inputs": { "filename_prefix": "openpose_preview_pose1", "images": ["2", 0] } }
}
```

pose2・pose3 用に `image` と `filename_prefix` だけ差し替えた2バリエーションも同様に作成する。

**Step 2: 実行**

Run: `.venv/bin/comfy --workspace workspace run <scratchpad>/openpose_preview_pose1.json`（pose2, pose3も同様に実行）
Expected: `workspace/output/openpose_preview_pose{1,2,3}_*.png` に骨格オーバーレイ画像が出力される。

**Step 3: 検証**

Read ツールで3枚を確認:

- 共通: 肩・腰・膝の主要関節が正しい位置で検出されている
- **②専用の重点確認**: 体幹の前傾角度と重心移動（骨盤〜肩ラインの傾き）が骨格線上で明確に読み取れるか

検出漏れがあれば Task3 に戻り別候補フレームを選定。候補プールに適したフレームが無ければ Task2 に戻り該当タイムスタンプ周辺を `INTERVAL_MS=100` で再抽出する。

---

### Task 5: ①基本姿勢のラフ画生成ワークフロー作成

**Files:**

- Create: `workflows/storyboard-yokkorasho/pose1-basic.json`

**Step 1: ワークフロー作成**

```json
{
  "1": {
    "class_type": "LoadImage",
    "inputs": { "image": "yokkorasho_pose1_basic.png" },
    "_meta": { "title": "Load Image (Pose1 Reference)" }
  },
  "2": {
    "class_type": "OpenposePreprocessor",
    "inputs": { "image": ["1", 0], "detect_hand": "disable", "detect_body": "enable", "detect_face": "disable", "resolution": 512 },
    "_meta": { "title": "OpenPose Preprocessor" }
  },
  "3": {
    "class_type": "CheckpointLoaderSimple",
    "inputs": { "ckpt_name": "v1-5-pruned-emaonly-fp16.safetensors" },
    "_meta": { "title": "Load Checkpoint" }
  },
  "4": {
    "class_type": "CLIPTextEncode",
    "inputs": {
      "text": "rough storyboard sketch, monochrome pencil line art, loose expressive linework, generic featureless mannequin figure sitting upright on a simple chair, feet flat on floor, knees turned slightly outward, back straight, hands on knees, wide front-angle camera framing showing full chair and figure, blank background, production storyboard illustration style",
      "clip": ["3", 1]
    },
    "_meta": { "title": "CLIP Text Encode (Positive)" }
  },
  "5": {
    "class_type": "CLIPTextEncode",
    "inputs": { "text": "photorealistic, color, detailed skin texture, text, watermark, extra limbs, deformed hands, elderly features, realistic face, background clutter", "clip": ["3", 1] },
    "_meta": { "title": "CLIP Text Encode (Negative)" }
  },
  "6": {
    "class_type": "ControlNetLoader",
    "inputs": { "control_net_name": "control_v11p_sd15_openpose_fp16.safetensors" },
    "_meta": { "title": "Load ControlNet Model (OpenPose)" }
  },
  "7": {
    "class_type": "ControlNetApplyAdvanced",
    "inputs": { "positive": ["4", 0], "negative": ["5", 0], "control_net": ["6", 0], "image": ["2", 0], "strength": 1.0, "start_percent": 0.0, "end_percent": 1.0 },
    "_meta": { "title": "Apply ControlNet" }
  },
  "8": {
    "class_type": "EmptyLatentImage",
    "inputs": { "width": 704, "height": 400, "batch_size": 1 },
    "_meta": { "title": "Empty Latent Image" }
  },
  "9": {
    "class_type": "KSampler",
    "inputs": { "seed": 1001, "steps": 20, "cfg": 8, "sampler_name": "euler", "scheduler": "normal", "denoise": 1.0, "model": ["3", 0], "positive": ["7", 0], "negative": ["7", 1], "latent_image": ["8", 0] },
    "_meta": { "title": "KSampler" }
  },
  "10": {
    "class_type": "VAEDecode",
    "inputs": { "samples": ["9", 0], "vae": ["3", 2] },
    "_meta": { "title": "VAE Decode" }
  },
  "11": {
    "class_type": "SaveImage",
    "inputs": { "filename_prefix": "storyboard_yokkorasho_pose1_basic", "images": ["10", 0] },
    "_meta": { "title": "Save Image" }
  }
}
```

**Step 2: 検証**

Run: `.venv/bin/comfy --workspace workspace validate workflows/storyboard-yokkorasho/pose1-basic.json`
Expected: バリデーション成功（envelope の `ok: true`）

**Step 3: Commit**

```bash
git add workflows/storyboard-yokkorasho/pose1-basic.json
git commit -m "feat: よっこらしょ対策①基本姿勢のラフ画ワークフローを追加"
```

---

### Task 6: ②お辞儀→立ち上がりのラフ画生成ワークフロー作成

**Files:**

- Create: `workflows/storyboard-yokkorasho/pose2-standup.json`

Task5と同一構成。差分のみ:

- node "1" `image`: `yokkorasho_pose2_standup.png`
- node "4" positive text: `"rough storyboard sketch, monochrome pencil line art, loose expressive linework, generic featureless mannequin figure captured mid-motion rising from a chair, torso bent forward with a strong spinal curve, head leading forward and down, weight shifting from hips toward the feet, dynamic action lines emphasizing the forward lean and rising motion, chair visible beside the figure, side-angle camera framing showing the full standing-up arc, blank background, production storyboard illustration style"`
- node "9" `seed`: `1002`
- node "11" `filename_prefix`: `storyboard_yokkorasho_pose2_standup`

**Step 2: 検証**

Run: `.venv/bin/comfy --workspace workspace validate workflows/storyboard-yokkorasho/pose2-standup.json`
Expected: バリデーション成功

**Step 3: Commit**

```bash
git add workflows/storyboard-yokkorasho/pose2-standup.json
git commit -m "feat: よっこらしょ対策②立ち上がりしなりのラフ画ワークフローを追加"
```

---

### Task 7: ③座位膝伸ばしのラフ画生成ワークフロー作成

**Files:**

- Create: `workflows/storyboard-yokkorasho/pose3-kneeext.json`

Task5と同一構成。差分のみ:

- node "1" `image`: `yokkorasho_pose3_kneeext.png`
- node "4" positive text: `"rough storyboard sketch, monochrome pencil line art, loose expressive linework, generic featureless mannequin figure sitting on a chair with one leg extended straight forward at knee height, other foot flat on floor, hands resting on chair sides for balance, front three-quarter camera framing showing the chair and extended leg clearly, blank background, production storyboard illustration style"`
- node "9" `seed`: `1003`
- node "11" `filename_prefix`: `storyboard_yokkorasho_pose3_kneeext`

**Step 2: 検証**

Run: `.venv/bin/comfy --workspace workspace validate workflows/storyboard-yokkorasho/pose3-kneeext.json`
Expected: バリデーション成功

**Step 3: Commit**

```bash
git add workflows/storyboard-yokkorasho/pose3-kneeext.json
git commit -m "feat: よっこらしょ対策③膝伸ばしのラフ画ワークフローを追加"
```

---

### Task 8: 3ワークフローを実行し、生成画像を収集・確認

**Files:**

- Output: `workspace/output/storyboard_yokkorasho_pose{1,2,3}_*.png`

**Step 1: 実行**

```bash
.venv/bin/comfy --workspace workspace run workflows/storyboard-yokkorasho/pose1-basic.json
.venv/bin/comfy --workspace workspace run workflows/storyboard-yokkorasho/pose2-standup.json
.venv/bin/comfy --workspace workspace run workflows/storyboard-yokkorasho/pose3-kneeext.json
```

**Step 2: 収集**

Run: `ls workspace/output/ | grep storyboard_yokkorasho`
Expected: 3枚（またはバッチにより複数）のPNGが存在

**Step 3: 目視検証（Read ツールで各画像を確認）**

- 3枚共通: OpenPose骨格由来のポーズが参照フレームと一致しているか、モノクロ線画のラフスケッチ調になっているか、汎用人体（高齢者特有の特徴が出ていないか）
- **構図要件（Major指摘反映）**: 椅子の位置・向き・カメラフレーミングが意図通り表現されているか。プロンプトのみでは弱い場合、`ControlNetApplyAdvanced` の `strength` を下げる、または参照フレームをベースにした低 denoise（0.4〜0.6）の img2img 併用を代替手段として検討する
- **②専用の重点確認**: 前傾姿勢と重心移動＝「しなり」が視覚的に明確に表現されているか

基準未達の場合はプロンプト・`strength`・seedを調整して該当ワークフローのみ再実行する（別タスク化はしない反復調整）。

**Step 4: Commit**

生成画像自体は `workspace/output/`（gitignore対象）のためコミット対象なし。最終成果物のパスをユーザーに報告する。
