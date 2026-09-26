> [!IMPORTANT]
> **このrepositoryは、RVC WebUIを祖先に持つfork独自の次世代声変換エンジンです。alpha段階の手元検証と実用は完了し、beta移行のためにコミュニティ支援を募集しています。**
>
> Windows専用だったRVC realtime系を、Intel Macの学習・推論からmacOS Audio Unit、WebUI所有runtime、Bonjour制作LANまで再設計しました。GarageBand標準offline Bounceで、実モデルによる単独vocalとオケ付きmixの変換を実用水準で完走しています。
>
> **beta移行に必要なもの: コード、物資(検証機材)、資金(電気代・投げ銭)、ミュージシャン・音楽スタジオ。** → [コミュニティ支援の募集](#beta移行のためにコミュニティ支援が必要です)
>
> 開発正本: [Issue #1](https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI/issues/1) / 上流との関係: [upstream追従の終了](#上流との関係-upstream追従の終了)

# RVC次世代エンジン fork — macOS / Audio Unit登攀版

![GarageBandでRVCRealtime AUを標準offline Bounce中](assets/fusamofu-img/AUv2inside.png)

**Windows用WebUIとVSTのスクリーンショットだけだったRVCを、Intel Mac上での学習・推論からGarageBand AUv2、実モデル変換、標準offline Bounce、runtime障害復旧まで貫通させた実機記録です。** 画像はUI mockではなく、オケ付きmixをGarageBandが実際にバウンスしている最中のhost画面です。

上流のWindows VST実装を単に起動しただけではありません。このforkでは、CPU学習・推論、共有RVC engine、Audio Unit、stream protocol、WebUI所有runner、異常終了Recoveryまでを一つの経路として実装しました。

## 到達点

| 領域 | 現在の実測 |
| --- | --- |
| Intel Mac / Python 3.12 | 前処理、F0/特徴抽出、CPU学習、実モデル推論を確認 |
| macOS Audio Unit v2 | Release build、codesign、`auval`成功 |
| GarageBand | AU挿入、実モデル接続、単トラックSolo Bounce、オケ付きMix Bounce成功 |
| 標準offline render | `OFFLINE`、約1183 ms、`0 drop`、host通知1回を実機画像で確認 |
| runtime architecture | WebUIがPython/RVC processを所有し、AUは`127.0.0.1:17865`のRSVC thin headとして動作 |
| 死活管理 | **live-host fault-injection test**: runtimeを3回連続`SIGKILL`して自動復旧を確認。さらにWebUIへ`SIGTERM`を注入し、所有runner・gateway・dns-sdの回収を確認 |
| realtime再生 | Intel CPUでは130 ms blockに対し推論約1.18秒のためプチプチする。未合格 |
| Bonjour / LAN | WebUI所有の広告・探索、local gateway、自己発見・明示選択、GarageBand AU→gateway→self backend→実RVC変換→offline Bounceまで単一Mac実測・Human Gate合格 |

## デモソング

**文部省唱歌「村祭」 RVCデモ**

- 1番: ずんだもん
- 2番: デルタもん
- 3番: 元の生歌唱 ふさもふ

[![文部省唱歌「村祭」 RVCデモ - ずんだもん / デルタもん / 生ふさもふ](https://img.youtube.com/vi/Y91K-4o8xz0/hqdefault.jpg)](https://youtube.com/shorts/Y91K-4o8xz0?si=g-VA_kcmIiyq3ZEP)

▶ [YouTube Shortsでデモを再生](https://youtube.com/shorts/Y91K-4o8xz0?si=g-VA_kcmIiyq3ZEP)

Human listeningの合格と機械試験は混同していません。詳細receiptは次にあります。

- [GarageBand offline Bounceとオケ付きMixのHuman Gate](experiments/20260904-1015__garageband-offline-bounce-in-progress.ja.md)
- [live-host fault-injection test: runtime 3連続SIGKILLと自動Recovery](experiments/20260904-1055__garageband-runtime-sigkill-recovery.ja.md)
- [Bonjour切替後の実RVC offline Bounce Human Gate](experiments/20260904-2013__bonjour-route-offline-human-pass.ja.md)
- [WebUI SIGTERM時の所有process回収](experiments/20260904-2050__webui-sigterm-live-host-fault-injection.ja.md)
- [WebUI所有runtimeとasset routing](experiments/20260903-1920__webui-owned-runtime-and-asset-routing.ja.md)
- [macOS AU thin head build/deploy](experiments/20260903-1848__macos-au-rsvc-thin-head-build-deploy.ja.md)

## architecture

```text
GarageBand / Logic / VST host
        |
        | RVCRealtime thin head
        | audio callbackにPython・model load・process起動を置かない
        v
RSVC realtime audio stream
  localhost: 127.0.0.1:17865
        |
        v
RVC WebUI / controller
  UI・control: 127.0.0.1:7865
  health、runner所有、model/index選択、bounded restart
        |
        v
共有RVC realtime engine
  CPU / CUDAなし / Intel Macを含むbackend
  model load、HuBERT、F0、SOLA、RMS、推論
```

GarageBandのsandbox内からPythonをspawnする旧案は廃止しました。runtimeが落ちた場合もGarageBandを巻き込まず、WebUI側が所有processだけを再生成します。AU画面の`RUNTIME / SCAN / SELECT`はWebUIが検出した一覧をlocalhost control APIから取得するだけで、AU自身はBonjour browseしません。モデル実パスもruntimeだけが所有し、AUにはopaque IDと表示名だけを返します。`WebUI default`はWebUIの既定モデルへ追従し、AUでモデルを明示選択した場合はそのAU sessionの指定が優先されます。この選択はRSVC `SESSION_OPEN`を通るため、Bonjourで選んだremote runtimeでも同じ契約です。通常のaudio callbackはnon-blockingを維持し、hostが標準offline propertyを通知した場合だけ、deadlineのないBounceとして推論完了を待ちます。

## 現在の製品境界

- **offline Bounce:** Intel Mac実機で実用合格。Soloとオケ付きMixを聴感確認済み
- **realtime monitoring:** 現在のIntel CPUでは性能未達。[Issue #35](https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI/issues/35)で最適化または高火力remote backendを追跡
- **Bonjour:** WebUI/controller所有の発見・明示選択を実装し、単一Macの自己発見、UI Human Gate、GarageBand AUからself backendの実変換とoffline Bounceまで合格。別Mac実測は外部実機で再開するペインステータス凍結
- **複数client:** 初期実装はlocal制作環境を対象とし、1 runtimeへ複数AU/Web clientが接続した場合の排他、公平性、資源予約は保証しません。SaaS化する場合はclient session管理とsession単位のworker orchestrationが別途必要です。
- **model/data:** `.pth`、`.index`、学習素材、生成audioはrepositoryへ同梱しない

## 次世代エンジンとしての段階

上流RVC WebUIからの差分は、移植修正の範囲を超えたmajor version相当の再設計です。

| 層 | 上流 | このfork |
| --- | --- | --- |
| ML stack | Windows・CUDA中心 | architectureから見直し、Intel Mac / Python 3.12でCPU前処理・特徴抽出・学習・推論を完走。CPU学習の完走モデルは3本（receiptは`experiments/`） |
| DAW plug-in | Windows VST2/VST3 | 既存VST境界を保ったままAudio Unit v2を追加。`auval`、GarageBand挿入、設定slot保存・復元まで確認 |
| runtime server | plug-inごとのworker | WebUIがPython/RVC processを所有し、AUは薄いaudio head。RSVC stream protocol、bounded restart、SIGKILL/SIGTERM fault injection合格 |
| network | なし | Bonjourによる広告・探索・明示選択、local gateway。Jam Session型の制作LAN runtimeへ拡張可能な構造 |
| 実用性 | — | GarageBand標準offline BounceでSolo/オケ付きMixを実用合格 |

### alpha: 完了

- 開発者の手元実機（Intel Mac / macOS 15 / GarageBand）で、学習からDAW Bounceまでを一つの経路として通した
- 実際の歌唱制作に使える水準を、人間の耳によるHuman Gateで確認した
- 失敗・blocked・未試験・Recoveryを`experiments/`へ残した

### beta: これから

betaは新機能の追加ではなく、手元で通った経路を他人の機材とスタジオへ届ける段階です。

- 一発installer（Python環境、依存、AU/VSTの配置、初回model取得）
- 互換性整備: Windows VST実機回帰、Apple Silicon、Logic Pro、CUDA機
- 別Mac間Bonjour、Wi-Fi断、LAN latency/drop
- realtime monitoring（Intel CPUでは未達。高火力backendまたは最適化、[Issue #35](https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI/issues/35)）
- 複数clientの排他・公平性

これらは設計で詰まっているのではなく、検証機材と電力で詰まっています。現在の開発機は12年運用のIntel機1台で、Windows実機、複数Mac、Apple Silicon、CUDA機がありません。「Windowsは大丈夫か」への現在の正確な回答は、資源未提供につきUNKNOWNです。未検証を互換保証とは書きません。

## beta移行のためにコミュニティ支援が必要です

alphaは開発者一人の手元資源で閉じました。betaは一人では閉じません。次の4つを募集します。

| 必要なもの | 具体例 |
| --- | --- |
| **コード** | installer、Windows回帰、Apple Silicon backend、CoreML/ONNX、test、docs。Issue / Pull Request歓迎 |
| **物資** | Apple Silicon Mac、Windows + NVIDIA機、audio interface、検証用の貸出・中古提供 |
| **資金** | 開発機の電気代、機材費への投げ銭。回収を急がないimpact投資・Patient Capitalの相談 |
| **ミュージシャン・スタジオ** | 実際の制作現場でのbeta試用、DAW・機材構成ごとのreceipt、他スタジオへの展開 |

実機報告には、OS、CPU/GPU、DAW、audio device、sample rate、block size、model backend、drop/latency、Bounce結果を添えてください。成功だけでなく失敗も等しく価値のあるreceiptです。

支援・投資・スタジオ展開の考え方は、開発者のnoteにまとめています。

- 技術解説: [GarageBandを生かしたまま、AIランナーだけを3回殺した話](https://note.com/fusamofu326/n/n1a29ca1ef393)
- 支援・投資の考え方: [元ベンチャー社長、現職NEETの、私が求める雇用主（正確にはPatient Capital / Impact Patron像）を説明する](https://note.com/fusamofu326/n/neb54d4397dc5)
- 連絡: [Issue](https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI/issues) / [YouTube @fusamofu](https://youtube.com/@fusamofu)

## 上流との関係: upstream追従の終了

このforkは上流RVC-Projectへの提出を前提とした開発stagingとして始まりましたが、次の観測事実により、upstreamへ差分を戻す開発方針を終了し、fork独自の次世代エンジンとして進化させます。

- 上流repositoryはPull Request機能が無効（`has_pull_requests: false`）で、提出用分岐`upstream/macos-au-webui-runtime`をPRとして提出できなかった
- そのため実装済み提案を[上流Issue #2854](https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/issues/2854)として提出したが、反応がない
- 上流側にIntel Mac、macOS/AU、DAW実機で統合検証する経路が見えず、これ以上upstreamへ寄せる技術的な利益がない
- こちらの検証が閉じない原因は設計ではなく検証機材と電力であり、これは上流ではなく独自の資金・支援で解決する

方針:

- 上流の著作権表示、MIT License、来歴は保持する（[LICENSE](LICENSE)、[RVCRealtime/THIRD_PARTY_NOTICES.md](RVCRealtime/THIRD_PARTY_NOTICES.md)）
- 上流はrevisionを固定した参照元として扱い、必要な修正だけを出典付きで取り込む
- 上流がPull Requestを再開し取込方法を示した場合、汎用差分の提供は拒まない
- 記名・知財境界は[Issue #39](https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI/issues/39)の非攻性防壁を維持する。これは上流への攻撃ではない
- repository名称の変更とGitHub fork networkからの切り離しは、検証機材と電力を調達した後に行う

判断記録: [experiments/20260926-upstream-independence-decision.ja.md](experiments/20260926-upstream-independence-decision.ja.md)

---

以下は祖先である上流RVC WebUIのREADMEです（来歴として保持）。

<div align="center">

<h1>Retrieval-based-Voice-Conversion-WebUI</h1>
简单易用的 语音音色转换/变声器 框架<br><br>

[![madewithlove](https://img.shields.io/badge/made_with-%E2%9D%A4-red?style=for-the-badge&labelColor=orange
)](https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI)

<img src="https://counter.seku.su/cmoe?name=rvc&theme=r34" /><br>

[![Licence](https://img.shields.io/badge/LICENSE-MIT-green.svg?style=for-the-badge)](https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/blob/main/LICENSE)
[![Huggingface](https://img.shields.io/badge/🤗%20-Models-yellow.svg?style=for-the-badge)](https://huggingface.co/lj1995/VoiceConversionWebUI/tree/main/)


[**更新日志**](./docs/cn/Changelog_CN.md) | [**常见问题解答**](https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/wiki/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98%E8%A7%A3%E7%AD%94) | [**AutoDL·5毛钱训练AI歌手**](https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/wiki/Autodl%E8%AE%AD%E7%BB%83RVC%C2%B7AI%E6%AD%8C%E6%89%8B%E6%95%99%E7%A8%8B) | [**对照实验记录**](https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/wiki/%E5%AF%B9%E7%85%A7%E5%AE%9E%E9%AA%8C%C2%B7%E5%AE%9E%E9%AA%8C%E8%AE%B0%E5%BD%95) | [**在线演示**](https://modelscope.cn/studios/FlowerCry/RVCv2demo)

[**English**](./docs/en/README.en.md) | [**中文简体**](./README.md) | [**日本語**](./docs/jp/README.ja.md) | [**한국어**](./docs/kr/README.ko.md) ([**韓國語**](./docs/kr/README.ko.han.md)) | [**Français**](./docs/fr/README.fr.md) | [**Türkçe**](./docs/tr/README.tr.md) | [**Português**](./docs/pt/README.pt.md)

</div>

> 底模使用接近50小时的开源高质量VCTK训练集训练，无版权方面的顾虑，请大家放心使用

> 请期待RVCv3的底模，参数更大，数据更大，效果更好，基本持平的推理速度，需要训练数据量更少。

<table>
   <tr>
		<td align="center">训练推理界面</td>
		<td align="center">实时变声界面</td>
	</tr>
  <tr>
		<td align="center"><img src="https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/assets/129054828/092e5c12-0d49-4168-a590-0b0ef6a4f630"></td>
    <td align="center"><img src="https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/assets/129054828/730b4114-8805-44a1-ab1a-04668f3c30a6"></td>
	</tr>
	<tr>
		<td align="center">go-webui.bat</td>
		<td align="center">go-realtime_gui.bat</td>
	</tr>
  <tr>
    <td align="center">可以自由选择想要执行的操作。</td>
		<td align="center">我们已经实现端到端170ms延迟。如使用ASIO输入输出设备，已能实现端到端90ms延迟，但非常依赖硬件驱动支持。</td>
	</tr>
</table>

## 简介
本仓库具有以下特点
+ 使用top1检索替换输入源特征为训练集特征来杜绝音色泄漏
+ 即便在相对较差的显卡上也能快速训练
+ 使用少量数据进行训练也能得到较好结果(推荐至少收集10分钟低底噪语音数据)
+ 可以通过模型融合来改变音色(借助ckpt处理选项卡中的ckpt-merge)
+ 简单易用的网页界面
+ 可调用pymss/MSST模型来快速分离人声和伴奏
+ 使用最先进的[人声音高提取算法InterSpeech2023-RMVPE](#参考项目)根绝哑音问题，速度快、资源占用小
+ A卡/I卡使用 CPU 依赖方案；Windows 可使用 DirectML，Linux 使用 CPU

点此查看我们的[演示视频](https://www.bilibili.com/video/BV1pm4y1z7Gm/) !

## 环境配置

本分支面向 **Python 3.12 x64**，请先进入仓库根目录。Ubuntu 推荐使用 Ubuntu 24.04 x86_64。

### Ubuntu 24.04

```bash
sudo apt update
sudo apt install -y python3.12 python3.12-venv python3.12-dev ffmpeg unzip libsndfile1 libportaudio2

python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip setuptools wheel
```

### Windows

安装 Python 3.12 x64 后创建虚拟环境：

```powershell
py -3.12 -m venv .venv
.venv\Scripts\activate
python -m pip install --upgrade pip setuptools wheel
```

### 按硬件选择依赖

| 硬件 | 安装方式 |
| --- | --- |
| CPU、AMD、Intel | 使用 `requirments_cpu_py312.txt`；Windows 可使用 DirectML，Linux 使用 CPU |
| NVIDIA RTX 50 系 | 先安装 CUDA 12.8 版 Torch，再安装 `requirments_cu128_py312.txt` |
| NVIDIA RTX 50 系以前 | 先安装 CUDA 11.8 版 Torch，再安装 `requirments_cu118_py312.txt` |

#### CPU、AMD、Intel

```bash
python -m pip install -r requirments_cpu_py312.txt
```

#### NVIDIA RTX 50 系：两阶段安装

```bash
python -m pip install torch==2.7.1+cu128 torchaudio==2.7.1+cu128 \
  --index-url https://download.pytorch.org/whl/cu128 \
  --extra-index-url https://pypi.org/simple
python -m pip install -r requirments_cu128_py312.txt
```

#### NVIDIA RTX 50 系以前：两阶段安装

```bash
python -m pip install torch==2.7.1+cu118 torchaudio==2.7.1+cu118 \
  --index-url https://download.pytorch.org/whl/cu118 \
  --extra-index-url https://pypi.org/simple
python -m pip install -r requirments_cu118_py312.txt
```

检查 Torch 与 CUDA 状态：

```bash
python -c "import torch; print('torch:', torch.__version__); print('cuda:', torch.version.cuda); print('cuda available:', torch.cuda.is_available())"
```


### 修改下载源

三个 `requirments_*.txt` 顶部已经包含下载源。中国大陆用户可保留默认镜像；需要使用官方源时，只替换 `--index-url` 和 `--extra-index-url`，保留包版本、CUDA 后缀和两阶段顺序。

| Default mirror | Official source |
| --- | --- |
| `https://mirrors.pku.edu.cn/pypi/simple` | `https://pypi.org/simple` |
| `https://mirrors.nju.edu.cn/pytorch/whl/cpu` | `https://download.pytorch.org/whl/cpu` |
| `https://mirrors.nju.edu.cn/pytorch/whl/cu118` | `https://download.pytorch.org/whl/cu118` |
| `https://mirrors.nju.edu.cn/pytorch/whl/cu128` | `https://download.pytorch.org/whl/cu128` |

## 模型与运行目录

WebUI 会自动创建运行目录。模型请从 [Hugging Face 模型仓库](https://huggingface.co/lj1995/VoiceConversionWebUI/tree/main) 下载，并保持以下路径：

```text
assets/
├── hubert_base/
│   ├── config.json
│   ├── preprocessor_config.json
│   └── pytorch_model.bin
├── rmvpe/rmvpe.pt
├── pretrained/
├── pretrained_v2/
├── pymss_weights/
├── weights/        # user RVC .pth models
└── indices/        # user .index files
logs/
└── mute/           # training silence samples

# Exact paths used by the code
assets/hubert_base/config.json
assets/hubert_base/preprocessor_config.json
assets/hubert_base/pytorch_model.bin
assets/rmvpe/rmvpe.pt
assets/pretrained/*.pth
assets/pretrained_v2/*.pth
assets/pymss_weights/*
assets/weights/*.pth
assets/indices/*.index
logs/mute/*
```

### 下载模型

```bash
python -m pip install --upgrade huggingface_hub

# Required for inference and feature extraction
hf download lj1995/VoiceConversionWebUI --revision main \
  --include "hubert_base/*" --local-dir assets
hf download lj1995/VoiceConversionWebUI rmvpe.pt --revision main \
  --local-dir assets/rmvpe

# Required for v1/v2 training
hf download lj1995/VoiceConversionWebUI --revision main \
  --include "pretrained/*" "pretrained_v2/*" --local-dir assets
hf download lj1995/VoiceConversionWebUI mute.zip --revision main \
  --local-dir .model-downloads
python -m zipfile -e .model-downloads/mute.zip logs

# Required only for pymss/MSST vocal separation
hf download lj1995/VoiceConversionWebUI --revision main \
  --include "pymss_weights/*" --local-dir assets
```

仅 Windows AMD/Intel DirectML 环境还需要：

```bash
hf download lj1995/VoiceConversionWebUI rmvpe.onnx --revision main \
  --local-dir assets/rmvpe
```

### FFmpeg

Ubuntu 已在前面的系统依赖命令中安装 FFmpeg。Windows 用户可把下面两个文件放到项目根目录：

- [ffmpeg.exe](https://huggingface.co/lj1995/VoiceConversionWebUI/resolve/main/ffmpeg.exe?download=true)
- [ffprobe.exe](https://huggingface.co/lj1995/VoiceConversionWebUI/resolve/main/ffprobe.exe?download=true)

## 开始使用

启动 WebUI：

```bash
python webui.py
```

无桌面的 Ubuntu 服务器：

```bash
python webui.py --noautoopen
```

默认服务监听端口为 `7865`。用户自己的 `.pth` 模型放入 `assets/weights/`，`.index` 文件放入 `assets/indices/`。

## 参考项目
+ [ContentVec](https://github.com/auspicious3000/contentvec/)
+ [VITS](https://github.com/jaywalnut310/vits)
+ [HIFIGAN](https://github.com/jik876/hifi-gan)
+ [Gradio](https://github.com/gradio-app/gradio)
+ [FFmpeg](https://github.com/FFmpeg/FFmpeg)
+ [Ultimate Vocal Remover](https://github.com/Anjok07/ultimatevocalremovergui)
+ [pymss-project/pymss](https://github.com/pymss-project/pymss)
+ [audio-slicer](https://github.com/openvpi/audio-slicer)
+ [Vocal pitch extraction:RMVPE](https://github.com/Dream-High/RMVPE)
  + The pretrained model is trained and tested by [yxlllc](https://github.com/yxlllc/RMVPE) and [RVC-Boss](https://github.com/RVC-Boss).

## 感谢所有贡献者作出的努力
<a href="https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/graphs/contributors" target="_blank">
  <img src="https://contrib.rocks/image?repo=RVC-Project/Retrieval-based-Voice-Conversion-WebUI" />
</a>
