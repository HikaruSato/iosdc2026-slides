---
theme: default
title: 映像変換サーバーなしで端末内でHLSを生成してライブ配信
info: |
  iOSDC Japan 2026 レギュラートーク（40分）
author: Hikaru Sato
colorSchema: light
fonts:
  sans: Hiragino Sans, Noto Sans JP, sans-serif
  mono: SFMono-Regular, Menlo, monospace
lineNumbers: true
transition: fade-out
aspectRatio: 16/9
canvasWidth: 1280
exportFilename: iosdc2026-hls-on-iphone
---

<div class="cover">
  <div class="cover-copy">
    <div class="event-line">iOSDC Japan 2026 · Regular Talk · 40 min</div>
    <h1>
      <span class="cover-line">映像変換サーバーなしで</span>
      <span class="cover-line"><span class="accent-primary">iPhone端末内で</span></span>
      <span class="cover-line"><span class="accent-primary">HLS</span>を生成してライブ配信</span>
    </h1>
    <div class="speaker">Hikaru Sato <span>@SatoHikaruDev</span></div>
  </div>
  <div class="cover-visual" aria-hidden="true">
    <div class="cover-file-stack">
      <span>playlist.m3u8</span>
      <b>init.mp4</b>
      <b>000001.m4s</b>
    </div>
  </div>
</div>

<!--
[Timing checkpoint: 00:00]

全Timing checkpointは練習用の仮目安で、実測済みの所要時間ではありません。
通常は本編を話し、Optional detailは時間超過時だけの予備とします。
デモ込みの通し練習で35〜37分を目指し、実測後に各チェックポイントを調整します。

今日は、iPhone自身をエンコーダー兼セグメンターにして、生成したHLSをS3へ順次アップロードし、ライブ配信にした実装を話します。
いきなり実装へ入らず、まずHLSがどのファイルをどう更新するとライブ配信になるのかを確認します。その全体像へ今回のiPhone実装を当てはめ、早い段階でデモを動かします。
-->

---
class: speaker-intro
---

<div class="kicker">ABOUT ME</div>

<div class="speaker-intro-layout">
  <div class="speaker-intro-copy">
    <h1>Hikaru Sato</h1>
    <div class="speaker-intro-handle">@SatoHikaruDev</div>
    <div>
      主に iOS / Android / Ruby on Rails のアプリ開発をやっています
    </div>
    <div class="speaker-intro-app">
      <p><a href="https://apps.apple.com/jp/app/%E3%82%B0%E3%83%AB%E3%83%BC%E3%83%97%E5%8B%95%E7%94%BB%E5%85%B1%E6%9C%89-momentnow/id6759968148" style="color: var(--accent-primary); text-decoration: underline;">MomentNow</a> という「今この瞬間」の動画をHLSで配信し、URLで共有できる iOSアプリ を個人開発してます</p>
    </div>
  </div>

  <div class="speaker-intro-portrait">
    <img src="/assets/speaker-hikaru.jpg" alt="SatoHikaruDev profile image" />
  </div>
</div>

<div class="speaker-intro-claim">今回は MomentNow での端末内HLS生成の仕組みや実装を分解します</div>

<!--
[Timing checkpoint: 00:20]

佐藤光、@SatoHikaruDevです。
今この瞬間の動画をHLSで配信し、共有URLから見られるiOSアプリ「MomentNow」を個人開発しています。
今日は、このアプリで必要になった端末内のHLS生成と公開順の実装を、公開サンプルと一緒に分解します。

[Sources]
- fortee: iOSDC Japan 2026 speaker profile image
- MomentNow-iOS/README.md
-->

---

<div class="kicker">TODAY'S ROUTE</div>

# 話すこと

<div class="chapter-overview">
  <div><b>WHY</b><span>なぜ端末内HLS生成をやろうと思ったのか</span></div>
  <div><b>01</b><span>HLSがライブになる仕組み</span></div>
  <div><b>02</b><span>端末内でHLSを生成</span></div>
  <div><b>03</b><span>動画断片の生成</span></div>
  <div><b>04</b><span>playlistの更新</span></div>
  <div><b>05</b><span>配信の完了</span></div>
</div>

<!--
最初に、なぜ端末内でHLSを作ろうと思ったのか、そのきっかけと実装できるまでの経緯を話します。
続いてHLSがライブになる仕組みを確認し、動くサンプルアプリを見ます。
その後、カメラとマイクの入力、fMP4生成、保存とplaylist公開、停止時の完了管理まで順番に追います。
Writerへ渡す時刻の扱いは、生成処理の中で短く触れます。
-->

---

<div class="kicker">WHY</div>

# 撮影後のアップロード時間を短縮したかった

<div class="d">
  <div class="d-lane d-axis-grid"><b>時間 →</b><span style="grid-column:2/5">撮影開始</span><span style="grid-column:10/14">撮影停止</span></div>
  <div class="d-lane"><b>1ファイル</b><span class="d-bar" style="grid-column:2/10">撮影</span><span class="d-bar warn" style="grid-column:10/14">まとめてupload</span></div>
  <div class="d-lane"><b>HLSの撮影</b><span class="d-bar blue" style="grid-column:2/10">撮影しながらsegmentを生成</span></div>
  <div class="d-lane"><b>HLSの送信</b><span class="d-bar blue" style="grid-column:4/6">seg 1</span><span class="d-bar blue" style="grid-column:6/8">seg 2</span><span class="d-bar blue" style="grid-column:8/10">seg 3</span><span class="d-bar warn" style="grid-column:10/12">最後</span></div>
  <div class="d-caption">模式図：撮影中に送信を進め、停止後の大きなuploadを減らす</div>
</div>
<div class="bottom-claim">同じplaylist URLで共有。停止後も最後の送信と終了処理は残る</div>

<!--
この実装のきっかけは、子どもの動画を家族へ送るときの待ち時間でした。
1ファイルでは撮影後に大きな動画をuploadします。HLSなら撮影中からsegmentを順次uploadし、同じplaylist URLを共有できます。
図は処理の重なりを示す模式図で、実測の所要時間ではありません。停止時にも最後のfragmentとENDLISTの公開が残るので、待ち時間が完全にゼロになるという説明にはしません。
-->

---

<div class="kicker">ORIGIN STORY · IMPLEMENTATION</div>

# [<span class="title-phrase">macOS向けAppleサンプルが</span><wbr><span class="title-phrase">とても参考になった</span>](https://developer.apple.com/documentation/avfoundation/writing-fragmented-mpeg-4-files-for-http-live-streaming)

<div class="implementation-story origin-story">
  <div class="attempt"><span>2025.03</span><b>AIで最初の試作</b><small>HLSとしての<br>実用まで至らず</small></div>
  <i>→</i>
  <div class="reference"><span>Apple公式</span><b>生成手順を確認</b><small>Writer設定<br>時刻補正<br>Dataの受け取り</small></div>
  <i>→</i>
  <div class="adapt"><span>iPhone local</span><b>HLS生成に成功</b><small>カメラ・マイク<br>短い動画を生成</small></div>
  <i>→</i>
  <div class="product"><span>MomentNow</span><b>ライブ配信へ発展</b><small>撮影中からupload<br>playlistを更新</small></div>
</div>

<div class="bottom-claim">端末内生成で変換サーバーを減らせると分かり、MomentNowの開発へ進んだ</div>

<div class="source">Apple: Writing fragmented MPEG-4 files for HTTP Live Streaming</div>

<!--
HLSを端末内で生成できれば、映像変換サーバーのコストを最小限にできると考え、実装を試行錯誤していました。
2025年3月ごろ、最初はAIへ実装させてみましたが、再生できるHLSとして実用まで到達できませんでした。

突破口になったのがApple公式のfmp4Writerです。
公式プロジェクトはmacOS 11以降向けのCommand Line Toolで、movie fileをAVAssetReaderで読み込みます。
一方、AVAssetWriterのHLS profile、URLなしWriter、10秒の時刻offset、segment delegateという核心部分はiOSでも利用できます。

入力をAVCaptureVideoDataOutputとAVCaptureAudioDataOutputへ置き換え、iPhone内でinit.mp4とm4sを生成できました。
端末内生成なら変換サーバーを持たずに配信できると分かり、MomentNowのライブ配信機能として開発を進めました。
このトークは、そのローカル生成を配信として成立させるまでの設計を扱います。

[Sources]
- https://developer.apple.com/documentation/avfoundation/writing-fragmented-mpeg-4-files-for-http-live-streaming
- https://developer.apple.com/videos/play/wwdc2020/10011/
-->

---
layout: center
class: chapter
---

<div class="chapter-no">01</div>
<div class="chapter-rule"></div>

# HLSで「ライブ」になる仕組み
<p>短い動画ファイル + 更新され続けるplaylist</p>

<!--
[Timing checkpoint: 02:30]

最初に、HLSのライブ配信が何を配っているのかを大きく捉えます。
HLSでは、長い動画を短い断片へ分け、その再生順を示すplaylistを配ります。ライブ中はplaylistが更新され続けます。

[Sources]
- RFC 8216: HTTP Live Streaming
-->

---

<div class="kicker">HLS FILE REFERENCES</div>

# playlistが、取得するファイルと再生順を示す

<div class="d d-link-map">
  <div class="d-line">#EXT-X-MAP:URI="init.mp4"</div><i>→</i><div class="d-file"><b>init.mp4</b><small>再生設定</small></div>
  <div class="d-line">#EXTINF:2.000,<br>seg/000001.m4s</div><i>→</i><div class="d-file"><b>000001.m4s</b><small>最初の約2秒</small></div>
  <div class="d-line">#EXTINF:2.000,<br>seg/000002.m4s</div><i>→</i><div class="d-file"><b>000002.m4s</b><small>次の約2秒</small></div>
</div>
<p class="d-note">Media Playlistの抜粋。ヘッダーなどは省略</p>
<div class="bottom-claim">Playerはplaylistを読み、書かれたURIからHTTPで取得する</div>

<!--
Playerはplaylistを取得し、そこに書かれた順でinitとsegmentを取得します。
ライブ中は同じplaylistを再取得すると、末尾へ新しいsegmentが増えています。大きな動画が完成するのを待つ必要はありません。

[Sources]
- RFC 8216: Media Playlist / Media Segment / EXT-X-MAP
-->

---

<div class="kicker">THREE FILE TYPES</div>

# HLSライブ配信を構成する、3種類のファイル

<div class="file-strip">
  <div class="file-type playlist-file">
    <span class="file-ext">M3U8</span>
    <b>playlist.m3u8</b>
    <small>更新される再生順</small>
  </div>
  <div class="file-type init-file">
    <span class="file-ext">MP4</span>
    <b>init.mp4</b>
    <small>再生を始めるための設定</small>
  </div>
  <div class="file-type media-file">
    <span class="file-ext">M4S</span>
    <b>000001.m4s</b>
    <small>2秒前後の映像と音声</small>
  </div>
</div>

<div class="bottom-claim">init.mp4（再生準備）+ m4s（約2秒）+ playlist（順番）= 最初の再生</div>

<!--
playlist、初期化セグメント、メディアセグメントの3種類です。
initだけにも、m4sだけにも、完全な再生体験はありません。playlistが関係を定義します。

[Sources]
- RFC 8216: Media Playlist / Media Segment / EXT-X-MAP
-->

---

<div class="kicker">FRAGMENTED MP4</div>

# fMP4は、再生設定と映像・音声の断片を分ける

<div class="d">
 <div class="d-row">
  <div class="d-node"><span>init.mp4 · 最初に1回</span><b>再生設定</b><small>圧縮方式や<br>映像・音声の構成</small></div>
  <i class="d-arrow">＋</i>
  <div class="d-node blue fill"><span>m4s · 約2秒ごと</span><b>映像 ＋ 音声</b><small>断片 1　断片 2　断片 3 …</small></div>
 </div>
 <div class="d-arrow down">↓</div>
 <div class="d-node blue"><span>playlistが両方のURIと順番を結ぶ</span><b>Playerが組み合わせて再生する</b></div>
</div>
<div class="bottom-claim">init単体・m4s単体を、完成した録画ファイルとして扱わない</div>

<!--
fragmented MP4、略してfMP4は、再生設定と映像・音声本体を分けて扱えるMP4です。
init.mp4にはftypとmoov、各m4sにはmoofとmdatが入ります。
後ほどAVAssetWriterDelegateから、この単位のDataを受け取ります。

[Sources]
- https://developer.apple.com/videos/play/wwdc2020/10011/
-->

---

<div class="kicker">PLAYLIST GET · 1 / 2</div>

# 最初は、playlistを読んで再生に必要なファイルを取得

<div class="d">
 <div class="d-row"><div class="d-node"><span>Player</span><b>playlistをGET</b></div><i class="d-arrow">→</i><div class="d-node blue"><span>サーバーのplaylist</span><b>init ＋ segment 1</b></div></div>
 <div class="d-arrow down">↓</div>
 <div class="d-row"><div class="d-node"><span>① 再生設定を取得</span><b>GET init.mp4</b></div><i class="d-arrow">→</i><div class="d-node blue fill"><span>② 映像・音声を取得</span><b>GET 000001.m4s</b></div></div>
</div>
<div class="bottom-claim">再生可能なファイルが揃う。再生開始のタイミングはPlayerにも依存する</div>

<!--
Playerの視点で最初の取得を追います。
まずplaylistをGETし、そこに記されたinit.mp4と最初のsegmentを取得します。順番は理解のための模式図で、Playerの具体的な取得・バッファリング実装を固定するものではありません。
[Sources]
- RFC 8216: Media Playlist / EXT-X-MAP
-->

---

<div class="kicker">PLAYLIST GET · 2 / 2</div>

# 同じURLを再取得すると、新しいsegmentが見つかる

<div class="d">
 <div class="d-row"><div class="d-node"><span>前回のplaylist</span><b>init ＋ segment 1</b><small>すでに取得した内容</small></div><i class="d-arrow">→</i><div class="d-node blue fill"><span>今回のplaylist</span><b>init ＋ segment 1・2</b><small>末尾にsegment 2が追加</small></div></div>
 <div class="d-arrow down">↓</div>
 <div class="d-row"><div class="d-node"><span>Playerが新しいURIを発見</span><b>GET 000002.m4s</b></div><i class="d-arrow">→</i><div class="d-node blue"><span>映像と音声を追加</span><b>続きへ再生が進む</b></div></div>
</div>
<div class="bottom-claim">ライブ中は繰り返す。終了時のENDLISTで「これ以上増えない」と伝える</div>

<!--
同じURLでもplaylistの本文が増えます。Playerは再取得し、新しいsegmentのURIを見つけて続きを取得します。
サーバーが動画をPlayerへpushしているのではなく、Player側からHTTP GETしていることが重要です。
今回のEVENT playlistは過去のsegmentも保持します。
[Sources]
- RFC 8216 §4.3.3: Media Playlist Tags
-->

---

<div class="kicker">FROM HLS TO THIS TALK</div>

# HLSを作る場所を、サーバーからiPhoneへ

<div class="architecture-compare">
  <div class="arch-row muted">
    <div class="arch-label">一般的</div>
    <div class="arch-node">配信端末<small>capture / live encode</small></div><div class="arch-arrow labeled">映像を連続送信<small>RTMPなど</small></div>
    <div class="arch-node server">Media server<small>受信 / encode / segment</small></div><div class="arch-arrow">→</div>
    <div class="arch-node">CDN</div><div class="arch-arrow">→</div>
    <div class="arch-node">Viewer</div>
  </div>
  <div class="arch-row">
    <div class="arch-label">今回</div>
    <div class="arch-node phone">iPhone<small>capture / encode / segment</small></div><div class="arch-arrow">→</div>
    <div class="arch-node">API + S3<small>object storageへ保存</small></div><div class="arch-arrow">→</div>
    <div class="arch-node">CloudFront<small>CDN</small></div><div class="arch-arrow">→</div>
    <div class="arch-node">Viewer</div>
  </div>
