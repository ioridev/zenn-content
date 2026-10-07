---
title: "課金したくないのでローカル3Dモデル生成を試してみた（Pixal3D + Qwen-Image）"
emoji: "🤖"
type: "tech"
topics: ["comfyui", "生成ai", "3dcg", "qwen", "vrchat"]
published: true
publication_name: "bestat"
---

AI画像生成→Tripoでの３D生成→フロンティアモデルで編集みたいなワークフローが流行っている（？）
が、契約するサブスクAIが増え続ける一方なのでローカルAIで同じことができないか試してみた。

@[tweet](https://x.com/iori_sf/status/2104837198583660633?s=20)


やりたいのはこういうこと。

![](/images/local-image-to-3d-pixal3d/flow-chibi-robot.jpg)
*絵を1枚用意して、4方向の絵にして、3Dモデルにする（この2頭身ロボットは、Codex に4面図をまとめて描かせた）*

できたモデルはこれ。ドラッグでぐるぐる回せる。

@[codepen](https://codepen.io/jvnkmdto-the-scripter/pen/YPZEmEb?default-tab=result)

やったことはこのへん

- TencentARC の Pixal3D を ComfyUI で動かして、画像から PBR テクスチャ付きの GLB を作る
- 裏側を推測で作らせないために、正面・側面・背面の絵（マルチビュー）をどう用意するのがいいか比べる
- 毛むくじゃらの生き物と、硬い面のロボットで結果を比べる
- できたモデルを VRChat のワールドに置く

:::message
3D化（Pixal3D）と Qwen の画像生成はローカル。元絵（ポメとロボット）と比較用の4面図の一部は、Codex CLI の画像生成（クラウド）で作っている。
検証用のスクリプトを書いて回すところは Claude Code にやってもらった。
:::

## 環境

| 項目 | 内容 |
|---|---|
| GPU | RTX 2080 Ti × 2（22GB の改造品と普通の 11GB） |
| CPU / メモリ | Core i9-9820X / 62GB |
| OS | Ubuntu 26.04 |
| 3D生成 | Pixal3D（ComfyUI v0.37.4 の標準ノード） |
| 画像生成 | Qwen-Image-Edit-2511（GGUF）+ 角度 LoRA、Qwen-Image-2.1（ComfyUI 0.38.0） |
| 後処理 | Blender 4.5、Unity（VRChat SDK） |

2080 Ti は2018年の Turing 世代で、今どきの生成モデルを動かすには古い。それでもちゃんと動いた。

## Pixal3D

https://github.com/TencentARC/Pixal3D

TencentARC の画像→3D モデル。SIGGRAPH 2026 の論文で、ライセンスは MIT。今の版は TRELLIS.2 ベースになっている。

入力画像の画素の特徴を 3D に逆投影して対応を取るので、画像に写っている部分の再現度が高いのが売り。2026年9月には、複数枚の画像から作るマルチビュー推論も公開された。

ComfyUI 本体に Pixal3D のノードがあり、ComfyUI 用に再配布された重み（Comfy-Org/Pixal3D）で動く。マルチビューには ComfyUI v0.35.0 以降が必要。

中の流れはだいたいこう

1. BiRefNet で背景を消して、1024×1024 に切り出す
2. MoGe-2 で画角を推定し、DINOv3 で画像の特徴を取る
3. 粗い構造（32³）→ 形状（512→1024）→ 色と材質のボクセル、の順に作る
4. メッシュにして約10万三角形まで減らし、UV を展開する
5. ベースカラー・メタリック・ラフネス・法線・AO を 2048px のテクスチャに焼いて GLB で出す

出てくるのは1メッシュ・1マテリアルの GLB で、1体 17〜18MB くらい。

## 必要な VRAM とメモリ

解像度 1024 なら VRAM はピーク 9.9GB で、11GB のカードでも動いた。1枚の GPU で2本同時に回すと VRAM が足りなくなって、順番に流すより遅くなった。

メインメモリは1本あたり 12GB、重い設定だと 25GB くらい使う。ComfyUI v0.37 は空きメモリが少なくなるまで途中の結果を抱えたままにするので、重いジョブを2本同時に回したら OOM killer に落とされた。

## 2080 Ti でハマったところ

### UV 展開で落ちる

`UnwrapMesh` の `torch.linalg.solve` が `no kernel image` で落ちる。PyTorch 2.11（CUDA 13.0 版）に同梱の MAGMA が Turing（sm_75）で動かないのが原因で、環境変数で cuSOLVER を使わせると通る。

```bash
TORCH_LINALG_PREFER_CUSOLVER=1 python main.py
```

### bf16 が使えない

Turing は bf16 に対応していないので、ComfyUI は fp32 で計算する（ログに `manual cast: float32` と出る）。遅いけど、fp16 でありがちな NaN 問題は起きない。重みは int8 版で十分だった。bf16 版は1.7倍遅いのに、見た目はほぼ同じ。

## 生成時間

解像度 1024・int8 での実測。

| 入力 | 時間 |
|---|---|
| 画像1枚（アームチェア） | 4.9分 |
| 画像1枚（ロボット） | 6.3分 |
| 画像1枚（箱形のラジオ） | 8.8分 |
| 3方向（ポメ） | 5.0〜6.8分 |
| 4方向（ロボット） | 7.5〜12.0分 |

公式の既定値の解像度 1536 にすると、アームチェアで27分かかった。10万三角形まで減らす用途なら 1024 との差はほとんどないので、この記事は全部 1024。

モデル自体が 1024 までで学習されているので、面数を50万に増やしたりステップ数を増やしたりしても、見た目はほとんど良くならなかった。入力画像を 4K に拡大しても形は変わらない（モデルは必ず 1024px に縮めて見ている）。

## 画像1枚だと裏側が推測になる

画像1枚でも、写っている側はかなり忠実に出る。問題は見えていない側で、ここはモデルの推測になる。正面の絵から作ったロボットは背中に胸の装甲みたいな面が映り込んだし（後の比較画像の左端）、横向きの絵から作ったポメラニアンは反対側の毛並みがまだらになった。

もう1つハマったのが背景のにじみ。モデルに渡す画像は切り抜いた物体を黒い背景に置いたものなので、背景除去が甘いと輪郭の色に黒が混ざる。背景を透過した PNG を用意して、BiRefNet ではなく PNG のアルファで切り抜いたら出なくなった。

元絵はこういうのがうまくいく

- 物体は1つだけで、全体が画面に収まっていて、周りに余白がある
- 無地の明るい背景、柔らかい均一な光（強い影や映り込みがない）
- ボケなし、文字やロゴなし
- 1枚だけで作るなら、斜め前の少し上から見た 3/4 ビュー

それでも裏側は推測になるので、複数方向の絵を入れるマルチビューを試した。

## マルチビューの入力を作る

Pixal3D のマルチビューには、正面・左側面・背面・右側面の絵を入れる（2〜3枚でもいい）。目の高さから90°おきに、ほぼ平行投影（画角20°）で見た絵を想定している。

入れるときの注意が2つある。

- left は「物体の左側面」。絵の中では物体が画面の左を向く。左右非対称な試験体で確かめたら、左右を逆に入れると形が崩れた。2026年9月末に見た時点では、ComfyUI 公式テンプレートの例は左右が逆につながっていた（例のロボットが左右対称なので問題が出ないだけ）。
- 全部の絵を同じ縮尺にそろえる必要がある。絵ごとに物体の大きさで切り抜くと、しっぽの長い動物なんかは正面と側面で縮尺がずれる。なので、全部の絵を共通の縮尺で中央に置いてから入れるようにした。

で、肝心の絵をどう用意するか。Qwen-Image 2.1 にマルチビュー生成があるので、Codex で描かせるより安定するかもと思って、使えそうな方法を一通り試した。

| 方法 | 動く場所 | やり方 |
|---|---|---|
| Codex の4面図 | クラウド | 元絵を添付して、同じ物を4方向から描いた1枚（ターンアラウンド）を描かせる |
| Qwen-Image-Edit-2511 + 角度 LoRA | ローカル | 元絵からカメラの向きを変えた絵を1枚ずつ作る |
| Qwen-Image-2.1 の4面図 | ローカル | 白いキャンバスと元絵を渡して、4面図を1枚で描かせる |
| Qwen-Image-2.1 の透過出力 | ローカル | 背景を透過した PNG を出せるので、背景除去に使う |

### Codex の画像生成

Codex CLI（ChatGPT アカウント）の画像生成で描かせた。1枚1分くらい。

### Qwen-Image-Edit-2511 + Multiple-Angles LoRA

https://huggingface.co/fal/Qwen-Image-Edit-2511-Multiple-Angles-LoRA

画像編集モデルの Qwen-Image-Edit-2511 に、カメラの向きを指定できる LoRA（Apache 2.0）を組み合わせる。プロンプトはこういう形。

```text
<sks> right side view eye-level shot medium shot
```

方位（`front view`・`right side view`・`back view` など8方向）、高さ（`eye-level shot` など）、距離（`medium shot` など）の順に並べる。2080 Ti では GGUF の Q5_K_M 版と4ステップの Lightning LoRA を使って、1枚2〜2.5分だった。

:::details ComfyUI のノード構成
```text
UnetLoaderGGUF (qwen-image-edit-2511-Q5_K_M.gguf)
 → LoraLoaderModelOnly (Qwen-Image-Edit-2511-Lightning-4steps-V1.0-bf16)
 → LoraLoaderModelOnly (qwen-image-edit-2511-multiple-angles-lora)
 → ModelSamplingAuraFlow (shift 3.1)
 → CFGNorm
 → KSampler (4 steps, cfg 1.0, euler / simple, denoise 1.0)

CLIPLoader (qwen_2.5_vl_7b_fp8_scaled, type: qwen_image)
 → TextEncodeQwenImageEditPlus (prompt, image1 = 元絵)
 → FluxKontextMultiReferenceLatentMethod (index_timestep_zero)
   ※ネガティブ側は空のプロンプトで同じ構成

元絵 → FluxKontextImageScale → VAEEncode (qwen_image_vae) → KSampler の latent
```
:::

使ってみてわかった癖

- 向きの指定がややこしい。正面の元絵に `right side view` を頼むと、被写体の左側面（画面の左を向いた絵）が出てきた。カメラが右に回り込む、という意味っぽい。一方で、左側面の元絵に `right side view` を頼んだら反対側の右側面が出て、`left side view`（元絵と同じ側）は崩れた。出てきた絵の向きを目で見てから割り当てるのが確実。
- `back view` は真後ろじゃなくて、斜め後ろになりがち。
- 元絵に写っていない特徴は消える。ポメの正面図からほかの向きを作ったら、正面からは見えない大きな尾びれが普通のしっぽになった。特徴が写っている絵から作る必要がある。

### Qwen-Image-2.1

Qwen-Image-2.1 は4チャンネルの VAE を持っていて、背景を透過した PNG を直接出せる。「背景を完全に消して、物体はそのまま」と頼むと、きれいに切り抜けた（1枚2.4分）。さっきの黒にじみ対策にちょうどいい。

:::details 背景透過のプロンプト
```text
Remove the background completely. Keep the creature exactly as it is: same pose, position, size,
framing, colours, fur and gills, nothing added or changed. Transparent background.
```
`TextEncodeQwenImage21` の `images.image_1` に元の絵を入れて、25ステップ・cfg 1.0 で生成している。
:::

4面図も1枚で描ける。ただ出力のサイズが最初に渡した画像に合わせられるので、1枚目に白いキャンバス、2枚目に元絵を渡した。4面図1枚に9〜10分かかる。一方、1方向ずつ「正面」などと頼むと、顔だけこっちを向いた斜めの絵になりがちだった。

:::message alert
Qwen-Image-2.1 のライセンスは Qwen Research License で、研究・評価目的にしか使えないので注意（商用で使うには別途ライセンスが必要）。Qwen-Image-Edit-2511 のほうは Apache 2.0。
:::

## ポメラニアン×ウーパールーパーで比較

最初の題材は、ウーパールーパーのエラと半透明の大きな尾びれを持つポメラニアン（元絵は Codex で生成）。ふわふわの毛と透ける尾びれで、3D にしにくい要素の組み合わせ。

![](/images/local-image-to-3d-pixal3d/pom-inputs.jpg)
*上が Codex に描かせた4面図、下が横向きの元絵から Qwen で作った3面（背面は使わない）*

Codex の4面図は一見そろっているけど、背面の尾びれが側面と矛盾している。側面で広い面が見えている尾びれなら、真後ろからは薄く見えるはず。なのに背面図でも、広い面がこっちを向いている。

Pixal3D に入れた結果

![](/images/local-image-to-3d-pixal3d/pom-results.jpg)
*左から Codex 4面図、Codex 3面（背面なし）、横向き1枚、Qwen で作った3面＋透過*

| 入力 | 結果 |
|---|---|
| Codex 4面図 | 目の上にも黒いかたまりが出て、目が4つあるように見える。毛も薄い板の重なり（うろこ状）に崩れた |
| Codex 3面（背面なし） | 背面を抜くと毛はマシになったけど、目ははっきり4つ |
| 横向き1枚 | 一番崩れなかった。顔もちゃんと1つにまとまった。見えていない側の毛並みがまだらなのと、黒背景のにじみは出た |
| Qwen 3面＋透過 | これも目が4つ。鼻のまわりにも黒い帯が出て、エラもトゲトゲになった |

一番致命的なのは顔の崩れ。正面の絵と横の絵で目の位置が合っていないので、Pixal3D がどっちの目も描いてしまい、目が4つになる。

![](/images/local-image-to-3d-pixal3d/pom-faces.jpg)
*顔を正面から見たところ。多視点にした3つは、どれも目がもう1組ある*

横向き1枚は見えていない側を推測で作るぶん雑になるけど、絵が1枚なので食い違いが起きず、顔がちゃんと1つにまとまる。黒背景のにじみは、あとでテクスチャを塗り直して消した（後述）。

毛はどの方法でも薄い板や筋の集まりになるので、ここはモデルの限界っぽい。

## ガンダムっぽいロボットで比較

そもそも毛むくじゃらの生き物はむずいので、ガンダムみたいなオリジナルのちょっと複雑なロボットでも試した。硬い面と細かい装甲、背中のスラスターなど、前と後ろで形がはっきり違う題材。

![](/images/local-image-to-3d-pixal3d/robot-front.jpg =400x)
*元絵（Codex で生成）*

:::details 元絵のプロンプト
```text
an original humanoid mecha robot of my own design (not any existing franchise): angular white armour
plates over a dark grey mechanical inner frame, a few red and cobalt-blue accent panels, broad layered
shoulder armour, chest air intakes, a single horizontal visor sensor instead of eyes, a helmet with two
short swept-back blade antennas (no V-shaped crest, no yellow), a thruster backpack with two nozzles
peeking out behind the shoulders, detailed panel lines and small vents, standing upright in a neutral
pose, arms held slightly away from the body, feet shoulder-width apart, full body from head to feet
```
スタイルは「clean hard-surface 3D render, realistic painted metal, soft even studio light」を指定した。4面図は、この絵を添付して「同じデザインで、正面・左側面・背面・右側面を横1列に」と頼んでいる。
:::

この元絵から、3つの方法で4方向の絵を作った。

![](/images/local-image-to-3d-pixal3d/robot-inputs.jpg)
*上から Codex の4面図、Edit-2511 + 角度 LoRA（左端は元絵そのまま）、Qwen-Image-2.1 の4面図*

どれも背中のスラスターまで描けているけど、細部はちょっとずつ違う。Edit-2511 は背中に赤い部品を足して、Qwen-Image-2.1 は背中に大きな青いバックパックを付けた。

これを Pixal3D に入れた結果。比べるために、元絵1枚からの結果も並べている。

![](/images/local-image-to-3d-pixal3d/robot-results-4view.jpg)
*4方向を入れた結果（左端は元絵1枚から）*

| 入力 | 結果 |
|---|---|
| 元絵1枚 | 背中に、胸の装甲みたいな青い面が映り込んだ |
| Codex 4面図 | 一番きれい。前後ともまとまっていて、背中のスラスターも自然 |
| Qwen-2.1 4面図 | 背中の装備が大きな青い箱になって、推測が強め |
| Edit-2511 4面＋透過 | 形はいいけど、生成した向きに赤い差し色が増えて、背中がごちゃつく |

一番きれいだった Codex 4面図版はこれ。

@[codepen](https://codepen.io/jvnkmdto-the-scripter/pen/bNqYXae?default-tab=result)

背面を抜いた3方向でも作ってみた。

![](/images/local-image-to-3d-pixal3d/robot-results-3view.jpg)
*背面を抜いて3方向にした結果*

どれも後ろ姿が推測になって、4方向のほうがよかった。絵が少ないほうがよかったポメとは逆の結果。

## 1枚か多視点か

Qwen のほうが Codex より安定するかも、という予想はあまり当たらなかった。ロボットは Codex の4面図が一番で、ポメはそもそも多視点にしない横向き1枚が一番崩れなかった。2つの比較をまとめるとこんな感じ。

- どの向きから描いても矛盾が出にくい硬い形の物（ロボット、家具、小物など）は**4方向**。背面の情報がそのまま役に立つ。
- 毛やヒレ、エラのような複雑な形で、多視点の絵が安定しない物は**横向き1枚**。生成した別の向きの絵は細部が少しずつ食い違っていて、Pixal3D が全部に合わせようとする。ポメでは目の位置が合わずに目が4つになり、毛も板の重なりやトゲトゲになった。特徴が一番写る向きの1枚だけなら矛盾がなく、見えない側は推測させたほうが崩れない。
- どっちの場合も、背景は透過 PNG で入れるのが安全。
- 全部ローカルで多視点を作るなら、Edit-2511 + 角度 LoRA で向きを作って、Qwen-Image-2.1 で透過する組み合わせが一番よかった。ロボットでは Codex に一歩及ばなかったけど、形はほぼ同じくらい。

## VRChat のワールドに置く

比較で一番崩れなかった横向き1枚版のポメを、自作の VRChat ワールドに置いた。

![](/images/local-image-to-3d-pixal3d/aether-pom.jpg)
*ワールドに置いたところ（Unity エディタでの描画）*

Pixal3D の出力はそのまま置くとゲームには重いので、手を入れている。

生の GLB に Blender の Decimate をそのままかけると、UV の継ぎ目でメッシュが割れて穴が開いた。しかも出力には、外から見えない内側の殻がある。手元で調べたモデルでは、三角形の約4割が外面の 5〜6mm 内側にある裏向きの面だった。

なので、頂点を溶接 → 160方向から描画して一度も見えない面を削除 → 減面 → UV を作り直し → 元の高ポリから色・材質・法線を焼き直す、という手順にした。ポメは約2万4千三角形まで減らした。

黒にじみは、暗くて色味のないテクセルを選んで、3D 的に近い毛の色で塗り直した（表面の約8%）。

![](/images/local-image-to-3d-pixal3d/pom-clean.jpg)
*背中の黒にじみを消した前後*

塗り直したあとのポメはこれ。

@[codepen](https://codepen.io/jvnkmdto-the-scripter/pen/KwWyOZm?default-tab=result)

Unity は GLB をそのまま読めないので、Blender で FBX にして取り込んだ。ORM テクスチャ（AO・ラフネス・メタリック）は、Standard シェーダー用のメタリック／スムースネスに詰め替えている。

## 使ってみた感じ

2018年の 2080 Ti でも、1体5〜12分で PBR テクスチャ付きの約10万三角形の GLB が作れた。回数を気にせず、シード違いをたくさん作って選べるのはローカルのいいところ。

結果を一番左右したのは、モデルの設定じゃなくて**入力の絵**だった。解像度や面数を上げてもほとんど変わらないのに、多視点にするかどうかや、その絵の作り方で大きく変わる。

苦手なのは、ふわふわの毛（板や筋になる）、透ける物や細い物、人の手（指がくっついた板になる）、全身を作ったときの顔（解像度が足りない）あたり。

Tripo と同じ画像で比べてはいないので、どっちがきれいかは言えない。ただ、一番きれいだったデカロボットでも、テクスチャがにじんだ感じになっていて、生成した3Dっぽさが出る。なので、近くでじっくり見せる物というより、VRChat のワールドに置く小物くらいなら手元の GPU だけで十分作れる、という感じ。

## 参考リンク

- Pixal3D: https://github.com/TencentARC/Pixal3D
- Pixal3D の ComfyUI 用の重み: https://huggingface.co/Comfy-Org/Pixal3D
- Qwen-Image-Edit-2511（GGUF）: https://huggingface.co/unsloth/Qwen-Image-Edit-2511-GGUF
- Multiple-Angles LoRA: https://huggingface.co/fal/Qwen-Image-Edit-2511-Multiple-Angles-LoRA
- Lightning LoRA: https://huggingface.co/lightx2v/Qwen-Image-Edit-2511-Lightning
- Qwen-Image-2.1: https://huggingface.co/Qwen/Qwen-Image-2.1 （ComfyUI 用: https://huggingface.co/Comfy-Org/Qwen-Image-2.1 ）
- ComfyUI-GGUF: https://github.com/city96/ComfyUI-GGUF
