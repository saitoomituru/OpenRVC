# GarageBandを生かしたまま、AIランナーだけを3回殺した話

Windows専用だったRVCを、Intel Mac一台で学習からGarageBandの本番bounceまで通した記録

![GarageBandでRVC Realtime AUv2を挿し、伴奏付きの「村祭」をoffline bounceしている実機画面](../assets/fusamofu-img/AUv2inside.png)

> 画像ファイル: `assets/fusamofu-img/AUv2inside.png`

🎵 実演動画：[ずんだもん・デルタもん・生声で歌う「村祭」](https://youtube.com/shorts/Y91K-4o8xz0)

---

## 先に結論を言う。GarageBandの再生中に、私は裏でAIの声変換エンジンを3回殺した

普通、これは事故として報告される数字だ。だが今回は違う。私自身がPIDを確認し、狙って、生きたセッションの最中にプロセスへシグナルを送った。3回とも、GarageBandは落ちなかった。曲は止まらず、裏で死んだエンジンだけが自動的に生き返り、また声を変換し始めた。

これは偶然「バグが少なかった」という話ではない。設計でそうなるように作った、という話だ。以下、その根拠を書く。

## なぜ大手じゃなくて寺子屋がこれをやるのか

RVC(Retrieval-based Voice Conversion)は、世界中で歌声変換や配信、キャラクターボイスに使われている技術だ。本家のRealtime実装は、WindowsのWebUIとVST2/VST3を主戦場にしている。READMEに並ぶのもWindowsの画面。macOS、しかもApple SiliconではなくIntel Macで、学習からGarageBandのAudio Unitまで通す海図は、どこにもなかった。

ここで「新しいMacを買え」は、正しいが何の役にも立たないアドバイスだ。潤沢な資本のR&Dなら、遅いマシンは廃棄してGPUを積んだ新型を発注すれば済む。だが寺子屋MADサイエンスでは、手元にある12年選手のHackintosh Proが全戦力だ。だから「遅いマシンがどこまで登れるか」を試すことそのものが研究になる。

これは負け惜しみではない。実際、この記事の技術的な勝ち筋の大半は、マシンが遅かったからこそ見えたものだ。それは後述する。

## 最初の設計判断——「プラグインに全部持たせる」を自分で捨てた

最初に思いついたのは、AU側からPythonやRVCのworkerを直接起動し、モデルを読み、共有メモリで音声を往復させる構成だった。単体VSTの発想としては自然に見える。

だがAppleのAudio Unitにとって、プラグインは自分だけの独立国家ではない。GarageBandというホストに同居し、ホストの実行境界、sandbox、終了手順に従う立場だ。音声スレッドは期限付きの現場であり、Pythonの起動やモデルのロード、ネットワーク待ちを積む場所ではない。コールバックの期限に遅れれば、音楽は理由を聞いてくれない。ただプチッと切れる。

そこで役割を分けた。RVCはもともとWebUIの背後で動くワーカーとして育っている。ならばWebUIが持つエンジン・モデル・Python環境をそのまま活かし、AUは音声の「頭」だけを担う。死んでいたらWebUIが起こし、生きていたらそれを使う。この判断ひとつが、後の全ての安定性を支えることになった。

```text
GarageBand / AUv2
    ↓  軽量なC++ audio head
localhost control / audio gateway
    ↓
WebUIが所有するruntime supervisor
    ↓
RVC engine / Python / PyTorch / model
```

## Intel Macでの学習は「未確認」ではない。実際に2モデル、完走した

学習したのは40kHz、F0あり、v2の歌唱モデル。デルタもん側では65ファイル・約805秒の入力から233 clipsを生成し、無音1本は「成功っぽく通過」させず明示的にskipした。F0抽出(RMVPE)、HuBERT特徴量抽出は233/233成功。

CPU学習は1 epoch約25分。20 epochの完走は翌朝7時19分だった。低火力のIntel CPUで、DDPのTCPStore bind失敗、DataLoader workerのハング、moduleの解決崩壊といった地雷を踏みながら、CPUをsingle process化し、DataLoader workerを0にし、`python -m`のmodule entrypointへ統一して通した。

書いておきたいのはここだ。**これは「未確認」の話ではない**。実際にCPUだけで2モデルが完走し、人間の耳で声質・古語訛り・ビブラート・しゃくりに重大なNGがないことを確認している。ただし、低火力Intel Macで数十時間かかる再学習を何度も繰り返す品質保証はしていない。事実の強度を下げているのではなく、実機資源の境界を正直に書いているだけだ。

## Appleの境界に3回焼かれ、3回とも境界の側に合わせた

ここからが今回の一番の武勇伝だ。3つの障壁、3つとも「Macだから無理」ではなく「境界の選び方が悪かった」だけだった。

**1. `shm_open`が`EPERM`で拒否された。**
GarageBand上でIPCを繋ぐと、単体CLIでは通っていたPOSIX名前付き共有メモリの作成が拒否された。AUはGarageBandのプロセス内でホストされる立場であり、素のTerminalプロセスと同じ穴が開いていると考える方が間違っている。解決は`$TMPDIR`配下の通常ファイルを`open`・`ftruncate`し、`mmap(MAP_SHARED)`で共有する方式への切り替えだった。Appleのsandboxを迂回するのではなく、用意された通り道に形を合わせた。

**2. OpenMPが3回目の推論で必ず落ちた。**
`libiomp5.dylib`の`__kmp_abort_process`、SIGABRT。ログを回数として読むと、必ず3回目の推論中に落ちていることが見えた。OpenMPが遅延的に追加スレッドを作ろうとし、ホストの実行境界で`pthread_create`が失敗し、致命的abortに入るという仮説を立て、`OMP_NUM_THREADS=1`で固定した。推論速度は約550msから約1097msへ落ちたが、落ちる高速化に価値はない。まず死なない経路を作り、速度は別の戦場で取り返す判断をした。

**3. MIDIを1本も使わないのに、CoreMIDIの初期化で止まった。**
MIDI input/outputが両方無効な構成でも、iPlug2はCoreMIDI初期化の経路を律儀に走らせていた。使わない装備の点呼で出航が止まるのは無駄なので、無効な構成ではCoreMIDI初期化とnull MIDI device選択を丸ごとskipするよう1ファイル・17行足す最小パッチを当てた。これはRVC固有の都合ではなく、iPlug2側の一般的な改善として本家へ返せる性質のものだ。

## 「ベイク」と言った一言のせいで、存在しない独自仕様が生えた話

これは失敗を武勇伝として書く。開発中、私は「重い処理は事前にベイクし、待った後はラグなしで再生したい」と伝えた。Blender的な語彙だ。この言葉を受けた実装側は、AUの中に独自のBAKE機能とcacheを生やしてしまった。

だが音楽業界には、そのためにすでにある道具があった。DAWのoffline render——GarageBandで言うバウンスだ。ホストが`kAudioUnitProperty_OfflineRender`を通知すれば、プラグインはリアルタイムの締切を越えて処理完了を待てる。標準規格の名前が違うだけで、目的は完全に同じだった。

これはAIの理解力の問題ではなく、私の言葉の設計(Fold Map)が甘かったことが原因だ。異なるフレームワーク間で目的語彙がずれたとき、名称の空白から独自仕様が生える——これはAI開発全般に一般化できる事故パターンだと思う。独自BAKEは凍結し、標準のoffline renderに置き換えた。

## リアルタイムでは負けた。だから製品全体を負けにしなかった

実測すると、Intel CPUの1ブロック推論は約1.2〜1.3秒。リアルタイム音声の締切より明らかに遅い。画面には500回以上のdropが出て、耳にはプチプチ聞こえる。これは気合いで直す不具合ではなく、現在のハードウェアと計算量の関係だ。

だから「Realtime」というラベルに全てを従わせるのをやめた。歌のレコーディングでは、常にリアルタイムで完成音を出す必要はない。操作中は簡易monitorで構成を決め、最後のbounceで時間をかければいい。

結果として、単一トラックのsolo bounceと、伴奏付きmixのbounce、両方を実際にGarageBand標準のoffline機能で通した。リアルタイム再生ではプチプチするのに、書き出したファイルはぬるぬると変換されている。低火力Intel Macに対する勝ち筋は、ここで確定した。**「Intel MacでRVCがリアルタイム動作した」とは書かない。書けるのは「GarageBand標準offline bounceが、単トラックでも伴奏付きmixでも変換効果ありプチプチなしで通った」ということだけだ。** この線引きを最後まで崩さなかったこと自体が、この開発の誠実さの証拠になっている。

## Bonjourで、WebUIを「ローカルの裏方」から「制作LANのランタイム」へ広げる

エンジンをWebUI側に出したことで、次に見えたのはネットワークだった。Macが一台でも、localhostの直結経路と、Bonjourで広告・探索・resolveした経路は切り替えられる構造にした。将来、軽量なMacのGarageBandから、同じ制作LANにいる高火力マシンのRVCを拾える——Appleの「Jam Session」的な手触りで、声変換の計算だけを外に出す発想だ。

独自のセキュリティルールは足さなかった。BonjourはmacOSの`dns-sd`/mDNSResponderがそのまま持つ仕組みで、パケットがどこまで届くかはOSとネットワークの責務に委ねた。これはSaaSのテナント分離を目指すものではなく、信頼した制作LANの道具として設計している。

単一Mac上の試験では、WebUIが自分自身を`_rvc-realtime._tcp.local`として広告し、自分で発見し、Bonjour selfへ切り替えてセッションを張り直し、実推論まで通った。**ただしこれは別Macまで到達した証拠ではない。**現在の実機は1台だけであり、自己広告・自己発見・経路切替・handshake・実推論の合格までが今回の到達点だ。Wi-Fi断、2台間の実音、Windows Bonjour相互運用は、実機かcontributorが現れたときに開く凍結項目として明示している。

## モデル名はパスではない、というもう一つの地雷

Bonjourで別マシンのruntimeへ繋ぐ以上、AUがファイルパスを直接持つのはおかしい。クライアント側のパスとruntime側のパスは一致しない。そこでモデル実体とindexのパスはruntimeが所有し、AUへは安定したopaque model IDと表示名だけを渡す設計にした。

実際、ずんだもんのモデルをGarageBandの設定slotに保存・復元したところ、表示が人間向けの名前ではなくID(`rvc-d5d27b9ef69f373b`のような文字列)のまま出てしまうバグを踏んだ。修正は、保存すべきIDと見せるべき表示名を分離し、状態復元処理とaudio callbackにはネットワークI/Oを一切入れず、UIのアイドルスレッド側でカタログから表示名を解決する方式にしたことだ。修正後、保存・復元・表示名の再解決・変換音の復帰まで人間の耳と目で確認した。

## 偶然死ぬのを待たず、再生中にこちらから殺しにいった

冒頭の話に戻る。私はエラーハンドリングを「薄く広くバグを数える作業」だとは思っていない。当たれば制作中の仕事そのものを壊す地雷を、狙って潰すことが重要だと考えている。

だから「3回テストした」ではなく、再生中のライブフォルトインジェクションをやった。GarageBandでプロジェクトを再生している最中に、バックエンドのruntimeプロセスへ狙ってシグナルを送る。PID、PPID、listener、所有関係を確認した上で、対象だけを殺した。

ランナーを止めるとAUは一時的に変換出力を失うが、GarageBandは落ちない。WebUI側のsupervisorがランナーの死亡を検出し、回数制限付きの再生成でgatewayの先を戻す。AUは再接続し、READYへ復帰する。これを再生中に3回行い、3回ともホストは生き残った。

逆方向のフォルトインジェクションもした。WebUI本体にSIGTERMを送る。修正前はWebUIだけが死に、ランナーとBonjourの子プロセスがPPID 1の孤児として残り、ポートを握り続けていた。修正後はsignalハンドラで重いcleanupをせず、既存の終了経路にSIGTERMを渡し、外側の`finally`で所有childを回収するようにした。WebUI、runner、advertiser、browser、listenerが全て正しく消えることを確認した。

## 「通った」を一つの言葉で済ませなかった

開発中、私は成功の境界を何度も書き分けた。buildの成功はDAWでの成功ではない。`auval`の成功は実モデルの声質変換成功ではない。CLIで非無音が返ったことは、GarageBandのsandbox内で同じように動く証拠ではない。バウンスファイルが生成されたことは、声が正しく変わった証拠ではない。最後は必ず人間が耳で聞いて判定した。

Human Gateとして確認したのは以下だ。

- GarageBandがAUv2を認識し、GUIを表示する
- 実モデルで声質が変わる
- pitch `-12 st`でオクターブ下の歌声になる
- WebUI既定モデルとAU明示モデルを切り替えられる
- GarageBandの設定slotを保存・復元し、表示名と変換が戻る
- LocalhostとBonjour selfを切り替え、セッションを張り直せる
- ランナー先行障害でGarageBandが生存し、ランナーが再生成される
- WebUI終了時に所有childとlistenerが回収される
- 単トラックと伴奏付きmixの標準offline bounceが完了し、聞いて変換効果がありプチプチがない

一方で、未確認は正直に残す。Windows実機回帰、Apple Silicon、Logic Pro、別Mac間のBonjour、Wi-Fi断、複数クライアントの排他・公平性・資源予約は未試験だ。これらを「おそらく大丈夫」で埋めることはしない。機材とcontributor待ちの凍結項目として、はっきり公開している。

## 港が閉じていたので、旗を立てる場所を変えた

forkの`main`には開発中の実験票や自動開発の運用記録が積まれている。これをそのまま本家へ投げるのは筋が違う。そこで本家`main`を起点に、汎用source・tests・最小docsだけを抽出した提出用分岐`upstream/macos-au-webui-runtime`(本家に対し8 commits ahead、71 files、約5,774 insertions / 218 deletions)を作った。

だがPull Requestを出そうとしたところ、GitHubは`CreatePullRequest`権限を拒否し、`/pulls`は404を返した。分岐関係もcompareも正常に成立するので、原因は本家repository側の`has_pull_requests: false`設定——つまり港そのものが今は閉じているという事実だった。

だから旗の立て方を変えた。実装はIssueとして提案し、実装・実機検証者、開発支援ツール、実装正本の公開forkを明記した。採用する場合はcommit authorかChangelogの出典を残してほしい、と普通に書いた。

- 本家への実装済み提案: [RVC-Project/Retrieval-based-Voice-Conversion-WebUI #2854](https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/issues/2854)
- 公開fork: [saitoomituru/Retrieval-based-Voice-Conversion-WebUI](https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI)
- 本家と提出分岐の比較: [`main...upstream/macos-au-webui-runtime`](https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/compare/main...saitoomituru:Retrieval-based-Voice-Conversion-WebUI:upstream/macos-au-webui-runtime)

これは本家の運用を批判する話ではない。中央管理型の運用、資本、法務、国際配布条件は、技術者本人の動機とは別の層にある。こちら側にできるのは、派生・出典・実装者を公開commitとして残し、本家のライセンスを守り、モデルや音声を混ぜないことだ。攻撃ではなく、海賊なりの非攻性の防壁として設計した。詳細は[記名・知財境界の凍結票 #39](https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI/issues/39)に分けている。

## 遅いマシンは、失敗したマシンではない

今回いちばん面白かったのは、性能不足がアーキテクチャをむしろ正直にしたことだ。高速なGPUがあれば、Pythonやモデルやネットワークの境界を曖昧にしたまま「なんか動いた」で押し通せたかもしれない。Intel Macは遅い。だからどこで待っているか、どこでスレッドが増えるか、どのプロセスがポートを持つか、どの状態がセッション固有か、全部見えた。

リアルタイム処理に失敗したときはサンプルを捨ててホストを守る。offline renderに入ったときは完了を待って音質を取る。ランナーが死んだときはAUがエンジンの内部に手を突っ込まず、WebUIが所有責任で戻す。remote runtimeを選ぶときはAUがremoteのファイルパスを知らず、opaque IDでモデルを指す。これらは全部、遅いマシンが要求した正直さだ。

そして最後に、GarageBandのバウンスボタンを押した。進捗バーは遅い。リアルタイムで聞こえたプチプチを思い出しながら待つ。書き出しが終わり、再生する。ずんだもんがオクターブ下で「村祭」を歌う。伴奏も一緒だ。プチプチはない。

それで十分だ。いや、それをやるためにここまで来た。

## 次の海域は、機材を持っている人に開いている

コードは公開している。成功だけでなく、失敗、blocked、未試験、Recovery、Human Gateを`experiments/`に残している。空のrepositoryに「これから作ります」と旗だけ立てたのではない。GarageBandで実際に書き出した音があり、殺したプロセスのPIDと、生き返ったセッションのログがある。

機材を持っている人、音楽を作っている人、macOSのオーディオの境界が好きな人、RVCを別マシンへ飛ばしたい人は、続きに参加してほしい。「Windowsではどうなんだ」と聞くだけでなく、Windows実機で試してログを持ってきてほしい。Apple Siliconがあるならbuildしてほしい。2台のMacがあるならBonjourで繋いでほしい。

無償のOSSに、持っていない機材の品質保証まで抱え込む義務はない。その代わり、持っている資源で何を通し、何を通せなかったかは、ごまかさず公開する。続きの海図はそれで十分描ける。

## リンク

- 開発fork: https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI
- 本家実装提案Issue #2854: https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/issues/2854
- 提出用分岐比較: https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/compare/main...saitoomituru:Retrieval-based-Voice-Conversion-WebUI:upstream/macos-au-webui-runtime
- fork側の開発親Issue #1: https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI/issues/1
- Bonjour / runtime選択Issue #29: https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI/issues/29
- Intel CPU realtime性能Issue #35: https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI/issues/35
- runtime障害復旧Issue #36: https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI/issues/36
- WebUI / AU model選択Issue #38: https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI/issues/38
- 記名・知財境界の凍結票 #39: https://github.com/saitoomituru/Retrieval-based-Voice-Conversion-WebUI/issues/39

---

### 著者と来歴

- 実装・実機検証: 齋藤みつる / saitoomituru
- 開発支援: Codex / Grok / Gemini
- 実機: Hackintosh Pro(Intel、12年運用の自作機) / macOS 15.7.7 / GarageBand 10.4.14 / Python 3.12.7 / Xcode 16.2
- モデル・学習音声・indexは本記事およびrepositoryでは配布しない

本記事は、上記の公開repository、Issue、commit、実験票、GarageBandの実機Human Gateに基づく。未確認の環境については成功を主張しない。