</div>

<div class="bottom-claim">今回なくすのは、映像を受信し続けて変換するサーバー</div>

<!--
一般的なライブ配信では、配信端末が映像をサーバーへ送り続け、サーバー側が受信、encode、segment化してHLSを作ります。
RTMPは代表例であり、配信システムによってSRTやWebRTCなど異なる入力方式もあります。

HLSでライブになる仕組みはそのまま使い、今回の構成ではsegment生成の場所をiPhoneへ移します。
iPhoneが生成済みのHLSファイルをオブジェクトストレージへ置くので、バックエンドは映像バイト列を変換しません。

バックエンドがゼロという意味ではありません。
API、保存、配信、状態管理は残ります。なくすのは、映像を受け取り続けて変換する役割です。

[Sources]
- Apple WWDC20 Session 10011: Author fragmented MPEG-4 content with AVAssetWriter
-->

---
layout: center
class: statement
---

<div class="kicker">THIS TALK'S IMPLEMENTATION</div>

# iPhoneが生成したHLSを<br><span class="accent-primary">撮影中からS3へ公開する</span>

<div class="result-flow end-to-end">
  <div class="result-step"><b>映像と音声を受け取る</b><span>Camera + Mic</span></div>
  <div class="result-arrow">→</div>
  <div class="result-step strong"><b>約2秒に分ける</b><span>AVAssetWriter</span></div>
  <div class="result-arrow">→</div>
  <div class="result-step"><b>保存先へ置く</b><span>HTTP PUT</span></div>
  <div class="result-arrow">→</div>
  <div class="result-step"><b>再生順を公開する</b><span>playlist更新</span></div>
</div>

<!--
録画開始後、init.mp4を1回、m4sを約2秒ごとに生成します。
各DataをS3へPUTし、保存に成功したsegmentだけをplaylistへ反映すると、視聴者が再生できる範囲が伸びていきます。
-->

---

<div class="kicker">PERMISSIONS · PUBLIC SAMPLE</div>

# 撮影・保存・接続に必要な権限

<table class="permissions-table">
  <thead><tr><th>権限</th><th>用途</th><th>許可するタイミング</th></tr></thead>
  <tbody>
    <tr><th>カメラ</th><td>配信映像の撮影</td><td>撮影開始時</td></tr>
    <tr><th>マイク</th><td>配信音声の録音</td><td>撮影開始時</td></tr>
    <tr><th>写真へ追加</th><td>完成したMP4の保存</td><td>配信開始前</td></tr>
  </tbody>
</table>

<div class="bottom-claim">Info.plistに用途を記載。写真は撮影終了後の「追加のみ」を要求する</div>

<!--
[本編必須: デモ前の権限確認]

初回デモの前に、カメラ・マイク・写真への追加を許可します。許可済みなら毎回ダイアログは出ません。
このSampleはカメラとマイクをプレビュー開始前に、写真への追加を配信開始前に確認します。
どれかが拒否されている場合は配信を開始しません。拒否した権限は設定アプリから許可します。
写真は完成したMP4を追加するだけなので、既存写真の読み取り権限は要求しません。

Info.plistの対応:
- カメラ: NSCameraUsageDescription。AVCaptureDevice.requestAccess(for: .video)。
- マイク: NSMicrophoneUsageDescription。AVCaptureDevice.requestAccess(for: .audio)。
- 写真への追加: NSPhotoLibraryAddUsageDescription。PHPhotoLibrary.requestAuthorization(for: .addOnly)。
- 同じLANのMacへの通信: NSLocalNetworkUsageDescription。実際にLANへ接続するときにシステムが許可を求めます。

ローカルネットワーク権限は、MacのLAN内IPへHTTP接続するデモで必要です。
公開ngrok URLやS3への通常のインターネット通信に、この権限が一律に必要という意味ではありません。
拒否するとLANの接続確認やPUTに失敗するため、デモ前に接続確認も済ませます。
NSAllowsLocalNetworkingというATS設定と、ユーザーが許可するローカルネットワーク権限は別です。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/Info.plist
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleStreamViewModel.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/PhotoVideoSaver.swift
- https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy
- https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html
-->

---

<div class="kicker">LIVE DEMO · PUBLIC SAMPLE</div>

# [公開サンプル](https://github.com/HikaruSato/iosdc2026HLSSample)を動かす：生成・保存・再生

<SampleDataFlow />

<div class="bottom-claim">撮影は1系統。配信用HLSと保存用MP4を同時に生成する</div>

<!--
[Timing checkpoint: 05:00]

ここまで確認したHLSの仕組みを、公開サンプルで実際に動かします。
図は上から読みます。CameraとMicのデータをCaptureSessionからCMSampleBufferとして受け取り、同じ撮影データを2つのWriterへ渡します。
青は配信経路です。Writerがinit.mp4とm4sを生成し、HLSStreamPublisherがinitを先に、以後はsegmentとplaylistの順にPUTします。Macはファイルを保存して配信するだけで、映像変換はしません。
緑は端末保存の経路です。別のWriterでMP4を並行生成し、停止してファイルが完成したら写真ライブラリへ保存します。MP4はMacへ送信しません。
iPhoneはHLSを生成してMacへHTTP PUTし、MacのViewer（Playerを使う再生画面）は同じファイルをHTTP GETして追従再生します。

Safariでplaylistが更新される間の追従再生を見せ、同じplaylist URLをiOSのAVPlayerへ渡した再生確認にも触れます。
Macにinit.mp4、m4s、playlistが残ることと、停止後も同じURLで再生できることを確認します。
事前に写真への追加権限を許可します。配信停止後は「保存しました」を確認し、写真アプリで端末に残ったMP4を再生します。
配信はH.264・720×1280、保存動画はHEVC（H.265）・1080×1920です。撮影中から別々のWriterで生成しています。
この説明はライブデモ成功時にも省略しません。

会場Wi-Fiが不安定な場合はライブ操作を省略し、次の5枚で同じ流れを説明します。ライブデモが成功した場合は、この5枚を省略します。

[Sources]
- iosdc2026HLSSample/README.md
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/LocalVideoWriter.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/PhotoVideoSaver.swift
-->

---
class: demo-step
---

<div class="kicker">PUBLIC SAMPLE FLOW · 1 / 5</div>

# Server は「PUTされたファイルを残すだけ」

<div class="demo-command-layout">
  <div class="terminal-card">
    <span>Mac</span>
    <code>$ python3 server/server.py</code>
    <code class="muted-line">Viewer: http://localhost:8080</code>
    <code class="muted-line">iPhone server URL: http://192.168.x.x:8080</code>
  </div>
  <div class="demo-contract-list">
    <div><b>PUT</b><span>init.mp4 / m4s / m3u8</span></div>
    <div><b>DISK</b><span>server/data/streams/...</span></div>
    <div><b>GET</b><span>同じファイルをViewerへ返す</span></div>
  </div>
</div>

<!--
[Optional demo walkthrough: ライブデモ成功時は省略]

デモ用サーバーを起動します。
ここには映像変換処理がありません。受け取ったHTTP bodyをファイルとして置き換え、GETで返すだけです。

配信開始の前に接続確認を押します。
HTTPHLSClientがhealth endpointへGETし、HTTP成功（2xx）なら接続済みにします。デモ中にURL誤りを早く見つけるための一手です。

[Sources]
- iosdc2026HLSSample/README.md
- iosdc2026HLSSample/server/server.py
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HTTPHLSClient.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/ContentView.swift
-->

---
class: demo-step
---

<div class="kicker">PUBLIC SAMPLE FLOW · 2 / 5</div>

# 「配信開始」で、iPhoneがHLS生成を始める

<div class="tap-to-bytes">
  <div class="tap-button">● 配信開始</div>
  <i>→</i>
  <div><b>Camera + Mic</b><span>video frame / audio block<br>（CMSampleBuffer）</span></div>
  <i>→</i>
  <div class="hot"><b>AVAssetWriter</b><span>圧縮してHLS用Dataへ分割</span></div>
</div>

<div class="demo-observe-line">
  <span>elapsed <b>0.0 →</b></span>
  <span>segment <b>0 →</b></span>
</div>

<!--
[Optional demo walkthrough: ライブデモ成功時は省略]

配信開始を押します。
Cameraから約1 frame、Micから短いaudio blockずつCMSampleBufferとして届きます。
AVAssetWriterは映像・音声を圧縮し、ファイルURLではなくdelegateからfMP4のDataを返します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleHLSStreamer.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---
class: demo-step
---

<div class="kicker">PUBLIC SAMPLE FLOW · 3 / 5</div>

# 最初に届くのは、1回だけのinit.mp4

<div class="object-arrival">
  <div class="arrival-time"><b>t ≈ 0</b><span>initialization callback</span></div>
  <i>→</i>
  <div class="arrival-object init"><b>init.mp4</b><span>再生を始めるための設定</span></div>
  <i>→</i>
  <div class="arrival-path"><code>PUT /streams/{id}/init.mp4</code></div>
</div>

<div class="bottom-claim">media segmentより先に、再生の前提を保存する</div>

<!--
[Optional demo walkthrough: ライブデモ成功時は省略]

delegateの最初のcallbackはinitializationです。
HLSStreamPublisherはinit.mp4の成功を覚え、これが済むまでmedia segmentを受け付けません。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---
class: demo-step
---

<div class="kicker">PUBLIC SAMPLE FLOW · 4 / 5</div>

# 以後は約2秒ごとに、2つのPUTが続く

<div class="put-sequence">
  <div><span>segment 1</span><b>PUT 000001.m4s</b></div>
  <i>→</i>
  <div class="playlist-step"><span>公開</span><b>PUT playlist.m3u8</b></div>
  <i>→</i>
  <div><span>segment 2</span><b>PUT 000002.m4s</b></div>
  <i>→</i>
  <div class="playlist-step"><span>公開</span><b>PUT playlist.m3u8</b></div>
</div>

<div class="bottom-claim">playlistが指すのは、保存に成功したsegmentだけ</div>

<!--
[Optional demo walkthrough: ライブデモ成功時は省略]

約2秒ごとにmedia segmentができます。
大事なのは、m4sをPUTしてからplaylistを置き換える順番です。Playerが404になる参照を先に公開しません。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---
class: demo-step viewer-demo-step
---

<div class="kicker">PUBLIC SAMPLE FLOW · 5 / 5</div>

# Viewerは最新streamを見つけ、playlistを追いかける

<div class="viewer-poll-flow">
  <div><b>1秒ごと</b><code>GET /api/streams</code></div>
  <i>→</i>
  <div><b>最新を選択</b><code>playlistURL</code></div>
  <i>→</i>
  <div class="viewer-live"><b>LIVE</b><span>Safari / AVPlayer / hls.js</span></div>
</div>

<div class="demo-observe-line">
  <span>配信中 <b>segmentが増える</b></span>
  <span>停止後 <b>写真へMP4を自動保存</b></span>
</div>

<!--
[Optional demo walkthrough: ライブデモ成功時は省略]

MacのViewerは1秒ごとにstream一覧を取得し、更新時刻が最も新しい配信を選びます。
SafariはネイティブHLS、それ以外は同梱したhls.jsで再生します。
同じplaylist URLをiOSのAVPlayerへ渡し、WebとiOSの両方で再生を確認します。
停止後もHLSは同じURLで再生でき、写真ライブラリにはフルHD・HEVCのMP4が自動保存されます。
このMP4は端末で別途生成した動画で、配信用segmentを結合したものではありません。

[Sources]
- iosdc2026HLSSample/server/static/app.js
- Apple: Using AVFoundation to play and persist HTTP live streams
-->

---
class: demo-step
---

<div class="kicker">DEMO RESULT · FILE LOCATIONS</div>

# 配信後は、MacとiPhoneに別々の動画が残る

<div class="d d-pair">
 <div class="d-node blue"><span>Mac · HLSの実出力例</span><b>streamディレクトリ</b><small>init.mp4<br>playlist.m3u8<br>seg/000001.m4s<br>…<br>seg/000018.m4s</small></div>
 <div class="d-node green"><span>iPhone · 停止後に写真保存</span><b>1本のMP4</b><small>1080 × 1920<br>HEVC / AAC<br>保存用Writerで別途生成</small></div>
</div>
<div class="bottom-claim">Macはplaylistから再生。iPhoneは写真アプリで保存動画を再生</div>

<!--
Mac側は既存の実出力stream-20260818-225315-C7A2AB11を例にしています。initが1つ、playlistが1つ、m4sが18個です。
保存先でffplay playlist.m3u8を実行するとHLS一式として再生できます。
今回のSampleではiPhoneに保存用Writerを追加しています。停止後に完成したフルHD・HEVCのMP4を写真へ保存します。これはMacへ送ったsegmentを結合したものではなく、撮影中から独立して生成したファイルです。
[Sources]
- iosdc2026HLSSample/server/data/streams/stream-20260818-225315-C7A2AB11
- iosdc2026HLSSample/ios/iosdc2026HLSSample/LocalVideoWriter.swift
-->

---

<div class="kicker">ONE FRAME'S JOURNEY</div>

# 1 frameは、約2秒のsegmentへ<br>まとめてから届ける

<div class="frame-journey">
  <div class="capture"><span>01 CALLBACK</span><b>1 video frame</b><small>CMSampleBuffer<br>SDK object</small></div>
  <i>→</i>
  <div class="writer"><span>02 WRITER</span><b>約60 frames</b><small>+ audio blocks</small></div>
  <i>→</i>
  <div class="fragment"><span>03 FRAGMENT</span><b>000001.m4s</b><small>約2秒分</small></div>
  <i>→</i>
  <div class="storage"><span>04 STORAGE</span><b>HTTP成功</b><small>2xx · 保存できた</small></div>
  <i>→</i>
  <div class="viewer"><span>05 PUBLISH</span><b>playlistへ追加</b><small>ViewerがGET</small></div>
</div>

<div class="bottom-claim">frame単体は送らない。約2秒分を保存してから、playlistで見えるようにする</div>

<!--
[Timing checkpoint: 10:00]

CMSampleBufferは、Cameraから約1 frame、Micから短いaudio blockずつ届くCore MediaのSDK objectです。
Cameraから届いた1つのvideo CMSampleBufferを、そのまま1ファイルとして送るわけではありません。
Writerへ順次appendし、ほかのvideo frameとaudio blockを約2秒分まとめます。30fpsなら、おおよそ60 frameです。

ほかのframeを参照せず再生を始められるkeyframeの境界でmedia segmentが確定すると、delegateから000001.m4sのDataが届きます。
PublisherはまずそのDataをstorageへ保存し、HTTP成功（2xx）を確認してからplaylistへURIを追加します。
Viewerが更新後のplaylistを取得した時点で、このframeを含むsegmentが再生対象になります。

init.mp4は、この流れより先に1回だけ保存します。

この5段階は後続の章に対応します。
callbackとTaskの境界、CMSampleBufferの時刻、fMP4への分割、storage保存とplaylist公開の順に掘り下げます。
-->

---
layout: center
class: chapter
---

<div class="chapter-no">02</div>
<div class="chapter-rule"></div>

# Camera / Micから、<br>fMP4 Dataを取り出す
<p>Camera / Mic → 1 frameずつ受け取る → Writer → HLS用Data</p>

<!--
[Timing checkpoint: 12:00]

