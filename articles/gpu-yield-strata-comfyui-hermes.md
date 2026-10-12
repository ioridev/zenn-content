---
title: "常駐LLMと画像・動画生成でGPUを譲り合う仕組み — Strata × ComfyUI × Hermes"
emoji: "🤝"
type: "tech"
topics: ["localllm", "comfyui", "gpu", "aiエージェント", "strata"]
published: true
---

ローカルLLMを常駐させたまま、エージェントに画像や動画の生成もさせたかった。でも LLM が GPU をほぼ占有しているので、ComfyUI と同時には載らない。

そこで GPU を「譲り合う」ラッパーを挟んだ。ポイントは、**ツールを実行している間、LLM は結果を待っているだけで何もしていない**こと。生成コマンドを実行している間だけ LLM を降ろして、終わったら戻せば会話は途切れない。

![](/images/gpu-yield-strata-comfyui-hermes/header.png)

:::message
全部ローカルで動かしている。ラッパーと設定は Claude Code に頼んで作ってもらった。
:::

## 環境

| 項目 | 内容 |
|---|---|
| GPU | RTX 2080 Ti 22GB（改造品）+ RTX 2080 Ti 11GB |
| LLM | Strata 0.1.42 + Qwen3.8-Flash-Next IQ3_S |
| 生成 | ComfyUI（画像・動画） |
| エージェント | Hermes Agent |

https://github.com/Niko1221/Strata

https://github.com/Comfy-Org/ComfyUI

https://github.com/NousResearch/hermes-agent

## 登場人物

- エージェント（Hermes）は会話して、必要ならツールで生成を指示する
- ローカルLLM（Strata）はエージェントの頭脳。普段は GPU をほぼ占有して常駐している
- 生成サーバ（ComfyUI）は画像や動画を作る。動くときは GPU を丸ごと使う
- ラッパー（strata-yield）は生成コマンドを包んで、GPU の受け渡しを仲介する小さなスクリプト

## 流れ

エージェントは生成コマンドを `strata-yield -- <生成コマンド>` の形で実行するだけ。あとはラッパーが順番にやってくれる。

1. LLM を降ろす（`/unload`）。返事を書いている途中なら、終わるまで待つ
2. 生成コマンドを実行して、ComfyUI のキューが空になるまで待つ
3. ComfyUI に VRAM を手放させる（`/free`）
4. LLM を読み込み直す（`/load`）
5. エージェントは戻ってきた LLM で結果を確かめて、返事する

LLM が降りている1〜4の間、GPU は丸ごと生成に回って、エージェントは待っているだけ。

## 安全装置

生成の途中で LLM にリクエストが来ても生成を落とさないように、LLM 側に安全装置を3つ入れている。

- 読み込み前のフックでラッパーを呼んで、ComfyUI の処理が終わるまで待ってから戻る
- 空き VRAM が LLM の載る分より少ないときは、読み込みを断って 503 を返す（22GB のカードで 20GB にしている）
- GPU を譲る処理はロックで同時に1件だけにする

## エージェントへの教え方

システムプロンプト（Hermes なら `environment_hint`）に「ComfyUI で GPU を使うコマンドは必ず `strata-yield --` で包む」と1段落書くだけ。スキルに載せているコマンド例も同じ形にそろえておくと、包み忘れはほぼなくなる。

## 最小構成

ラッパーの中身はこれだけ（疑似コード）。

```bash
# strata-yield -- <生成コマンド>
flock $LOCK                      # 同時に1件だけ
post $LLM/unload                 # 409（返事中）なら待って再試行
"$@"                             # 生成コマンドを実行
until comfy_queue_empty; do sleep 3; done
post $COMFY/free '{"unload_models":true,"free_memory":true}'
post $LLM/load &                 # 戻すのは裏で
```

LLM 側（Strata）の設定は2行。

```json
"before_load": ["strata-yield", "before-load"],
"min_free_vram_mib": 20000
```

Strata のこのへんの設定はここに書いてある。

https://github.com/Niko1221/Strata/blob/main/docs/DETAILS.md#sharing-the-gpu-with-other-programs-optional

## コツ

- LLM が戻るまでに1〜3分かかる。入れ替えのたびにモデルを読み直すので、生成用のモデルは SSD に置くと速い
- キューが空になるまで待つので、「投入だけしてすぐ戻る」コマンドでも大丈夫
- ブラウザから ComfyUI を直接使うときは、先に `/unload` を叩くだけ。次に話しかけたら LLM は勝手に戻ってくる

## ほかの環境

LLM を降ろす・載せる方法（API か再起動）があれば、Ollama や llama.cpp、vLLM でも同じ形で組めると思う。生成側も ComfyUI に限らず、「終わったか分かる」「VRAM を手放せる」の2つがあればいい。

ツールの実行中は LLM が暇、と気づいたらあとは単純だった。