ここから公開サンプルのHLSSegmentRecorderへ入ります。
標準SDKのobjectをどう接続すると、CameraとMicからfMP4 Dataを取り出せるのかを順番に見ます。
本番の必須導線は、責務分界、SDK object、Capture callback、Writer / Receiver、segment生成、delegate出力、最小手順です。
通常は詳細ページも含めて説明します。Optional detailは時間超過時の予備であり、省略を前提にはしません。
品質・ビットレートの選択はプロポーザルで予告した設計判断として必ず話します。
-->

---

<div class="kicker">RESPONSIBILITY</div>

# Writerが作るDataを、アプリが保存・公開する

<div class="d">
 <div class="d-row"><div class="d-node"><span>AVFoundation</span><b>AVAssetWriter</b><small>映像・音声を圧縮<br>約2秒に分割</small></div><i class="d-arrow">→</i><div class="d-node blue fill"><span>delegateの出力</span><b>init / media Data</b><small>まだファイル名はない</small></div><i class="d-arrow">→</i><div class="d-node blue"><span>アプリ側</span><b>Publisher</b><small>名前を決めて保存<br>成功後にplaylist公開</small></div></div>
 <div class="d-legend"><span>Writerの担当：HLS用のData生成</span><span>アプリの担当：保存先・HTTP・playlist</span></div>
</div>
<div class="bottom-claim">AVAssetWriterは、playlistを生成しない</div>

<!--
この責任分界が今日の中心です。
AVAssetWriterは映像と音声を圧縮し、HLSで使えるfMP4のDataを返しますが、playlistや保存先のファイル名は作りません。
アプリがDataの保存先を決め、保存成功後にplaylistへ載せます。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriterdelegate
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---

<div class="kicker">HLS PATH · SIX SDK OBJECTS</div>

# HLS生成は、6つのSDK objectを接続する

<div class="recorder-object-map">
  <div class="recorder-object-group capture">
    <span>CAPTURE</span>
    <b>CaptureSession</b>
    <div><code>VideoData<wbr>Output</code><code>AudioData<wbr>Output</code></div>
  </div>
  <div class="recorder-object-arrow">
    <span>delegate</span><i>→</i><small>撮影データ</small>
  </div>
  <div class="recorder-object-group writer">
    <span>WRITE</span>
    <div><code>videoReceiver</code><code>audioReceiver</code></div>
    <b>AVAssetWriter</b>
  </div>
  <div class="recorder-object-arrow">
    <span>delegate</span><i>→</i><small>HLS Data</small>
  </div>
  <div class="recorder-object-output">
    <span>RECORDER OUTPUT</span>
    <b>HLSFragment</b>
    <small>init / media</small>
  </div>
</div>

<div class="bottom-claim">撮影データを、HLSとして保存できるDataへ変える</div>

<!--
HLSSegmentRecorderは、CameraやMicを直接エンコードする巨大なAPIではありません。
Capture側の3 objectとWriter側の3 objectを、2種類のdelegate callbackでつないでいます。
ここではHLS側の接続に絞っています。保存用のLocalVideoWriterは同じCapture callbackから別のWriterへ入力します。

左ではCaptureSessionがDataOutputへCMSampleBufferを流します。
図中のVideoDataOutputとAudioDataOutputは、AVCaptureVideoDataOutputとAVCaptureAudioDataOutputの短縮表記です。
中央ではReceiverからWriterへそのbufferを渡します。
右ではWriter delegateからfMP4のDataを受け取ります。
-->

---

<div class="kicker">CAPTURE TOPOLOGY</div>

# CaptureSessionが、<br>Camera / MicをDataOutputへつなぐ

<div class="capture-topology">
  <div class="device-column">
    <div class="device-node"><b>Back camera</b><span>AVCaptureDeviceInput</span></div>
    <div class="device-node"><b>Microphone</b><span>AVCaptureDeviceInput</span></div>
  </div>
  <div class="session-bus">
    <span>AVCaptureSession</span>
    <i></i>
  </div>
  <div class="output-column">
    <div class="output-node"><b>VideoDataOutput</b><span>video CMSampleBuffer</span></div>
    <div class="output-node"><b>AudioDataOutput</b><span>audio CMSampleBuffer</span></div>
  </div>
</div>

<div class="bottom-claim">完成した録画ファイルではなく、撮影中のframe / audio blockを受け取る</div>

<!--
CaptureSessionへ背面CameraとMicをDeviceInputとして追加し、出口にはVideoDataOutputとAudioDataOutputを追加します。

カメラ構成前には、録音権限の許可に加えてAVAudioSessionのcategoryとmodeを設定します。
Sampleでは次の設定です。これはマイク権限を要求するAPIではなく、録音時の振る舞いの設定です。

```swift
let audio = AVAudioSession.sharedInstance()
try audio.setCategory(.playAndRecord, mode: .videoRecording,
                      options: [.defaultToSpeaker])
try audio.setActive(true)
```

MomentNowではさらに.allowBluetoothHFPを指定し、Bluetooth HFPの入力routeも許可しています。

MovieFileOutputなら完成した録画ファイルを受け取れますが、ライブ中にWriterへ少しずつ渡せません。
DataOutputを使うことで、videoは約1 frame、audioは短いblockごとのCMSampleBufferを撮影中から受け取れます。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">ONE CAPTURE · THREE USES</div>

# 1つのCaptureSessionを、3つの用途で使う

<div class="d d-branch">
 <div class="origin"><span>Camera ＋ Mic</span><b>CaptureSession</b></div>
 <i>→</i><div class="d-node"><b>PreviewLayer</b><small>画面表示：Cameraの映像を表示</small></div>
 <i>→</i><div class="d-node blue"><b>HLS用Writer</b><small>ライブ配信：短いfMP4 Dataを生成</small></div>
 <i>→</i><div class="d-node green"><b>保存用Writer</b><small>端末保存：1本のMP4を生成</small></div>
</div>
<div class="bottom-claim">Sample・MomentNow共通。撮影データを2つのWriterへ入力する</div>

<!--
まず用途を分けます。PreviewLayerには同じCaptureSessionを接続し、カメラ映像を画面へ表示します。
DataOutputから届いた映像・音声は、HLS用Writerと保存用Writerへ渡します。PreviewがWriterの生成物を再生しているわけではありません。
次のページで2つのWriterの出力設定と保存先を比べます。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/CameraPreviewView.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/LocalVideoWriter.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveStreamView.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveSegmentRecorder.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveStreamViewModel.swift
-->

---

<div class="kicker">TWO WRITERS · TWO DESTINATIONS</div>

# 配信はH.264、端末保存はフルHD・HEVC

<div class="d d-pair">
 <div class="d-node blue fill"><span>Sample · HLS用Writer</span><b>H.264 / AAC</b><small>720 × 1280<br>映像 1.5 Mbps</small><div class="d-arrow down">↓</div><b>init ＋ m4s</b><small>撮影中からMacへPUT</small></div>
 <div class="d-node green fill"><span>Sample · 保存用Writer</span><b>HEVC / AAC</b><small>1080 × 1920<br>映像 5 Mbps</small><div class="d-arrow down">↓</div><b>完成MP4</b><small>停止後に写真へ自動保存</small></div>
</div>
<div class="bottom-claim">MomentNowは元動画upload ＋ 設定に応じた写真保存も行う</div>

<!--
Sampleの保存音声はAAC 96 kbps・44.1 kHz・monoです。SampleはMP4をサーバーへ送信しません。
MomentNowは同じく2つのWriterを使い、元動画として別途uploadし、設定が有効なら写真へも保存します。
同じ撮影データから別々に生成しているため、配信用segmentを結合して高品質化したものではありません。
[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/LocalVideoWriter.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveSegmentRecorder.swift
-->

---

<div class="kicker">WHY · STREAM AND LOCAL RECORDING</div>

# 配信用と保存用で、優先するものを変える

<div class="d d-pair">
 <div class="d-node blue"><span>配信：H.264・720p</span><b>ブラウザーでの再生互換性</b><small>HEVCより幅広い再生環境を優先<br>解像度・bitrateを抑えて送信</small></div>
 <div class="d-node green"><span>保存：HEVC・1080p</span><b>画質と端末容量の両立</b><small>配信より高い解像度を保持<br>HEVCの圧縮効率を利用</small></div>
</div>
<p class="d-note">数値は初期設定のままのものが多い。実ユーザーの声による最適化はこれから</p>
<div class="bottom-claim">HLSでもHEVCは使える。今回は配信先との互換性でH.264を選択</div>

<!--
記事で明示している採用理由は、配信用H.264がブラウザー互換性、fMP4がAVFoundationとの相性とSafari / iOSでの再生、別の端末保存が通信・upload失敗への備えです。
ここから設定の意味を説明しますが、すべての値を比較検証して決めたわけではありません。
MomentNowは想定ほど利用が広がらず、実ユーザーの声を集めて設定の見直しにつなげるところまで十分にできていません。一度決めた値のままになっている設定が多い、というのが現状です。
方式の性質や一般的なトレードオフと、私が実際に検証して選んだ理由を分けて話します。
HLSそのものがHEVCを扱えないわけではありません。
HEVCは同等の見た目の品質でH.264より高い圧縮効率を得られる方式です。ただし、このアプリの5 Mbpsで特定の画質や削減率を保証するものではありません。
720×1280より1080×1920の方が画素数は2.25倍です。保存側は配信側より多くの細部を残せますが、実際の画質はcodec・bitrate・撮影条件にも依存します。
5 Mbpsという数値自体はHLSやHEVCの規格が要求する値ではなく、現行実装の選択です。比較実験で得た最適値とは説明しません。
SampleのMP4はMacへ送信しないので、その5 MbpsをHLSの上り帯域に足す必要はありません。MomentNowの元動画uploadは別経路です。

[Sources]
- https://zenn.dev/hs7/articles/080eac650f65ba （HLSの映像と音声仕様、なぜfMP4を使ったか、端末への動画保存）
- https://support.apple.com/en-la/116944
- https://developer.apple.com/documentation/http-live-streaming/hls-authoring-specification-for-apple-devices
- iosdc2026HLSSample/ios/iosdc2026HLSSample/LocalVideoWriter.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveSegmentRecorder.swift
-->

---

<div class="kicker">DATA OUTPUT · SERIAL QUEUE</div>

# 映像と音声のcallbackを、1本のqueueで順に処理

<div class="d">
 <div class="d-row"><div class="d-node blue"><b>VideoDataOutput</b><small>映像1frameずつ</small></div><div class="d-node"><b>AudioDataOutput</b><small>短い音声blockずつ</small></div></div>
 <div class="d-arrow down">↓</div><div class="d-label">共通のserial writingQueue　時間 →</div>
 <div class="d-queue"><span>video 1</span><i>→</i><span>audio 1</span><i>→</i><span>video 2</span><i>→</i><span>audio 2</span></div>
 <div class="d-caption">各callback内で、Writer開始・時刻処理・入力を順に行う</div>
</div>
<div class="bottom-claim">Writerの状態を同時に更新しない。映像と音声の時刻はPTSで扱う</div>

<!--
図はcallbackを直列に処理する例です。videoとaudioが必ず交互に届くわけではありません。
serial queueを共有することで、Writerの開始やappendなどの状態更新を1か所で行います。
queue上の到着順と、映像・音声を提示する時刻であるPTSは別です。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avcapturevideodataoutput/setsamplebufferdelegate(_:queue:)
- https://developer.apple.com/documentation/avfoundation/avcaptureaudiodataoutput/setsamplebufferdelegate(_:queue:)
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">DATA OUTPUT · DELEGATE CODE</div>

# 2つのDataOutputに、同じqueueを指定する


```swift
videoOutput.setSampleBufferDelegate(
    self, queue: writingQueue)
audioOutput.setSampleBufferDelegate(
    self, queue: writingQueue)
```
<div class="d d-row"><div class="d-node"><span>delegate</span><b>self</b><small>captureOutputで受信</small></div><i class="d-arrow">→</i><div class="d-node blue"><span>callbackの実行先</span><b>writingQueue</b><small>同じserial queue</small></div></div>
<div class="bottom-claim">設定は準備時に1回。撮影中は同じcallbackが繰り返し呼ばれる</div>

<!--
VideoとAudioのDataOutputへ同じdelegateとqueueを指定します。
Video用delegate queueにはserial queueが必要です。SampleはAudioも同じqueueへ渡しています。
[Sources]
- https://developer.apple.com/documentation/avfoundation/avcapturevideodataoutput/setsamplebufferdelegate(_:queue:)
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">ORIENTATION</div>

# video connectionへ90度を指定

<div class="orientation-visual">
  <div class="landscape-frame">1920 × 1080</div>
  <div class="rotate-arrow">↻ <span>90°</span></div>
  <div class="portrait-frame">1080 × 1920</div>
</div>

```swift
// 端末側の撮影を縦固定にしているため 90°回転する
if let connection = videoOutput.connection(with: .video),
   connection.isVideoRotationAngleSupported(90) {
    connection.videoRotationAngle = 90
}
```

<!--
[Optional detail: 時間が厳しい場合は省略]

出力サイズはportraitで指定し、video connectionへ90度のrotation angleを設定します。
Sampleはhd1920x1080で撮影し、保存用Writerは1080×1920、配信用Writerは720×1280へ圧縮します。
図はCaptureの出力サイズです。実装では90度の回転を設定できない場合、開始エラーにします。
配信中の向き変更は扱わず、portrait 90度で固定します。
-->

---

<div class="kicker">CONFIG · MOMENTNOW</div>

# 配信開始前に、上り回線に合う単一品質を選ぶ

<div class="d">
 <div class="d-row"><div class="d-node"><span>NWPath</span><b>回線の状態</b></div><div class="d-node"><span>API応答</span><b>準備の待ち時間</b></div><div class="d-node"><span>test upload</span><b>上り速度</b></div></div>
 <div class="d-arrow down">↓</div><div class="d-label">MomentNow：開始前に1つ選択</div>
 <div class="d-row"><div class="d-node blue"><span>Low · 480 × 854</span><b>0.9 Mbps</b></div><div class="d-node blue"><span>Medium · 720 × 1280</span><b>1.5 Mbps</b></div><div class="d-node blue"><span>High · 720 × 1280</span><b>2.5 Mbps</b></div></div>
 <div class="d-caption">Sampleは720 × 1280・1.5 Mbps固定<br>値を上げるほど、画質に使えるデータ量と通信量が増える</div>
</div>
<div class="bottom-claim">開始前に選んだ単一品質で配信。複数品質を公開しないため、視聴側のABRは非対応</div>

<!--
[本編必須: 品質・ビットレートの選択]

個人アプリではNWPathと小さなprobe PUTから、high、medium、lowの1品質を配信開始前に選びます。
途中でrenditionを切り替えるABRではなく、端末の上り回線に合わせた開始時の選択です。
単一品質のHLSだけを公開するため、Playerには品質の切替先がありません。端末内HLS生成そのものがABRを不可能にするという意味ではありません。
配信端末が途中からencode bitrateを変える制御も、現在の実装にはありません。
bitrateを上げるほど映像の細部に使えるデータ量は増えますが、上り回線と視聴側の回線にも余裕が必要です。ここではbyte数への換算はせず、この関係だけを補足します。
0.9 / 1.5 / 2.5 Mbpsは現行アプリの3段階であり、一般的な推奨値や実ユーザーの評価に基づく最適値ではありません。
AVVideoAverageBitRateKeyは圧縮に使う平均bitrateの設定です。各segmentのサイズや瞬間的な上限を保証する値ではなく、実際の送信には音声・コンテナ・HTTP等の分も加わります。
記事の約2.7 Mbps（映像2771 kb/s）と93 kbps（音声）はffprobeによる当時の出力例で、現行コードの設定値2.5 Mbps / 96 kbpsとは区別します。
記事の30 fpsも出力例です。現在のSampleとMomentNowにはactiveVideoMinFrameDuration / activeVideoMaxFrameDurationで30 fpsに固定する指定はありません。

[Sources]
- https://developer.apple.com/documentation/http-live-streaming/creating-a-multivariant-playlist
- https://developer.apple.com/documentation/avfoundation/avvideoaveragebitratekey
- https://zenn.dev/hs7/articles/080eac650f65ba （ffprobeの出力例）
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveSegmentRecorder.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveHLSStreamer.swift
-->

---

<div class="kicker">INPUT FORMAT → VIDEO ENCODE</div>

# 未圧縮映像を、WriterでH.264へ圧縮

<div class="format-bridge">
  <div class="format-node raw"><span>Capture</span><b>未圧縮映像</b><small>video frame</small></div>
  <div class="format-arrow">append</div>
  <div class="format-node encoded"><span>Writer</span><b>H.264 High</b><small>圧縮された映像</small></div>
</div>

<div class="video-profile-note"><b>High</b><span>H.264の圧縮profile名</span></div>

```swift
videoOutput.videoSettings = [
    kCVPixelBufferPixelFormatTypeKey as String:
        kCVPixelFormatType_420YpCbCr8BiPlanarFullRange
]
```

<div class="bottom-claim compact">サンプルは1.5 Mbps · 個人アプリは上り回線に合わせて0.9-2.5 Mbps</div>

<div class="source">HLSSegmentRecorder.setupCaptureSessionLocked()</div>

<!--
[Optional detail: 時間が厳しい場合は省略]

DataOutputからは未圧縮映像のpixel bufferを受け取ります。
H.264への圧縮はAVAssetWriterInputのoutputSettingsが担当します。
HighはApple SDKで指定できるH.264 profileの名前です。
公開サンプルは1.5 Mbps固定です。個人アプリは上り回線に応じて、品質設定ごとに0.9、1.5、2.5 Mbpsから選びます。段階ごとの数値を、実ユーザーの評価に基づく最適値としては示しません。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveSegmentRecorder.swift
-->

---

<div class="kicker">WHY · PIXEL FORMAT</div>

# 入力形式は、色変換とメモリー転送量に効く

<div class="rationale-rows">
 <div><b>Y′CbCr 4:2:0</b><strong>輝度と色差</strong><span>BGRAより小さい画素表現</span></div>
 <div><b>8-bit・BiPlanar</b><strong>8 bit・2つの面</strong><span>輝度と色差を分けて格納</span></div>
 <div><b>FullRange</b><strong>輝度：0〜255</strong><span>画素値の範囲を指定</span></div>
</div>
<p class="d-note">Captureの対応形式を使い、BGRAへの不要な変換を避けるのが基本</p>
<div class="bottom-claim">FullRangeは今回の選択。HLSが要求する入力形式ではない</div>

<!--
長い定数名を分けて読むページです。4:2:0は輝度より色差のサンプル数を少なくする表現、8は各成分8 bit、BiPlanarは輝度面と色差面の2-planeを表します。
Apple TN3121は、BGRAを無条件に選ぶとネイティブ形式からの変換とメモリー帯域の負担が生じると説明しています。8-bit 4:2:0は概ね1.5 byte/pixel、BGRAは4 byte/pixelです。
どの形式がネイティブかは端末のactiveFormatに依存します。一般にはavailableVideoPixelFormatTypesを確認して、後段の処理が扱える形式を選びます。Sampleの現行コードはFullRangeを固定指定しており、形式を動的に選び直す実装ではありません。
FullRangeは8-bit輝度に0〜255の範囲を使う指定です。解像度、HDR、H.264 High profile、色域が広いという意味ではありません。
FullRangeをVideoRangeより常に高画質・高速と説明したり、すべての端末で変換が不要と断定したりはしません。
記事のffprobeに出てくるyuvj420pや色のタグは、符号化後の出力例です。DataOutputの2-plane bufferのメモリ配置と同じものとして扱いません。

[Sources]
- https://developer.apple.com/documentation/technotes/tn3121-selecting-a-pixel-format-for-an-avcapturevideodataoutput
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">VIDEO SETTINGS</div>

# 配信用videoフォーマットは、互換性と上り帯域を優先する

<div class="settings-table">
  <div class="settings-head"><span>KEY</span><span>VALUE</span><span>INTENT</span></div>
  <div><code>AVVideoCodecKey</code><b>H.264</b><span>広い再生互換性</span></div>
  <div><code>Width / Height</code><b>480p / 720p</b><span>細部と送信量のバランス</span></div>
  <div><code>AverageBitRate</code><b>0.9–2.5 Mbps</b><span>開始前に上り帯域で選ぶ</span></div>
  <div><code>ProfileLevel</code><b>High Auto</b><span>圧縮profile・level</span></div>
  <div><code>FrameReordering</code><b>false</b><span>表示順とdecode順を揃える</span></div>
</div>

<div class="source">LiveSegmentRecorder.makeHLSVideoSettings()</div>

<!--
[Optional detail: 時間が厳しい場合は省略]

個人アプリはH.264 portraitで、品質に応じて480pまたは720p、0.9から2.5Mbpsを選びます。
端末保存品質ではなく、ネットワークへ継続的に送れる配信品質として設定します。
解像度を下げると表現できる細部は減りますが、低いbitrateで画質を保ちやすくなります。同じ720pでも動きや細部によって必要なbitrateは変わるため、解像度だけで値を決めません。
High Autoとframe reorderingは次のページで意味を分けます。
-->

---

<div class="kicker">WHY · PROFILE AND FRAME ORDER</div>

# 圧縮profileと、frame順序を選ぶ理由

<div class="d d-pair">
 <div class="d-node blue"><span>H.264 High Auto</span><b>圧縮方式と対応範囲を設定</b><small>High：Appleが推奨するprofile<br>Auto：Levelの選択をencoderへ</small></div>
 <div class="d-node blue"><span>FrameReordering = false</span><b>表示順とdecode順を揃える</b><small>frameの並べ替えを禁止<br>B-frameの圧縮効率と引き換え</small></div>
</div>
<p class="d-note">falseにすると、撮影中のframeを表示順のまま扱える</p>
<div class="bottom-claim">AutoはLevelの選択。ビットレートの自動切替（ABR）ではない</div>

<!--
H.264 Highは符号化に使える機能のprofile名です。アプリのHigh / Medium / Lowという画質段階とは別の分類です。
AppleのHLS Authoring SpecificationはMainやBaselineよりHigh profileを推奨しています。ただし、再生互換性はprofileだけでなくlevelや再生機器にも依存します。
Levelは解像度や処理量などの対応範囲を示します。HighAutoLevelでは特定のlevel番号を固定せず、encoderに選択を委ねます。出力がどの機器でも再生できる保証ではありません。
frame reorderingは、表示順とdecode順を変える処理です。B-frameをencodeする際には並べ替えが必要になるため、falseはその並べ替えを禁止します。
一般的なトレードオフは、B-frameで得られる圧縮効率を使わず、時刻やframe順の扱いを簡単にすることです。falseがHLSの必須条件だったり、配信遅延全体をなくしたりするわけではありません。
SampleとMomentNowは配信用・保存用の両方でfalseを指定しています。保存用HEVC Main Autoも、8-bit入力に対応したprofileとlevel自動選択という構成です。

[Sources]
- https://developer.apple.com/documentation/http-live-streaming/hls-authoring-specification-for-apple-devices （1.4）
- https://developer.apple.com/documentation/avfoundation/avvideoprofilelevelh264highautolevel
- https://developer.apple.com/documentation/videotoolbox/kvtcompressionpropertykey_allowframereordering
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/LocalVideoWriter.swift
-->

---

<div class="kicker">AUDIO SETTINGS</div>

# AudioはAAC mono、品質別に48–96kbps

<div class="audio-spec">
  <div><span>FORMAT</span><b>AAC</b></div>
  <div><span>CHANNEL</span><b>mono</b></div>
  <div><span>SAMPLE RATE</span><b>44.1 kHz</b></div>
  <div><span>BITRATE</span><b>48–96 kbps</b></div>
</div>

```swift
[
    AVFormatIDKey: kAudioFormatMPEG4AAC,
    AVNumberOfChannelsKey: 1,
    AVSampleRateKey: 44_100,
    AVEncoderBitRateKey: config.audioBitrate
]
```

<!--
[Optional detail: 時間が厳しい場合は省略]

音声はAAC、mono、44.1kHzで、low 48、medium 64、high 96kbpsです。
monoと各bitrateは現行実装の設定です。利用者の音質評価を比較して、この組み合わせが適切だと確認したわけではありません。
-->

---

<div class="kicker">WHY · AUDIO SETTINGS</div>

# 音声設定の意味と、選択によるトレードオフ

<div class="rationale-rows audio-rationale">
 <div><b>AAC</b><strong>音声の圧縮方式</strong><span>HLSで扱える一般的な方式</span></div>
 <div><b>mono：1 ch</b><strong>音声を1系統に</strong><span>左右の空間表現を省く</span></div>
 <div><b>44.1 kHz</b><strong>44,100 sample / 秒</strong><span>一般的な選択。48 kHzも使える</span></div>
 <div><b>48 / 64 / 96 kbps</b><strong>音声のデータ量</strong><span>映像品質に合わせて段階を用意</span></div>
</div>
<div class="bottom-claim">monoは今回の選択。Appleの配信仕様はstereoを要求する点に注意</div>

<!--
AACはHLSで利用できる音声codecです。記事の出力例ではAAC-LCが確認できています。
monoは左右の空間表現を持たない1ch音声です。少ないbitrateで音声を扱う構成ですが、monoにしただけで指定済みbitrateの送信量が自動的に半分になるわけではありません。
Appleの現行HLS Authoring Specification 2.3はstereo音声の提供を要求しています。今回のmonoをApple推奨設定や配信仕様への全面準拠として紹介しません。一般向けに仕様準拠を目指す場合はstereoの提供を検討します。
44.1 kHzと48 kHzはAACで一般的なsample rateです。44.1 kHzはHLS固有の必須値ではなく、現行実装の設定です。音声入力と異なるrateなら変換が必要になり得るため、実入力も確認します。
低いbitrateは送信量を抑える一方、音質には不利になり得ます。48 / 64 / 96 kbpsの各値は規格上の正解ではありません。
MomentNowはLow 48、Medium 64、High 96 kbps。SampleのHLSは64 kbps固定、保存用MP4は96 kbpsです。
音声の細部をどの程度残したいかで値を検証するもので、会話・音楽・騒音下など全条件で同じ品質になるとは説明しません。
これらは設定の一般的な性質です。MomentNowでは実ユーザーの声に基づく設定の見直しが十分できておらず、44.1 kHzやbitrateの数値を比較検証の結論としては示しません。

[Sources]
- https://developer.apple.com/documentation/http-live-streaming/preparing-audio-for-http-live-streaming
- https://developer.apple.com/documentation/http-live-streaming/hls-authoring-specification-for-apple-devices （2.3）
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/LocalVideoWriter.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveSegmentRecorder.swift
-->

---

<div class="kicker">HLS WRITER · iOS 26+</div>

# URLを持たないWriterから、Dataを受け取る

<div class="d d-row d-compact"><div class="d-node blue"><span>HLS向け設定</span><b>AVAssetWriter</b></div><i class="d-arrow">→</i><div class="d-node blue"><span>delegate</span><b>init / media Data</b></div></div>

```swift
let writer = AVAssetWriter(contentType: .mpeg4Movie)
writer.outputFileTypeProfile = .mpeg4AppleHLS
writer.preferredOutputSegmentInterval = .init(
    seconds: 2, preferredTimescale: 600)
writer.initialSegmentStartTime = startTimeOffset
writer.delegate = self
```

<div class="bottom-claim">600は時刻の分母。2秒 ＝ 1200 / 600秒。fpsの指定ではない</div>

<!--
このサンプルのWriterコードはiOS 26以上が対象です。
通常の録画では出力先URLを指定しますが、segment delegateを使う構成ではcontentTypeだけでWriterを作ります。
Apple HLS profile、希望segment間隔、initial start time、delegateを設定します。
mpeg4AppleHLSは、HLS向けのfMP4としてsegment Dataを受け取るための出力profileです。codecのH.264 High profileとは別の設定です。
CMTimeはvalue / timescale秒という有理数です。600は24・25・30の倍数で、それらのfpsのframe時刻を表しやすい値としてAppleの説明でも使われます。
ただし、このコードで600を使う対象はsegment間隔の2秒です。2秒の表現に600が必須なわけではなく、1200 / 600も2 / 1も2秒です。
撮影fpsや音声sample rateを600へ変更する設定ではなく、すべての入力timestampを600に丸める指定でもありません。29.97 fpsなどを常に正確に表すという説明もしません。

contentTypeのmpeg4MovieはMP4というcontainerの種類、outputFileTypeProfileのmpeg4AppleHLSはHLS向けfragment出力の指定です。
名前は似ていますが競合する設定ではありません。

このdelegate methodを実装すると通常のファイル書き込みは抑止され、Writerがsegment Dataをcallbackします。
各propertyの意味と2秒境界はChapter 03で詳しく見ます。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriter
- https://developer.apple.com/documentation/avfoundation/avfiletypeprofile/mpeg4applehls
- https://developer.apple.com/documentation/avfoundation/writing-fragmented-mpeg-4-files-for-http-live-streaming
- https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/06_MediaRepresentations.html
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">INPUT · RECEIVER · WRITER</div>

# Inputは圧縮設定、Receiverはbufferの入口

<div class="d">
 <div class="d-row"><div class="d-node"><span>① 準備時に設定</span><b>Input</b><small>mediaType<br>outputSettings</small></div><i class="d-arrow">→</i><div class="d-node blue"><span>② Writerへ接続</span><b>Receiver</b><small>入力窓口を取得して保持</small></div><i class="d-arrow">→</i><div class="d-node blue"><span>③ 生成を担当</span><b>Writer</b><small>映像・音声を圧縮<br>fMP4 Dataを出力</small></div></div>
 <div class="d-row"><div class="d-node fill"><span>撮影中の繰り返し</span><b>CMSampleBufferをReceiverへ入力する</b><small>videoReceiver / audioReceiverを使い分ける</small></div></div>
</div>
<div class="bottom-claim">Inputは設定。撮影データを書き込む窓口はReceiver</div>

<!--
Input、Receiver、Writerの関係を先に確認します。Inputへ圧縮設定を渡し、inputReceiver(for:)でWriterへ接続します。
返されたReceiverを保持し、以後のcallbackごとにCMSampleBufferを入力します。
Inputをデータが順番に通過する処理ステップとしてではなく、接続する設定として捉えます。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriter
- https://developer.apple.com/documentation/avfoundation/avassetwriterinput/samplebufferreceiver
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">RECEIVER SETUP · iOS 26+</div>

# InputをWriterへ接続し、Receiverを保持する


```swift
let videoInput = AVAssetWriterInput(
    mediaType: .video,
    outputSettings: videoSettings()
)
videoInput.expectsMediaDataInRealTime = true
self.videoReceiver = writer.inputReceiver(for: videoInput)
```
<div class="d d-row"><div class="d-node"><span>setup内で作成</span><b>videoInput</b></div><i class="d-arrow">→</i><div class="d-node blue fill"><span>以後のcallbackで使用</span><b>videoReceiver</b></div></div>
<div class="bottom-claim">撮影から逐次届く入力なので、real-time向けの処理を指定する</div>

<!--
inputReceiver(for:)がInputをWriterへ接続し、書き込むためのReceiverを返します。
従来のwriter.add(input)とinput.append(sampleBuffer)に相当する接続と書き込みをReceiver経由で行う、iOS 26以降のAPIです。
AudioもmediaTypeとoutputSettingsを替え、同じ手順で接続します。
expectsMediaDataInRealTime = trueはCaptureのようなリアルタイム入力に合わせてWriter Inputの処理を調整する指定です。ファイルを高速変換する入力とは違うことを伝えます。
書き込み開始前に設定します。入力を無制限に受け付けたり、常に一定の処理時間を保証したりする指定ではありません。
[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriterinput/expectsmediadatainrealtime
- https://developer.apple.com/documentation/avfoundation/avassetwriterinput/samplebufferreceiver
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">CMSAMPLEBUFFER · SEGMENT</div>

# 小さなbufferをまとめ、約2秒のsegmentにする

<div class="d">
 <div class="d-label">撮影中に届く入力の例</div>
 <div class="d-queue"><span>video</span><span>audio</span><span>video</span><span>audio</span><span>…</span></div>
 <div class="d-row"><div class="d-node"><span>1つのvideo buffer</span><b>約1 frame</b><small>30fpsなら約33ms</small></div><i class="d-arrow">→</i><div class="d-node blue"><span>AVAssetWriter</span><b>圧縮 ＋ 分割</b><small>映像と音声を格納</small></div><i class="d-arrow">→</i><div class="d-node blue fill"><span>1つのmedia Data</span><b>約2秒分</b><small>映像と音声を含む</small></div></div>
</div>
<div class="bottom-claim">init Dataは最初に1回。media Dataは撮影中に繰り返し届く</div>

<!--
CMSampleBufferはsegmentファイルではありません。Videoなら約1 frame分、Audioなら短い音声blockです。

AVAssetWriterが多数のCMSampleBufferをH.264とAACへ圧縮し、映像と音声を時間順に格納して、約2秒分のfragmentへまとめます。
30fpsなら1つのsegmentにおよそ60 frameが入ります。実際の境界はkeyframeにより前後します。

最初に圧縮方式や映像・音声の構成を持つinitialization Dataが届き、その後に映像・音声本体を持つseparable Dataが繰り返し届きます。

[Sources]
- https://developer.apple.com/videos/play/wwdc2020/10011/
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">CALLBACK GATES</div>

# Writerへ渡す前に、2つのguardで確認する

<div class="guard-funnel">
  <div class="guard-input">captureOutput</div>
  <div class="guard-step"><b>配信中か</b><span>isWriting</span></div>
  <div class="guard-step"><b>Dataはreadyか</b><span>利用可能か確認</span></div>
  <div class="guard-output">Writer開始<br>→ append</div>
</div>

```swift
guard isWriting else { return }
guard CMSampleBufferDataIsReady(sampleBuffer) else { return }
```

<!--
[Optional detail: 時間が厳しい場合は省略]

CaptureSessionが動いていても、Writerへ渡すのは配信中だけです。
CMSampleBufferのDataがreadyでない場合も早期returnし、Writerの状態遷移を単純に保ちます。
Writerが受け取れないときにframeを貯めない判断は、Chapter 05で説明します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">CAPTURE CALLBACK · TWO COPIES</div>

# 同じ撮影データをコピーし、2つのWriterへ

<div class="d d-branch two">
 <div class="origin"><span>Capture callback</span><b>元の<br>CMSampleBuffer</b></div>
 <i>→</i><div class="d-node blue fill"><span>HLS用のコピー</span><b>時刻を補正</b><small>HLS用Receiverへ入力</small></div>
 <i>→</i><div class="d-node green fill"><span>保存用のコピー</span><b>元の時刻を使用</b><small>保存用Receiverへ入力</small></div>
</div>
<div class="bottom-claim">HLS側が受け取れなくても、保存用Writerへの入力は続ける</div>

<!--
ここは同じcallback内の分岐です。処理が別threadで同時実行されることを表す図ではありません。
HLS側は時刻を補正したコピーを作り、保存側は元の時刻を持つ独立したコピーを作ります。
HLS側のメソッドがdropや失敗でreturnしても、callbackからの保存側appendは続きます。一方の入力失敗で両方をスキップしない構成です。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriterinput/samplebufferreceiver/appendimmediately(_:)
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">APPEND IMMEDIATELY · iOS 26+</div>

# HLS側のReceiverへ、待たずに入力を試す


```swift
let readySampleBuffer = CMReadySampleBuffer(
    unsafeBuffer: transferableSampleBuffer)
let didAppend = try receiver.appendImmediately(
    readySampleBuffer)
guard didAppend else { return }
```
<div class="d d-row"><div class="d-node blue"><span>true</span><b>入力できた</b></div><div class="d-node warn"><span>false</span><b>このbufferをdrop</b></div><div class="d-node warn"><span>throw</span><b>HLS側が失敗</b></div></div>
<div class="bottom-claim">保存用Writerへの入力は別に実行する</div>

<!--
HLS側のappendの抜粋です。時刻補正後のコピーをCMReadySampleBufferへ変換してReceiverに渡します。
appendImmediatelyは同期的に受け入れを試し、falseならbufferを保留せずdropします。throw時はHLSのfragment streamをerrorで閉じますが、正常な保存用Writerは停止操作まで継続します。
[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriterinput/samplebufferreceiver/appendimmediately(_:)
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">WRITER DELEGATE → FILE PATH</div>

# WriterのDataを、Publisherが保存する

<div class="fragment-routing">
  <div class="fragment-route init">
    <code>.initialization</code><i>→</i><b>init Data</b><i>→</i><strong>init.mp4</strong>
  </div>
  <div class="fragment-route media">
    <code>.separable</code><i>→</i><b>media Data</b><i>→</i><strong>seg/000001.m4s</strong>
  </div>
</div>

```swift
switch segmentType {
case .initialization:
    continuation.yield(.initialization(segmentData))
case .separable:
    segmentIndex += 1
    continuation.yield(.media(sequence: segmentIndex,
        data: segmentData, duration: duration))
}
```

<div class="bottom-claim">Recorderの出力はData。HTTPHLSClientが保存先のpathを組み立てる</div>

<!--
Writer delegateには、segmentDataとsegmentTypeが届きます。
initializationは配信ごとに最初の1回、separableは約2秒ごとです。

HLSSegmentRecorderはDataをHLSFragmentへ変換するところまでを担当します。
この時点ではinit.mp4や000001.m4sというローカルファイルは作っていません。

後段のHLSStreamPublisherがfragmentを受け取り、HTTPHLSClientがinitializationをinit.mp4、mediaをsequence付きのm4s pathへPUTします。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriterdelegate/assetwriter(_:didoutputsegmentdata:segmenttype:segmentreport:)
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HTTPHLSClient.swift
-->

---

<div class="kicker">IMPLEMENTATION ORDER · SETUP</div>

# HLS生成の準備：SDK objectを接続する

<div class="d d-row">
 <div class="d-node"><span class="d-number">01</span><b>Capture</b><small>Camera / Micを<br>Sessionへ追加</small></div><i class="d-arrow">→</i>
 <div class="d-node"><span class="d-number">02</span><b>DataOutput</b><small>delegateと<br>serial queue</small></div><i class="d-arrow">→</i>
 <div class="d-node blue"><span class="d-number">03</span><b>Writer</b><small>HLS profile<br>約2秒の設定</small></div><i class="d-arrow">→</i>
 <div class="d-node blue"><span class="d-number">04</span><b>Receiver</b><small>Inputを接続し<br>窓口を保持</small></div>
</div>
<div class="bottom-claim">準備では、入力元・callbackの実行先・生成先をつなぐ</div>

<!--
HLS生成の8ステップのうち、最初の4つです。ここは準備時の接続を復習します。
まだfragmentを送信している段階ではありません。次に、撮影データが届いた後の開始と繰り返しを見ます。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">IMPLEMENTATION ORDER · RUN / STOP</div>

# HLS生成の実行：開始・入力・受信・終了

<div class="d d-row">
 <div class="d-node"><span class="d-number">05</span><b>開始</b><small>最初のVideoで<br>startSession</small></div><i class="d-arrow">→</i>
 <div class="d-node blue"><span class="d-number">06</span><b>入力</b><small>時刻を補正<br>Receiverへappend</small></div><i class="d-arrow">→</i>
 <div class="d-node blue"><span class="d-number">07</span><b>Dataを受信</b><small>init：1回<br>media：反復</small></div><i class="d-arrow">→</i>
 <div class="d-node"><span class="d-number">08</span><b>終了</b><small>finishWritingで<br>最後のData</small></div>
</div>
<div class="bottom-claim">05・06の時刻処理は、次の1枚で確認する</div>

<!--
最初のvideoでsessionを開始し、各callbackで時刻を補正したbufferをappendします。入力とData出力は撮影中に繰り返されます。
停止時はfinishWritingの完了まで待ちます。次の数値例で、入力する時刻の調整だけを短く説明します。
[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---
class: timing-overview
---

<div class="kicker">WRITER TIMING</div>

# 映像と音声の時刻を、同じ量だけ移動する

<p class="d-note">HLS側はApple公式サンプルの方針を採用</p>
<div class="d d-time-pair">
 <div><div class="d-label">調整前</div><div class="d-time-track"><div>100.00<small>映像</small></div><div>100.02<small>音声</small></div></div><div class="d-time-gap">時間差 0.02秒</div></div>
 <div class="d-time-shift">−90秒<br>→</div>
 <div><div class="d-label">Writerへ渡す時刻</div><div class="d-time-track"><div>10.00<small>映像</small></div><div>10.02<small>音声</small></div></div><div class="d-time-gap">時間差 0.02秒</div></div>
</div>
<div class="bottom-claim warning">検証結果：時刻調整を外すと、映像・音声が入らなかった</div>
<p class="d-note"><b>検証条件：Writerの開始位置は10秒のまま、時刻調整だけを外した場合</b></p>

<!--
[Timing checkpoint: 20:00]

時刻の扱いは、Apple公式サンプルの方針を参考にしています。
この実装では、最初の映像を基準に、Writer用の開始位置へ時刻をずらします。
例えば映像100.00秒、音声100.02秒なら、両方から90秒を引いて10.00秒と10.02秒にします。
映像と音声には同じ調整を適用して、元の時間差を保ちます。

同じCaptureSessionから届く映像と音声は、もともと共通の時計上にあります。
別々の時計を同期させたり、発生した音ズレを修復したりする処理ではありません。
Appleが示す一括の時刻移動を採用し、Capture入力では最初のvideo PTSから移動量を決めています。
10秒はAACのprimingを扱うために選んだ余白です。再生開始まで10秒待つ意味はありません。
ここは省略すると動かなくなった点でもあります。私の検証では、Writerの開始位置を10秒のままにして時刻調整だけを外すと、生成ファイルに映像・音声が入りませんでした。
具体的にはstartSessionとinitialSegmentStartTimeを10秒のまま、元のCapture timestampのCMSampleBufferをappendした場合の結果です。
今回の設定で経験した症状として紹介します。開始位置も変更した別構成や、AVAssetWriter全般で時刻調整が必須だと断定するものではありません。
initialization segmentに映像・音声の本体が入らない通常の仕様とは区別します。
本編は約1分でこの例を説明し、10秒の背景やPTS・DTSのコードは補足へ回します。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avcapturesession/synchronizationclock
- https://developer.apple.com/videos/play/wwdc2020/10011/?time=976
- https://developer.apple.com/documentation/avfoundation/writing-fragmented-mpeg-4-files-for-http-live-streaming
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- 発表者による時刻調整無効化の検証結果（Writer開始位置とinitialSegmentStartTimeは10秒のまま）
-->

---
layout: center
class: chapter
---

<div class="chapter-no">03</div>
<div class="chapter-rule"></div>

# 映像と音声を、<br>約2秒のfMP4へ分ける
<p>Four HLS settings and segment boundaries</p>

<!--
[Timing checkpoint: 21:00]

Writerへ渡す時刻の扱いを確認したので、次は約2秒のfragment境界を作るHLS固有設定を見ます。
-->

---

<div class="kicker">HLS WRITER · FOUR SETTINGS</div>

# 4つの設定が、出力形式と境界・受け口を決める

<div class="d d-link-map">
 <div class="d-line">outputFileTypeProfile</div><i>→</i><div class="d-file"><b>Writerの出力形式</b><small>Apple HLS向けfMP4</small></div>
 <div class="d-line">preferredOutputSegmentInterval</div><i>→</i><div class="d-file"><b>segmentの間隔</b><small>約2秒ごとの境界</small></div>
 <div class="d-line">initialSegmentStartTime</div><i>→</i><div class="d-file"><b>最初の開始位置</b><small>時刻補正に合わせる</small></div>
 <div class="d-line">delegate</div><i>→</i><div class="d-file"><b>Dataの受け口</b><small>init / mediaを受信</small></div>
</div>

<!--
この4つが、ファイルURLを持たないHLS用Writerの核です。
特にintervalは「必ず」ではなく「preferred」です。
outputFileTypeProfileはHLS向けfMP4の出力を選ぶ設定で、playlistを作る指定ではありません。記事で選んだfMP4は、AVAssetWriterDelegateから直接受け取りSafari / iOSへ配信する今回の構成に合います。
2秒のintervalは後の比較図、10秒のinitialSegmentStartTimeは本編の時刻図と補足で理由を説明します。

[Sources]
- https://zenn.dev/hs7/articles/080eac650f65ba
- https://developer.apple.com/documentation/avfoundation/writing-fragmented-mpeg-4-files-for-http-live-streaming
-->

---

<div class="kicker">SEGMENT BOUNDARY · SCHEMATIC</div>

# 約2秒付近のIDRを境界に、segmentを作る

<div class="d">
 <div class="d-axis full"><span>0.00秒</span><span>2.03秒</span><span>4.00秒</span></div>
 <div class="d-frames"><span class="idr">IDR</span><span>P</span><span>…</span><span>P</span><span class="idr">IDR</span><span>P</span><span>…</span><span>P</span></div>
 <div class="d-brackets"><span>segment 1：2.03秒</span><span>segment 2：1.97秒</span></div>
 <div class="d-row"><div class="d-node"><span>希望する間隔</span><b>2.00秒</b></div><i class="d-arrow">→</i><div class="d-node blue"><span>実際に生成した長さ</span><b>reportのduration</b></div><i class="d-arrow">→</i><div class="d-node blue fill"><span>playlistへ記載</span><b>EXTINF</b></div></div>
 <div class="d-caption">模式例：2秒ぴったりになるとは限らない</div>
</div>

<!--
今回のSampleは未圧縮入力をWriterでH.264にencodeします。Appleの説明では、encodeモードは希望間隔に達するか超えるvideo sampleをsync sampleにするため、ほぼ希望間隔で出力します。
すでに存在するIDRを長時間待つpassthroughだけの挙動として説明しません。図の2.03秒・1.97秒は実測ではなく境界のずれを示す模式例です。
playlistにはAVAssetSegmentReportのvideo trackの実durationを渡します。Sampleはreport未取得・無効時だけ設定値へfallbackします。
IDRは前のframeを参照せず、そこから再生を開始できるkeyframeです。

[Sources]
- Apple: AVAssetWriter.preferredOutputSegmentInterval
- Apple HLS Authoring Specification for Apple Devices
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLS/HLSManifest.swift
-->

---

<div class="kicker">WHY · SEGMENT DURATION</div>

# segmentを短くすると、早く送れるがPUTが増える

<div class="d d-pair">
 <div class="d-node blue"><span>今回の希望間隔：2秒</span><b>短い断片で順次送信</b><small>1分あたり約30 segment<br>生成完了までの待ちを短くする</small></div>
 <div class="d-node"><span>Appleのサンプル：6秒</span><b>1回にまとめる量を増やす</b><small>1分あたり約10 segment<br>segment取得・PUTの回数が少ない</small></div>
</div>
<p class="d-note">IDRの「約2秒ごと」の推奨と、segment長の目安は別の設定</p>
<div class="bottom-claim">2秒は今回の選択。HLSの必須値でも、視聴遅延の保証でもない</div>

<!--
撮影中にできた分から送りたいので、短いsegmentには生成完了を早く迎えられる利点があります。一方、短くするとHTTP requestやplaylist更新が増えます。
同じbitrateで理想的に2秒 / 6秒ずつ出力する概算なら、1分で30個 / 10個です。SampleはmediaごとにplaylistもPUTするため、2秒の例なら定常時に約60 PUT/分になります。initやretryは別です。
この回数はnominalな概算で、実durationや回線の詰まりで実際の時刻は変わります。2秒にしただけでend-to-endの遅延が2秒になるわけではありません。
Apple HLS Authoring Specificationはsegment / target durationの一般的な目安を6秒、IDRを約2秒ごととしています。2つの推奨を混同しません。
今回の2秒segmentは早く小分けに送りたい構成での選択です。HLSの固定要件や、Low-Latency HLSの実装として説明しません。
保存用MP4のkeyframe間隔2秒も、HLS segmentを生成する設定ではありません。停止まで1本のMP4へ書き込みます。

[Sources]
- https://developer.apple.com/documentation/http-live-streaming/hls-authoring-specification-for-apple-devices （1.13、7.5、7.6）
- https://developer.apple.com/documentation/avfoundation/avassetwriter/preferredoutputsegmentinterval
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---

<div class="kicker">PUBLIC SAMPLE · SEGMENT DURATION</div>

# 実durationの反映：サンプルで実装済み

<div class="duration-compare">
  <div class="duration-side sample">
    <span>公開サンプル · 実装済み</span>
    <b>実durationをEXTINFへ</b>
    <small>reportが無効・未取得の場合だけ2秒</small>
  </div>
  <div class="duration-side production">
    <span>MomentNow · 現状</span>
    <b>2秒固定で送信</b>
    <small>改善案：同じ実durationの取り込みを適用</small>
  </div>
</div>

```swift
let duration = report?.trackReports
    .first { $0.mediaType == .video }?
    .duration.seconds
guard let duration, duration.isFinite, duration > 0 else {
    return config.segmentSeconds
}
return duration
```

<!--
[Optional detail: 時間が厳しい場合は省略]

公開サンプルはAVAssetSegmentReportのvideo track durationをplaylistへ渡します。
reportがない、無効、0以下の場合だけ設定値の2秒へ戻します。frame reorderingは無効なので、このサンプルではvideo reportを採用しています。
現在の個人アプリ実装はまだfragmentSecondsの2.0固定なので、同じ取り込み方を適用できる改善点です。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">KEYFRAME ALIGNMENT</div>

# IDRを用意すると、<br>segmentの先頭から再生できる

<div class="d">
 <div class="d-label">映像frameの依存関係（模式図）</div>
 <div class="d-frames"><span class="idr">IDR</span><span>→ P</span><span>→ P</span><span>→ P</span><span class="idr">IDR</span><span>→ P</span><span>→ P</span><span>→ P</span></div>
 <div class="d-brackets"><span>ここから再生可能 · 約2秒</span><span>ここから再生可能 · 約2秒</span></div>
</div>

```swift
AVVideoMaxKeyFrameIntervalDurationKey: config.segmentSeconds
```

<div class="bottom-claim">Appleは約2秒ごとのIDRを推奨。segmentの希望間隔と合わせる</div>

<!--
IDRの位置で、前のsegmentに依存せず再生を開始できます。Pは前のframeを参照して圧縮するframeの模式表現です。
I-frame全般とIDRを同一視しないため、図ではIDRと明記しています。
Apple HLS Authoring Specificationは約2秒ごとのIDRを推奨します。この設定はsegmentを常に2.000秒固定にする保証ではありません。

[Sources]
- https://developer.apple.com/documentation/http-live-streaming/hls-authoring-specification-for-apple-devices
-->

---

<div class="kicker">GENERATION TIMELINE</div>

# Captureを止めず、約2秒ごとにDataを出す

<div class="lane-timeline upload-lanes">
  <div class="lane"><b>Capture</b><span class="continuous">frame · frame · frame · frame</span></div>
  <div class="lane"><b>Writer</b><span></span><span class="fragment">seg 1</span><span class="fragment">seg 2</span><span class="fragment">seg 3</span></div>
  <div class="lane"><b>Delegate</b><span class="init-mark">init</span><span class="callback-mark">Data 1</span><span class="callback-mark">Data 2</span><span class="callback-mark">Data 3</span></div>
  <div class="lane"><b>保存先</b><span class="init-mark">PUT init</span><span class="callback-mark">PUT 1</span><span class="callback-mark">PUT 2</span><span class="callback-mark">PUT 3</span></div>
</div>

<div class="bottom-claim">Writerのfinishを待たず、生成と保存を並行できる</div>

<!--
通常の録画ファイルと違い、finishWritingまで待ちません。
Captureが続く間にfragmentがdelegateへ届くため、生成と保存を並行できます。
-->

---
layout: center
class: chapter
---

<div class="chapter-no">04</div>
<div class="chapter-rule"></div>

# 保存できたsegmentだけを、<br>playlistへ公開する
<p>Save bytes first, then make the segment visible to viewers</p>

<!--
[Timing checkpoint: 25:00]

ここから保存と公開の順序を見ます。
まずコード名を使わず、segment本体が保存されてからplaylistへ載るまでを捉えます。
その後、公開サンプルとMomentNowの保存先へ対応づけます。
-->

---

<div class="kicker">SAME HLS · DIFFERENT DESTINATION</div>

# 同じHLSを、MacまたはS3へ置く

<div class="environment-map">
  <div class="environment-row sample">
    <b>公開サンプル</b><span>iPhone</span><i>HTTP PUT</i><span>Macのファイル保存</span><i>HTTP GET</i><span>ローカルViewer</span>
  </div>
  <div class="environment-row production">
    <b>MomentNow</b><span>iPhone</span><i>署名付きURLへPUT</i><span>Amazon S3<br><small>object storage</small></span><i>CloudFront<br><small>CDN</small></i><span>Web Viewer</span>
  </div>
</div>

<div class="bottom-claim">init.mp4とm4sのDataは、どちらもiPhoneが生成する</div>

<!--
公開サンプルと個人アプリの対応です。
サンプルはMacのファイル保存とローカルViewer、個人アプリはS3とCloudFrontです。端末が生成するHLSの構造は変わりません。
S3はファイルをobjectとして保存するサービス、CloudFrontはそのファイルを視聴者の近くから配るCDNです。

[Sources]
- iosdc2026HLSSample/README.md
- MomentNow-Lambda/AGENTS.md
-->

---

<div class="kicker">ATOMIC REPLACE · PUBLIC SAMPLE</div>

# 書き込み中は、古い完成ファイルを公開し続ける

<div class="d">
 <div class="d-row"><div class="d-node"><span>① PUTを受信中</span><b>一時ファイルへ書く</b><small>flush ＋ fsyncまで完了させる</small></div><div class="d-node blue"><span>その間のViewer</span><b>古い完成版をGET</b><small>一時ファイルは参照しない</small></div></div>
 <div class="d-arrow down">↓</div>
 <div class="d-row"><div class="d-node blue fill"><span>② os.replace</span><b>公開pathを一気に置換</b><small>書き切ったファイルへ切り替える</small></div><div class="d-node blue"><span>置換後のViewer</span><b>新しい完成版をGET</b><small>途中の内容を見せない</small></div></div>
</div>
<div class="bottom-claim">playlistの途中状態や、半分だけのm4sを公開しない</div>

<!--
[Optional detail: 時間が厳しい場合は省略]

Macサーバーはrequest bodyをdestinationへ直接書きません。
同じdirectoryの一時ファイルへ書き切り、fsyncした後にos.replaceします。Viewerには古い完成品か新しい完成品だけが見えます。

[Sources]
- iosdc2026HLSSample/server/server.py
-->

---

<div class="kicker">PLAYABLE STATE</div>

# Dataの生成だけでは、<br>Viewerからは再生できない

<div class="d">
 <div class="d-row"><div class="d-node"><span>端末内</span><b>Dataを生成</b><small>Writer delegateで受信</small></div><i class="d-arrow">→</i><div class="d-node blue"><span>保存先</span><b>init ＋ m4sを保存</b><small>HTTP成功を確認</small></div><i class="d-arrow">→</i><div class="d-node blue fill"><span>公開</span><b>playlistに掲載</b><small>URIから取得可能になる</small></div></div>
 <div class="d-arrow down">↓</div>
 <div class="d-node blue"><span>Viewer</span><b>playlistを読み、initとsegmentをGETして再生</b></div>
</div>
<div class="bottom-claim">保存と公開が揃うと再生可能になる。実際の開始時刻はPlayerにも依存する</div>

<!--
AVAssetWriterがDataを返しただけでは、まだ視聴者は再生できません。
initと最初のm4sが保存先に存在し、playlistへ最初のsegmentが載った時点が最初の再生可能状態です。
-->

---

<div class="kicker">PLAYLIST STATE · 1 / 2</div>

# PUT成功を確認してから、候補のplaylistを採用する

<div class="d">
 <div class="d-row"><div class="d-node"><span>アプリ内の現在値</span><b>segment 1</b></div><i class="d-arrow">→</i><div class="d-node blue"><span>コピーして候補を作成</span><b>segment 1・2</b></div><i class="d-arrow">→</i><div class="d-node blue"><span>サーバーへ送信</span><b>playlist PUT</b></div></div>
 <div class="d-arrow down">↓</div>
 <div class="d-row"><div class="d-node blue fill"><span>成功を確認できた</span><b>現在値を1・2へ更新</b></div><div class="d-node warn"><span>成功を確認できない</span><b>現在値は1のまま</b></div></div>
</div>
<div class="bottom-claim">segment 2本体は保存済み。この図はplaylistとアプリ内状態の更新</div>

<!--
segment本体を保存した後のplaylist処理です。アプリ内のmanifestをコピーして候補を作り、その候補をPUTします。
PUT成功を確認した場合だけ現在値へ代入します。通信エラー時にアプリ内だけ先へ進まないようにします。
レスポンスを受け取れなかった場合でもサーバーで保存済みの可能性はあるため、失敗時にサーバーの内容が必ず古いままとは説明しません。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSManifest.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HTTPHLSClient.swift
-->

---

<div class="kicker">PLAYLIST STATE · 2 / 2</div>

# 候補を送信し、成功後にmanifestへ代入する


```swift
var nextManifest = manifest
nextManifest.addSegment(seq: seq, durationSec: durationSec)
try await client.putPlaylist(
    streamId: streamId, text: nextManifest.text)
manifest = nextManifest
```
<div class="d d-row"><div class="d-node"><span>通信前</span><b>nextManifestだけ変更</b></div><i class="d-arrow">→</i><div class="d-node blue fill"><span>awaitが成功した後</span><b>manifestを置き換える</b></div></div>
<div class="bottom-claim">PUTがthrowした場合、最後の代入へ進まない</div>

<!--
HLSStreamPublisherの抜粋です。実装ではputPlaylistをretryingで包んでいますが、ここは状態の確定位置へ注目するため省略しています。
候補の作成、通信、現在値への代入の順番をコードへ対応づけます。
[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---

<div class="kicker">MOMENTNOW · PUBLISH SEQUENCE</div>

# S3へDataをPUTし、APIでplaylistを公開する

<div class="d d-sequence">
 <b class="d-actor">iPhone</b><b class="d-actor">API</b><b class="d-actor">S3</b>
 <div class="d-msg" style="grid-column:1/3">① presign</div>
 <div class="d-msg back" style="grid-column:1/3">② 署名付きURL</div>
 <div class="d-msg bytes" style="grid-column:1/4">③ segment DataをPUT</div>
 <div class="d-msg back" style="grid-column:1/4">④ 保存成功（2xx）</div>
 <div class="d-msg" style="grid-column:1/3">⑤ commit</div>
 <div class="d-msg" style="grid-column:2/4">⑥ playlistを更新</div>
</div>
<div class="bottom-claim">APIは保存の許可と公開を制御。映像Dataを変換しない</div>

<!--
MomentNowの書き込み処理は3段階です。
S3へPUTするための署名付きURLを受け取り、segment本体を保存し、成功したあとにだけplaylistへ載せます。

MomentNowではpresign API、S3 PUT、commit API、CloudFrontが担当します。
commitは今回選んだ責務配置です。HLSに必須のAPIではなく、公開サンプルのようにiOS側でplaylistと公開順を管理する構成も可能です。
公開サンプルはiOSがplaylist本文を生成し、MacのHTTPサーバーが受け取ったファイルを保存します。S3自体はHLSの必須要素ではありません。

[Sources]
- MomentNow-Lambda/src/presign.ts
- MomentNow-Lambda/src/commit.ts
- iosdc2026HLSSample/server/server.py
-->

---

<div class="kicker">S3 PRESIGNED PUT URL</div>

# S3へのPUTを、署名付きURLで許可する

<div class="choice-compare">
  <div class="choice muted"><span>DO NOT</span><b>AWS credentialを内包</b><small>漏えい範囲が広い</small></div>
  <div class="choice-arrow">→</div>
  <div class="choice selected"><span>PRESIGNED URL</span><b>S3へ1ファイルだけPUT</b><small>保存先 / 有効期限を署名</small></div>
</div>

<div class="bottom-claim">iPhoneは発行されたURLへDataをPUTするだけ。AWS credentialは持たない</div>

```swift
var request = URLRequest(url: presignedURL)
request.httpMethod = "PUT"
request.setValue(
    contentType, forHTTPHeaderField: "Content-Type")
request.httpBody = data
```

<!--
[Optional detail: 時間が厳しい場合は省略]

iOSアプリはS3のcredentialを持ちません。
認証済みAPIが、S3の特定ファイルへPUTするためのpresigned URLを発行します。
URLには保存先と有効期限が署名されているため、アプリへAWS credentialを配布する必要がありません。

[Sources]
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html
- MomentNow-Lambda/src/presign.ts
-->

---

<div class="kicker">INIT ONCE · MEDIA REPEATS</div>

# mediaは、PUT後にcommitを追加

```swift
let url = try await api.presign(kind: kind, seq: seq)
try await api.put(to: url, data: data)
```

<div class="publish-code-pair">
  <div>
    <span>INITIALIZATION · 1回</span>
    <code class="publish-parameters">kind: "init", seq: nil</code>
    <b class="publish-result">PUT成功で完了</b>
  </div>
  <div>
    <span>MEDIA · 約2秒ごと</span>
    <code class="publish-parameters">kind: "segment", seq: seq</code>
    <b class="publish-result">PUT成功後に追加</b>
    <code class="publish-commit">try await api.commit(seq: seq)</code>
  </div>
</div>

<div class="bottom-claim">mediaだけは、PUT成功後にcommitしてplaylistへ載せる</div>

<!--
最初のinitialization Dataは1配信で一度だけ保存します。
上の2行は共通処理です。kindとseqは下段のように切り替え、dataにはinitDataまたはsegmentDataを渡します。
media Dataは約2秒ごとに、seqに対応する署名付きURLを取得してPUTします。
保存成功後にcommitし、サーバーへplaylistへ載せてよいseqを通知します。
-->

---

<div class="kicker">SAVE BEFORE PUBLISH</div>

# segmentの保存前に公開すると、<br>Viewerは404になる

<div class="d">
 <div class="d-label d-warn">順序を逆にした場合</div>
 <div class="d-row"><div class="d-node warn"><b>playlistへ先に掲載</b></div><i class="d-arrow">→</i><div class="d-node warn"><b>ViewerがGET</b></div><i class="d-arrow">→</i><div class="d-node warn fill"><b>本体がまだない</b><small>404</small></div></div>
 <div class="d-label" style="margin-top:28px">今回の順序</div>
 <div class="d-row"><div class="d-node blue"><b>本体のPUT成功</b></div><i class="d-arrow">→</i><div class="d-node blue"><b>playlistへ掲載</b></div><i class="d-arrow">→</i><div class="d-node blue fill"><b>Viewerが取得可能</b></div></div>
</div>

<!--
順序が重要です。
PUTの2xxを確認してからcommitします。逆なら、Playerがplaylistで見つけたURIをGETして404になります。
公開サンプルは同じファイルを最大3回まで再送し、それでも失敗した場合はplaylist更新へ進まずerrorとして残します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---

<div class="kicker">RETRY · PUBLIC SAMPLE</div>

# 同じPUTを最大3回試し、失敗したら公開を進めない

<div class="d">
 <div class="d-row"><div class="d-node"><span>1回目が失敗</span><b>250 ms待つ</b></div><i class="d-arrow">→</i><div class="d-node"><span>2回目が失敗</span><b>500 ms待つ</b></div><i class="d-arrow">→</i><div class="d-node"><span>3回目</span><b>最後の試行</b></div></div>
 <div class="d-arrow down">↓</div>
 <div class="d-row"><div class="d-node blue fill"><span>いずれかで成功</span><b>playlistのPUTへ進む</b></div><div class="d-node warn"><span>3回とも失敗</span><b>公開せずerrorを残す</b></div></div>
 <div class="d-caption">初回を含めて最大3回。同じURLへ同じ内容を送る</div>
</div>
<div class="bottom-claim">segment本体のPUTが成功するまで、そのsegmentをplaylistへ載せない</div>

<!--
[本編必須: アップロード失敗時の扱い]

各PUTは初回を含め最大3回試します。ここではsegment本体のPUTを例にしています。
成功すれば次のplaylist PUTへ進みます。3回とも失敗した場合はerrorを残し、後続の公開も進めません。
playlist PUT自体にも同じretry方針を適用しています。
3回という上限は、短い一時障害を再試行しつつ失敗を無期限に隠さないためのSampleの設定例です。HLSが要求する回数でも、どの回線でも最適な回数でもありません。
250 ms / 500 msは再試行を直ちに連打しないための短い待ちです。実装は250 × attemptなので、一般的な指数backoffやjitterを実装済みとは説明しません。
HTTP request自体の待ち時間もあるため、合計750 ms以内で必ず成功・失敗が確定するという意味ではありません。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSampleTests/HLSStreamPublisherTests.swift
-->

---

<div class="kicker">UPLOAD ORDER · MOMENTNOW</div>

# uploadの完了順と、再生順は別に扱う

<div class="d">
 <div class="d-axis"><span>upload開始</span><span>時間 →</span></div>
 <div class="d-lane"><b>segment 1</b><span class="d-bar blue" style="grid-column:2/10">upload</span><span style="grid-column:10/14">commit 1</span></div>
 <div class="d-lane"><b>segment 2</b><span class="d-bar blue" style="grid-column:3/7">upload</span><span style="grid-column:7/14">commit 2（先に完了）</span></div>
 <div class="d-lane"><b>segment 3</b><span class="d-bar blue" style="grid-column:4/11">upload</span><span style="grid-column:11/14">commit 3</span></div>
 <div class="d-row"><div class="d-node"><span>完了順の例</span><b>2 → 1 → 3</b></div><i class="d-arrow">→</i><div class="d-node blue"><span>MomentNowの現状</span><b>sequence順へ整列</b><small>1 → 2 → 3</small></div></div>
</div>
<div class="bottom-claim">整列だけでは、途中の欠番を待てない</div>

<!--
MomentNowはsegmentごとにTaskでuploadするため、await中に完了順が入れ替わる可能性があります。
各commitはそのsegmentのPUT成功後に行います。commit APIはsequence順へ整列します。
右下の1・2・3は全segmentが揃った時点の順序です。次のページで、揃う前に何が公開されるかを区別します。

[Sources]
- MomentNow-Lambda/src/commit.ts
-->

---

<div class="kicker">PUBLICATION GAP · CURRENT / PROPOSAL</div>

# 欠番があるとき、どこまで公開するか

<div class="d">
 <div class="d-label">segment 1・3が保存済み、segment 2はまだ届かない時点</div>
 <div class="d-queue"><span>1 保存済み</span><span class="warn">2 未到着</span><span>3 保存済み</span></div>
 <div class="d-pair" style="margin-top:24px">
  <div class="d-node blue"><span>MomentNow · 現状</span><b>1・3を番号順に掲載</b><small>欠番は待たない</small><span class="d-chip blue">1</span><span class="d-chip blue">3</span></div>
  <div class="d-node future"><span>改善案 · 未実装</span><b>連続した1だけを公開</b><small>3は2が届くまで保留</small><span class="d-chip blue">1</span><span class="d-chip missing">2待ち</span><span class="d-chip missing">3保留</span></div>
 </div>
</div>
<div class="bottom-claim">番号順への整列と、連続した範囲だけの公開は別の制御</div>

<!--
1と3が保存され、2がまだ届かない例です。現在のcommit APIは欠番を待つ契約ではありません。
改善するなら連続したsequenceまでの公開済み境界を管理し、2が来るまで3を保留します。これは未実装の設計案であり、現状の保証として説明しません。
[Sources]
- MomentNow-Lambda/src/commit.ts
-->

---

<div class="kicker">CACHE · MUTABLE PLAYLIST</div>

# playlistは同じURLでも内容が増える

<div class="d">
 <div class="d-row"><div class="d-node"><span>前回の内容</span><b>segment 1</b></div><i class="d-arrow">→</i><div class="d-node blue fill"><span>サーバーの最新</span><b>segment 1・2</b></div></div>
 <div class="d-arrow down">↓</div>
 <div class="d-row"><div class="d-node warn"><span>古い内容を使い続けると</span><b>segment 2を発見できない</b></div><div class="d-node blue"><span>今回のキャッシュ方針</span><b>no-store / no-cache</b><small>古いplaylistを使い続けない</small></div></div>
</div>
<div class="bottom-claim">変わるのはplaylistの内容。Playerは同じURLを繰り返し取得する</div>

<!--
[本編必須: キャッシュ制御]

playlistは同じURLの本文が変化します。
SampleおよびMomentNowはplaylistにno-store、no-cache等を設定します。no-storeとno-cache自体は同義ではなく、前者は保存禁止、後者は再利用前の検証を求める指定です。
CDNでも古いplaylistを固定的に配らない設定が必要です。
記事でも、更新されるplaylistを古いcacheで固定しない方針を説明しています。
max-age=0は直ちに古くなる扱いです。CloudFrontではMinimum TTLが正だとoriginのno-cache / no-storeよりTTLが優先されるため、playlist向けのcache policyも合わせて確認します。

[Sources]
- iosdc2026HLSSample/server/server.py
- MomentNow-Lambda/src/create_stream.ts
- MomentNow-Lambda/src/presign.ts
- https://zenn.dev/hs7/articles/080eac650f65ba （CloudFrontキャッシュ設定）
- https://www.rfc-editor.org/rfc/rfc9111.html
- https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Expiration.html
-->

---

<div class="kicker">CACHE · IMMUTABLE MEDIA</div>

# initとsegmentは内容を変えず、<br>キャッシュを再利用する

<div class="d">
 <div class="d-row"><div class="d-node blue"><span>保存済み</span><b>000001.m4s</b><small>同じURLでは<br>内容を変えない</small></div><i class="d-arrow">→</i><div class="d-node"><span>CDN / キャッシュ</span><b>完成Dataを長期再利用</b><small>max-age：365日</small></div><i class="d-arrow">→</i><div class="d-node blue"><span>Viewer</span><b>同じDataを取得</b><small>再利用できる</small></div></div>
 <div class="d-row"><div class="d-node"><span>次のsegment</span><b>000002.m4sは別URL</b></div><div class="d-node"><span>部分取得への対応</span><b>Range request / 206</b></div></div>
</div>
<div class="bottom-claim">更新されるplaylistと、変わらないメディアで方針を分ける</div>

<!--
initとm4sは一度保存したら変更しないため長期cacheします。SampleのファイルサーバーとMomentNowのS3 objectでmax-age=31536000を設定しています。
配信ごとに保存先が異なり、segmentは番号付きの別URLなので、新しいsegmentと古いcacheが衝突しません。
Range / 206はキャッシュ方針とは別の部分取得機能です。取得済みだから全てのViewerが必ず通信ゼロになるという図にはしていません。
31536000秒は365日です。内容が変わらないURLを長く再利用するための値で、HLSの必須TTLではありません。
immutableは有効期間中に同じURLの内容が変わらないという宣言です。新しいsegmentには新しいURLを使う前提と組み合わせます。
cacheの有効期間はS3の保存期限とは別です。365日の保存や、cacheに必ず365日間残ることを保証するものではありません。CDNのMaximum TTLなどにも影響されます。
[Sources]
- iosdc2026HLSSample/server/server.py
- MomentNow-Lambda/src/presign.ts
- MomentNow-Lambda/src/create_stream.ts
- https://www.rfc-editor.org/rfc/rfc9111.html
- https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Expiration.html
-->

---
layout: center
class: chapter
---

<div class="chapter-no">05</div>
<div class="chapter-rule"></div>

# 配信の完了
<p>AVFoundation callback → Swift Task → final playlist</p>

<!--
[Timing checkpoint: 30:00]

最後に、iOS上のcallback、Task、URLSessionを1本のライブ配信として整理します。
焦点は、メディア処理をHTTP待ちから切り離す境界です。
Writerが受け取れない問題と、生成済みsegmentが送信待ちになる問題を分けて説明します。
-->

---

<div class="kicker">CALLBACK TO TASK · PUBLIC SAMPLE</div>

# callbackはすぐ戻り、HTTPは別のTaskで待つ

<div class="d">
 <div class="d-caption"><b>Writer delegate</b> の値を <b>writingQueue</b> へ戻して受け渡す</div>
 <div class="d-axis"><span>録画中</span><span>時間 →</span></div>
 <div class="d-lane"><b>Capture</b><span class="d-bar" style="grid-column:2/14">frame / audio blockを継続して受信</span></div>
 <div class="d-lane"><b>受け渡し</b><span class="d-bar blue" style="grid-column:2/5">yield 1</span><span class="d-bar blue" style="grid-column:6/9">yield 2</span><span class="d-bar blue" style="grid-column:10/14">yield 3</span></div>
 <div class="d-lane"><b>HTTP Task</b><span class="d-bar blue" style="grid-column:2/10">segment 1とplaylistをawait</span><span class="d-bar blue" style="grid-column:10/14">次を送信</span></div>
 <div class="d-caption">受け渡しはAsyncThrowingStream。Taskが順番に取り出す</div>
</div>
<div class="bottom-claim">HTTP待ちでcallbackを止めない。ただし、送信待ちの上限は未設定</div>

<!--
AVAssetWriterDelegate自体のcallback queueをwritingQueueと断定しません。Sampleのdelegate実装は受け取ったDataをwritingQueue.asyncへ渡し、そこでsequenceとContinuationを操作します。
continuation.yieldでHLSFragmentを渡したらすぐ戻り、PublisherのTaskがfor awaitで取り出してPUTとplaylist更新を順番にawaitします。
図は処理の重なりの模式図で、実行時間は実測ではありません。AsyncThrowingStreamは既定の無制限bufferなので、送信が遅いと生成済みDataが滞留します。


-->

---

<div class="kicker">BACKPRESSURE · TWO LOCATIONS</div>

# Writer側のdropと、生成後の送信待ち

<div class="d">
 <div class="d-row d-backpressure"><div class="d-node"><span>生成前</span><b>CMSampleBuffer</b><small>frame / audio block</small></div><i class="d-arrow">→</i><div class="d-node blue"><span>Writer</span><b>圧縮・分割</b></div><i class="d-arrow">→</i><div class="d-node blue fill"><span>生成後</span><b>segment Data</b><small>送信待ちの列</small></div><i class="d-arrow">→</i><div class="d-node blue"><span>HTTP</span><b>送信</b></div></div>
 <div class="d-pair" style="margin-top:24px"><div class="d-node warn"><span>Writer側 · 実装済み</span><b>受け取れないbufferをdrop</b><small>映像のカクつき・音声の欠けを許容</small></div><div class="d-node future"><span>送信側 · 改善案</span><b>滞留量の上限と停止判断</b><small>Sampleでは未実装</small></div></div>
</div>
<div class="bottom-claim">別の位置・別のデータに対する問題。送信待ちは現在、上限なし</div>

<!--
生成前のCMSampleBufferと、生成後のsegment Dataを区別します。
appendImmediatelyがfalseの場合にdropするのは前者です。生成済みsegmentの待ち行列には作用しません。
Capture側のalwaysDiscardsLateVideoFrames = trueも、古い映像frameを溜め続けないための別の設定です。video callbackへ渡す前の制御で、Receiverのfalseとは別です。
画の欠落を避ける録画とはトレードオフがあります。Sampleでは同じDataOutputから保存用Writerにも渡すため、Capture側で落ちたframeは保存側にも届きません。
この設定も生成済みsegmentのHTTP待ちを解消しません。
生成に対して送信が遅い状態が続くと滞留が増えます。上限と超過時の停止判断は今後の改善として扱い、ライブ遅延全体を解消する実装済み対策とは説明しません。

[Sources]
- https://developer.apple.com/library/archive/technotes/tn2445/_index.html
- https://developer.apple.com/documentation/avfoundation/avassetwriterinput/samplebufferreceiver/appendimmediately(_:)
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">WRITER FINISH</div>

# HLS用Writerを、最後に渡した時刻で終了する

<div class="stop-flow">
  <div><b>lastAdjustedPTS</b><span>最後に受け入れたPTS</span></div>
  <i>→</i>
  <div><b>endSession</b><span>範囲を閉じる</span></div>
  <i>→</i>
  <div><b>Receiver.finish</b><span>video / audio</span></div>
  <i>→</i>
  <div><b>finishWriting</b><span>最後のDataを出力</span></div>
</div>

```swift
if lastAdjustedPTS.isValid {
    writer.endSession(atSourceTime: lastAdjustedPTS)
}
```

<!--
[Optional detail: 時間が厳しい場合は省略]

終了時刻も補正後のPTSを使います。
ここはHLS用Writerの終了手順です。保存用WriterもReceiverをfinishしてfinishWritingの完了を待ち、completedかつ映像・音声が揃っている場合だけ写真保存へ進みます。
finishWritingによって最後のsegmentがdelegateへ届く可能性があるため、stopはその完了まで待ちます。
-->

---

<div class="kicker">STOP · RECORDER</div>

# Captureを止めてから、<br>両Writerの生成を完了させる

<div class="d">
 <div class="d-row"><div class="d-node"><span>① Capture停止</span><b>新しい入力を止める</b></div><i class="d-arrow">→</i><div class="d-node"><span>② writingQueue</span><b>投入済みcallbackを処理</b></div></div>
 <div class="d-arrow down">↓</div>
 <div class="d-row"><div class="d-node blue fill"><span>③ HLS Writerをfinish</span><b>最後のDataを受信</b><small>fragment streamを閉じる</small></div><i class="d-arrow">→</i><div class="d-node green fill"><span>④ 保存用Writerをfinish</span><b>MP4の完成を確認</b><small>URLまたは生成エラーを返す</small></div></div>
</div>
<div class="bottom-claim">Sampleの現在の呼び出し順。HTTPの送信Taskはこの間も継続する</div>

<!--
HLSSegmentRecorder.stopの現在の順序です。まずsessionQueue上でCaptureSession.stopRunningの完了を待ち、その後writingQueue上で投入済みcallbackの後に終了処理を行います。
HLS Writerをfinishし、最後のDataを受けてfragment streamを閉じます。その後、保存用WriterをfinishしてMP4を完成させます。
この2つのWriterのfinishを同時に開始する実装ではありません。途中で一方が失敗していても、もう一方の結果を独立に扱います。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleHLSStreamer.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---

<div class="kicker">STOP · HTTP AND PHOTOS</div>

# 写真保存は、HTTP送信の完了を待たずに始める

<div class="d">
 <div class="d-lane d-axis-grid"><b>時間 →</b><span style="grid-column:2/5">録画中</span><span style="grid-column:8/11">停止操作</span><span style="grid-column:11/14">終了処理</span></div>
 <div class="d-lane"><b>HLS送信</b><span class="d-bar blue" style="grid-column:2/10">生成済みsegmentを順に送信</span><span class="d-bar blue" style="grid-column:10/14">ENDLIST公開</span></div>
 <div class="d-lane"><b>端末MP4</b><span class="d-bar green" style="grid-column:2/8">撮影中から生成</span><span class="d-bar green" style="grid-column:8/11">MP4完成</span></div>
 <div class="d-lane"><b>写真保存</b><span class="d-bar green" style="grid-column:11/14">写真へ追加</span></div>
 <div class="d-row"><div class="d-node blue"><span>HLSの結果</span><b>公開完了 / 送信エラー</b></div><div class="d-node green"><span>写真保存の結果</span><b>保存成功 / 完成MP4を保持</b><small>保存失敗時は再試行</small></div></div>
</div>
<div class="bottom-claim">2つの結果を別々に確認。完了順は通信・保存の状況によって変わる</div>

<!--
図は並行する経路と依存関係の模式図です。ENDLIST公開と写真保存の完了順は固定ではありません。
SampleHLSStreamerはRecorder.stopでMP4の結果を受け取るとonLocalRecordingを呼びます。HTTP Taskは録画中から進んでおり、その完了を待たず写真保存を開始します。
写真保存の処理後、uploadTask.valueをawaitして最終snapshotを得ます。HTTPに失敗していても完成MP4は保存対象です。写真保存に失敗した場合は完成MP4を保持し、再試行できます。
[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleHLSStreamer.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/PhotoVideoSaver.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---

<div class="kicker">MOMENTNOW · OPERATIONAL RESPONSIBILITIES</div>

# 4つの運用制御を、配信の担当箇所へ配置

<div class="d">
 <div class="d-row"><div class="d-node"><span>iPhone / user</span><b>HLSを生成</b></div><i class="d-arrow">→</i><div class="d-node blue"><span>S3</span><b>Dataを保存</b></div><i class="d-arrow">→</i><div class="d-node blue"><span>playlist / CDN</span><b>Viewerへ公開</b></div></div>
 <div class="d-row"><div class="d-node"><span>AUTH</span><b>誰の配信か</b><small>user / group</small></div><div class="d-node blue"><span>PRESIGN</span><b>どこへPUTできるか</b><small>ファイル・期限を限定</small></div><div class="d-node blue"><span>COMMIT</span><b>何を公開するか</b><small>playlist更新を制御</small></div><div class="d-node"><span>DELIVERY</span><b>誰に配るか</b><small>CloudFront / ticket / status</small></div></div>
</div>
<div class="bottom-claim">MomentNowで選んだ配置。SampleはiOS側でplaylistと公開順を管理</div>

<!--
iosdc2026HLSSampleはMacを小さなobject serverとして使います。
個人アプリは先にpresignしてS3へPUTし、Lambdaのcommitでplaylistを更新します。生成するDataと保存順序は共通です。

サンプルから個人アプリへ足すものです。
認証、署名URL、playlist競合制御、CDN配信。どれも重要ですが、映像の再エンコードではありません。
4つすべてがHLSの必須構成という意味ではなく、MomentNowの用途に合わせて採用した構成です。
公開順もiOS側で制御できます。今回のサーバー側commitはplaylist更新時の競合制御もまとめた責務配置であり、唯一の正解とは位置づけません。

[Sources]
- iosdc2026HLSSample/README.md
- MomentNow-Lambda/src/presign.ts
- MomentNow-Lambda/src/commit.ts
- MomentNow-Lambda/src/create_stream.ts
- MomentNow-Lambda/src/ticket.ts
-->

---

<div class="kicker">まとめ</div>

# 実ユーザーの声を、改善につなげきれていない

<p class="d-note">MomentNowは、想定ほど利用が広がらなかった</p>
<div class="d">
 <div class="d-node warn"><b>利用者の声を十分に集められていない</b><small>画質・待ち時間への評価を、設定の見直しへつなげきれていない<br>一度決めた値のまま、継続している設定が多い</small></div>
</div>
<div class="d d-pair">
 <div class="d-node green"><span>想定している用途</span><b>少人数・短時間・単一品質</b><small>端末や上り回線の条件にも依存</small></div>
 <div class="d-node blue"><span>別の構成を検討する条件</span><b>大規模・長時間・複数画質</b><small>専用media platformなどを検討</small></div>
</div>
<div class="bottom-claim">設定の妥当性を、利用実績とユーザーの声で検証することが課題</div>

<!--
最後に、正直な反省点です。MomentNowは想定ほど利用が広がりませんでした。
本番でのリアルなユーザーの声を十分に集められず、画質や待ち時間の評価を設定の見直しへ反映しきれていません。設定値にも、一度決めた値のまま使い続けているものが多いです。
紹介した設定の一般的な性質は説明できますが、それによって、この数値が利用者にとって適切だと検証できたことにはなりません。
今回共有できるのは、端末内でHLSを生成して配信する実装と、その過程で分かったことです。実装時の動作確認と、実ユーザーの評価に基づく最適化は分けて考えています。
少人数・短時間・単一品質は想定した用途です。その適合性を本番の利用実績で十分に裏付けた、とは説明しません。新しいiPhoneや安定した上り回線といった条件にも依存します。
大規模、長時間、複数画質のABR、厳しい可用性要件があるなら、専用media platformを含めて別の構成を検討します。「端末でできる」と「端末でやるべき」は別の判断です。
今後は画質や待ち時間について利用者の声を集め、実際の配信状況と合わせて設定を見直すことが課題です。実施済みの改善や計測結果としては扱いません。
-->

---
layout: center
class: closing
---

<div class="kicker">まとめ</div>

# 端末内で生成し、保存できた範囲を公開する

<div class="d d-row">
 <div class="d-node"><span>生成</span><b>Camera / Mic<br>からWriterへ</b><small>撮影データ<br>を入力</small></div><i class="d-arrow">→</i>
 <div class="d-node"><span>区切り</span><b>約2秒の<br>fMP4 Data</b><small>IDRと境界を揃える</small></div><i class="d-arrow">→</i>
 <div class="d-node"><span>公開</span><b>保存成功後に<br>playlist更新</b><small>ViewerがGET</small></div><i class="d-arrow">→</i>
 <div class="d-node"><span>終了</span><b>最後の送信と<br>ENDLIST</b><small>保存MP4も完了確認</small></div>
</div>
<div class="bottom-claim">AVCaptureSession → DataOutput → Receiver → AVAssetWriterDelegate</div>

<!--
[Timing checkpoint: 34:30]

まとめです。
AVAssetWriterDelegateでiPhoneからfMP4を逐次取り出し、保存できたsegmentだけをplaylistへ追加します。
CameraとMicの入力をWriterへ渡してHLS用Dataを生成し、segment単位で保存・公開していきます。
撮影停止後も最後のsegmentとENDLISTの公開まで待つところが、配信全体の完了です。

実装の接続は、AVCaptureSessionからDataOutput、Receiver、AVAssetWriterDelegateまでの1本です。

公開サンプルは、そのうちAVFoundationの生成処理を読みやすくした教材です。

ここまでが本編です。次のページで締めます。
チェックポイントは練習用の仮目安であり、約3分の余裕が実測で確認できているわけではありません。
デモ込みで35〜37分を通し練習の目標とし、所要時間は実測で判断します。補足は本編に含めません。
-->

---
layout: center
class: closing thanks-slide
---

<div class="closing-footer">
  <div>
    <b>ありがとうございました</b>
    サンプルアプリのコード<br>
    <a href="https://github.com/HikaruSato/iosdc2026HLSSample">https://github.com/HikaruSato/iosdc2026HLSSample</a><br><br>
    MomentNowの開発記事<br>
    <a href="https://zenn.dev/hs7/articles/080eac650f65ba">https://zenn.dev/hs7/articles/080eac650f65ba</a>
  </div>
</div>

<!--
ありがとうございました。
サンプルアプリは、このURLで公開しています。
MomentNowの開発経緯はZennの記事にもまとめています。
-->

---
layout: center
class: chapter
---

<div class="kicker">APPENDIX</div>
<div class="chapter-rule"></div>

# 補足：Writerの時刻処理
<p>開始の基準 / 10秒の理由 / 時刻を変えるコード</p>

<!--
ここからは質問や実装時の参照用の補足です。40分の本編には含めません。
Apple公式サンプルの時刻移動の方針と、今回のCapture入力に合わせた処理を説明します。
-->

---

<div class="kicker">APPENDIX · SESSION START</div>

# 最初の映像を基準にWriterを開始

<div class="start-sequence">
  <div class="sequence-item audio"><span>先に届いた音声</span><small>まだappendしない</small></div>
  <div class="sequence-line"></div>
  <div class="sequence-item video"><span>最初の映像</span><small>writer.start()</small></div>
  <div class="sequence-line active"></div>
  <div class="sequence-item session"><span>10秒を開始位置に</span><small>映像・音声に共通</small></div>
</div>

```swift
guard output === videoOutput else { return }
guard !didStartSession, let writer else { return }
try writer.start()
let pts = CMSampleBufferGetPresentationTimeStamp(sampleBuffer)
writer.startSession(atSourceTime: startTimeOffset)
timeOffsetDelta = startTimeOffset - pts
```

<div class="source">HLSSegmentRecorder.startWriterIfNeeded（エラー処理・状態更新は省略）</div>

<!--
今回のCapture入力では、最初のvideo frameでWriterのsessionを開始します。
そのframeのPTSから一度だけtimeOffsetDeltaを決めます。
先にaudioが届いても、基準が決まるまではappendしません。
callbackの到着順と、AAC圧縮に伴うprimingは別の話です。10秒の余白は次のページで説明します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---
class: timing-priming
---

<div class="kicker">APPENDIX · AAC PRIMING</div>

# 10秒の開始位置は、音声の時刻を前へ動かす余白

<div class="d">
 <div class="d-lane d-axis-grid"><b>時間 →</b><span style="grid-column:2/5">0秒</span><span style="grid-column:8/14">映像の開始位置：10秒</span></div>
 <div class="d-lane"><b>映像</b><span class="d-bar blue" style="grid-column:8/14">映像本体</span></div>
 <div class="d-lane"><b>AAC音声</b><span class="d-bar warn" style="grid-column:5/8">priming</span><span class="d-bar blue" style="grid-column:8/14">音声本体</span></div>
 <div class="d-caption">priming分だけ音声の時刻を前へ補償しても、0秒より前にならない</div>
</div>

```swift
private let startTimeOffset = CMTime(value: 10, timescale: 1)
writer.initialSegmentStartTime = startTimeOffset
```

<div class="bottom-claim">模式図。10秒はAppleの設定例であり、再生開始まで待つ時間ではない</div>

<!--
AAC encoderは、正しくencode / decodeするために先頭へprimingを加えます。
Apple HLS profileはedit listを使わず、音声のbaseMediaDecodeTimeをpriming分だけ前へ移して補償します。
この値は符号なし整数なので、負にできません。そのためAppleは両方のmedia timeを同じ量だけ後ろへ移すことを勧めています。
initialSegmentStartTimeも同じ開始位置に合わせます。
10秒はHLS仕様の固定値ではありません。Appleのmediafilesegmenterと同じ値を選べる、という説明に合わせています。
必要なのは音声の補償で負の時刻にならない余白です。10秒を唯一の正解や、動画の先頭に無音を10秒挿入する処理として説明しません。
HLSの再生は最初の映像の提示時刻から始まるため、この設定による10秒の待ち時間は発生しません。
マイクのcallbackが映像より先に届くこととは区別して説明します。

[Sources]
- https://developer.apple.com/videos/play/wwdc2020/10011/?time=976
- https://developer.apple.com/documentation/avfoundation/writing-fragmented-mpeg-4-files-for-http-live-streaming
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---
layout: center
class: formula-slide
---

<div class="kicker">APPENDIX · ONE DELTA</div>

# 最初の映像で移動量を決め、<br>すべてのsampleに適用

<div class="d">
 <div class="d-row"><div class="d-node"><span>移動量を最初に1回だけ決定</span><b>delta = 10s − firstVideoPTS</b></div></div>
 <div class="d-pair"><div class="d-node blue"><span>映像の全sample</span><b>sourcePTS ＋ delta</b></div><div class="d-node blue"><span>音声の全sample</span><b>sourcePTS ＋ delta</b></div></div>
 <div class="d-time-track"><div>映像 100.00 → 10.00</div><div>音声 100.02 → 10.02</div></div>
 <div class="d-time-gap">同じ−90秒。元の0.02秒の時間差を保つ</div>
</div>
<div class="bottom-claim">映像と音声それぞれの先頭を、別々に10秒へ合わせない</div>

<!--
CaptureのPTSは配信開始からの経過時間ではなく、CaptureSessionの共通の時計上の位置です。
最初のvideo PTSからdeltaを一度だけ決め、その後は映像と音声の全CMSampleBufferへ同じ値を足します。
再生速度や映像と音声の時間差は変えません。
映像と音声それぞれの最初のsampleを別々に10秒へ合わせると、元の時間差を変えてしまいます。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avcapturesession/synchronizationclock
- https://developer.apple.com/documentation/coremedia/cmsamplebuffer/presentationtimestamp
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">APPENDIX · COPY TIMING</div>

# コピーの内容はそのまま、PTSと有効なDTSを補正

<div class="d d-row d-compact"><div class="d-node"><span>元のbuffer</span><b>映像・音声 ＋ 元の時刻</b></div><i class="d-arrow">→</i><div class="d-node blue"><span>独立したコピー</span><b>同じ内容 ＋ 補正した時刻</b></div></div>

```swift
var t = info
t.presentationTimeStamp = t.presentationTimeStamp + offset
if info.decodeTimeStamp.isValid {
    t.decodeTimeStamp = t.decodeTimeStamp + offset
}
return t
```
<p class="d-note">timing infoを変換するmap内部の抜粋。新しいtimingでbufferをコピーする</p>

<!--
CMSampleBufferの映像・音声データはそのままに、timing infoを差し替えたコピーを作ります。
PTSだけでなく、frameをdecodeする時刻であるDTSも、有効な場合は同じ量だけ補正します。
サンプルのoffsettingTimingでは、この後にoutputPresentationTimeStampも同じ量だけ補正しています。
画面はsampleTimingInfos().mapの内部の抜粋です。新しいtiming配列をCMSampleBuffer(copying:withNewTiming:)へ渡してコピーを作ります。エラー処理、コピー作成の呼び出し、output PTSの更新は画面から省略しています。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->
