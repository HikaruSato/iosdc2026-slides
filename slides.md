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
      <p><a href="https://apps.apple.com/jp/app/%E3%82%B0%E3%83%AB%E3%83%BC%E3%83%97%E5%8B%95%E7%94%BB%E5%85%B1%E6%9C%89-momentnow/id6759968148" style="color: var(--accent-primary); text-decoration: underline;">MomentNow</a> という「今この瞬間」の動画をHLSで配信し、<br>URLで共有できる iOSアプリ を個人開発してます</p>
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
  <div><b>WHY</b><span>なぜ端末内HLSを作ろうと思ったのか</span></div>
  <div><b>01</b><span>HLSがライブになる仕組み</span></div>
  <div><b>02</b><span>端末内でHLSを生成</span></div>
  <div><b>03</b><span>約2秒の動画断片を生成</span></div>
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
<div class="story-split">
  <div>
    <h1>撮影後のアップロードを待たずに、動画をURLで共有したい</h1>
    <p class="lead">HLSなら、撮影中から小さくアップロードし、撮影後URLで再生できる</p>
  </div>
  <div class="why-files">
    <div class="why-file large"><span>1ファイル</span><b>recording.mov</b><small>撮影後にまとめてupload</small></div>
    <div class="why-divider">↓ 待ち時間をなくす</div>
    <div class="why-segments">
      <span>HLS</span>
      <b>再生設定</b><b>短い動画 1</b><b>短い動画 2</b>
      <small>撮影中から順次upload → 同じURLで共有</small>
    </div>
  </div>
</div>

<!--
この実装のきっかけは、子どもの動画を家族へ送るときの待ち時間でした。
1ファイルでは、撮影を終えて大きな動画をアップロードし終わるまで、共有URLを渡せません。

HLSなら撮影中から短いsegmentを順次アップロードでき、同じplaylist URLを共有できます。
撮影後の大きなアップロードを待たず、URLで動画を共有したい。それがHLSを使おうと思った理由です。
-->

---

<div class="kicker">ORIGIN STORY · IMPLEMENTATION</div>

# [macOS向けAppleサンプルが<br>とても参考になった](https://developer.apple.com/documentation/avfoundation/writing-fragmented-mpeg-4-files-for-http-live-streaming)

<div class="implementation-story origin-story">
  <div class="attempt"><span>2025.03</span><b>AIで最初の試作</b><small>再生できるHLSとして<br>実用まで至らず</small></div>
  <i>→</i>
  <div class="reference"><span>Apple公式サンプル</span><b>正しい生成手順を確認</b><small>Writer設定 · 時刻補正<br>分割Dataの受け取り方</small></div>
  <i>→</i>
  <div class="adapt"><span>iPhone local</span><b>HLS用Data生成に成功</b><small>Camera / Micから<br>短い動画を逐次生成</small></div>
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

<div class="kicker">HLS LIVE IN ONE SENTENCE</div>

# HLSは、短い動画と<br><span class="accent-primary">更新されるplaylist</span>をHTTPで配る

<div class="playlist-model">
  <div class="manifest-card">
    <code>#EXTM3U</code>
    <code>#EXT-X-MAP: init.mp4</code>
    <code>#EXTINF: 2.000</code>
    <code class="hot">seg/000001.m4s</code>
    <code>#EXTINF: 2.000</code>
    <code class="hot">seg/000002.m4s</code>
  </div>
  <div class="playlist-arrow">→</div>
  <div class="segment-sequence">
    <span class="init">init</span>
    <span>01</span>
    <span>02</span>
    <span class="future">03</span>
  </div>
</div>

<div class="hls-tag-legend">
  <span><code>EXT-X-MAP</code><b>初期化segmentの場所</b></span>
  <span><code>EXTINF</code><b>次のsegmentの長さ</b></span>
  <span><code>URI</code><b>次に取得するファイル</b></span>
</div>

<div class="source">RFC 8216: Media Playlist / Media Segment / EXT-X-MAP</div>

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
目次、初期化セグメント、メディアセグメントの3種類です。
initだけにも、m4sだけにも、完全な再生体験はありません。playlistが関係を定義します。

[Sources]
- RFC 8216: Media Playlist / Media Segment / EXT-X-MAP
-->

---

<div class="kicker">FRAGMENTED MP4</div>

# fragmented MP4（fMP4）は<br>「設定」と「映像・音声の断片」を分ける

<div class="fmp4-beginner-model">
  <div class="fmp4-init"><span>1配信に1つ</span><b>init.mp4</b><small>Playerが再生を始めるための設定</small></div>
  <i>+</i>
  <div class="fmp4-media"><span>約2秒ごと</span><b>000001.m4s</b><small>実際の映像と音声</small></div>
  <i>=</i>
  <div class="fmp4-playable"><span>playlistが結ぶ</span><b>再生可能</b><small>Playerはこの組み合わせを取得</small></div>
</div>

<div class="source">Apple WWDC20: Author fragmented MPEG-4 content with AVAssetWriter</div>

<!--
fragmented MP4、略してfMP4は、再生設定と映像・音声本体を分けて扱えるMP4です。
init.mp4にはftypとmoov、各m4sにはmoofとmdatが入ります。
後ほどAVAssetWriterDelegateから、この単位のDataを受け取ります。

[Sources]
- https://developer.apple.com/videos/play/wwdc2020/10011/
-->

---

<div class="kicker">WHY IT IS LIVE</div>

# playlistが増えるたび、<br>Playerは次のsegmentを取得する

<div class="event-timeline">
  <div class="event-phase">
    <span class="time">t = 0</span>
    <b>playlistだけ</b>
    <small>segmentなし</small>
  </div>
  <div class="event-connector"></div>
  <div class="event-phase live">
    <span class="time">t = 2s</span>
    <b>+ seg 1</b>
    <small>追記</small>
  </div>
  <div class="event-connector"></div>
  <div class="event-phase live">
    <span class="time">t = 4s</span>
    <b>+ seg 2</b>
    <small>追記</small>
  </div>
  <div class="event-connector"></div>
  <div class="event-phase ended">
    <span class="time">stop</span>
    <b>#EXT-X-ENDLIST</b>
    <small>もう増えない</small>
  </div>
</div>

<div class="bottom-claim">Player（AVPlayer / Safariなどの再生エンジン）は同じplaylistを繰り返しGETする</div>

<div class="source">Apple: Event playlist construction / RFC 8216 §4.3.3</div>

<!--
配信側は短いsegmentを作り、そのURIをplaylistの末尾へ追加します。
Playerはplaylistを再取得し、新しく見つけたsegmentを順に取得します。この繰り返しがHLSのライブ配信です。停止時は#EXT-X-ENDLISTで閉じます。

[Sources]
- RFC 8216 §4.3.3: Media Playlist Tags
-->

---

<div class="kicker">FROM HLS TO THIS TALK</div>

# 一般的には、サーバーが受信した映像をHLSへ変換する

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

<div class="kicker">LIVE DEMO · PUBLIC SAMPLE</div>

# 公開サンプルアプリを動かす

<div class="live-demo-grid">
  <div class="live-demo-phone">
    <span>HLSSample</span>
    <div class="live-demo-camera">Camera Preview</div>
    <b><i></i> 配信中</b>
  </div>
  <div class="live-demo-arrow">→<small>HTTP PUT</small></div>
  <div class="live-demo-result">
    <div class="live-upload-strip">
      <div><span>start</span><b>PUT init.mp4</b></div>
      <div><span>+2s</span><b>PUT seg 1</b></div>
      <div><span>after PUT</span><b>PUT playlist</b></div>
    </div>
    <div class="live-demo-viewer">
      <span>Mac HTTP Server / Viewer</span>
      <b>LIVE <i>LOCAL</i></b>
      <small>playlist.m3u8を追従再生</small>
    </div>
  </div>
</div>

<div class="demo-repository">github.com/HikaruSato/iosdc2026HLSSample</div>

<!--
[Timing checkpoint: 05:00]

ここまで確認したHLSの仕組みを、公開サンプルで実際に動かします。
iPhoneはHLSを生成してMacへHTTP PUTし、MacのViewer（Playerを使う再生画面）は同じファイルをHTTP GETして追従再生します。

Safariでplaylistが更新される間の追従再生を見せ、同じplaylist URLをiOSのAVPlayerへ渡した再生確認にも触れます。
Macにinit.mp4、m4s、playlistが残ることと、停止後も同じURLで再生できることを確認します。
この説明はライブデモ成功時にも省略しません。

会場Wi-Fiが不安定な場合はライブ操作を省略し、次の5枚で同じ流れを説明します。ライブデモが成功した場合は、この5枚を省略します。

[Sources]
- iosdc2026HLSSample/README.md
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
  <span>STREAM <b>stream-...</b></span>
  <span>SEGMENTS <b>1 → 2 → 3</b></span>
  <span>PLAYLIST <b>/streams/.../playlist.m3u8</b></span>
</div>

<!--
[Optional demo walkthrough: ライブデモ成功時は省略]

MacのViewerは1秒ごとにstream一覧を取得し、更新時刻が最も新しい配信を選びます。
SafariはネイティブHLS、それ以外は同梱したhls.jsで再生します。
同じplaylist URLをiOSのAVPlayerへ渡し、WebとiOSの両方で再生を確認します。

[Sources]
- iosdc2026HLSSample/server/static/app.js
- Apple: Using AVFoundation to play and persist HTTP live streams
-->

---
class: demo-step
---

<div class="kicker">DEMO RESULT · REAL OUTPUT</div>

# 配信後に残る、3種類のHLSファイル

<div class="real-output-tree">
  <div class="tree-root">server/data/streams/<b>stream-20260818-225315-C7A2AB11/</b></div>
  <div class="tree-columns">
    <div><code>├── init.mp4</code><small>1,136 bytes</small></div>
    <div><code>├── playlist.m3u8</code><small>682 bytes</small></div>
    <div><code>└── seg/</code><small>18 files</small></div>
    <div class="segment-list"><code>000001.m4s</code><code>000002.m4s</code><code>…</code><code>000018.m4s</code></div>
  </div>
</div>

<div class="bottom-claim">保存先で <code>ffplay playlist.m3u8</code> → HLS一式をそのまま再生確認</div>

<!--
停止時は、Writerを閉じて最後のsegmentを受け取ったあと、AsyncThrowingStreamをfinishします。
すべてのイベントを処理した最後にENDLIST付きplaylistをPUTします。

これはリポジトリに残っている実際の出力です。
initが1つ、更新されるplaylistが1つ、2秒単位のm4sが18個あります。個人アプリでは同じ相対構造をS3 prefixへ置きます。
保存先のstreamディレクトリでffplayにplaylist.m3u8を渡せば、m4s単体ではなくHLS一式として再生確認できます。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleHLSStreamer.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
- iosdc2026HLSSample/server/data/streams/stream-20260818-225315-C7A2AB11
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

# AVAssetWriterはsegmentを作る。playlistは作らない。

<div class="ownership-split">
  <div class="ownership-side writer-side">
    <span>AVFoundation</span>
    <b>圧縮して約2秒に分割</b>
    <ul>
      <li>映像はH.264</li>
      <li>音声はAAC</li>
      <li>init.mp4 + media segment</li>
    </ul>
  </div>
  <div class="ownership-divider">/</div>
  <div class="ownership-side app-side">
    <span>App</span>
    <b>保存してから公開</b>
    <ul>
      <li>保存先のpathを決める</li>
      <li>HTTP成功を確認する</li>
      <li>playlistへ追加する</li>
    </ul>
  </div>
</div>

<div class="source">AVAssetWriterDelegate / HLSSegmentRecorder / HLSStreamPublisher</div>

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

<div class="kicker">HLSSegmentRecorder · SIX SDK OBJECTS</div>

# 6つのSDK objectを、2つのdelegateで接続

<div class="recorder-object-map">
  <div class="recorder-object-group capture">
    <span>CAPTURE</span>
    <b>AVCaptureSession</b>
    <div><code>AVCaptureVideoDataOutput</code><code>AVCaptureAudioDataOutput</code></div>
  </div>
  <div class="recorder-object-arrow">
    <code>captureOutput</code><i>→</i><small>CMSampleBuffer</small>
  </div>
  <div class="recorder-object-group writer">
    <span>WRITE</span>
    <div><code>videoReceiver</code><code>audioReceiver</code></div>
    <b>AVAssetWriter</b>
  </div>
  <div class="recorder-object-arrow">
    <code>writer delegate</code><i>→</i><small>segment Data</small>
  </div>
  <div class="recorder-object-output">
    <span>RECORDER OUTPUT</span>
    <b>HLSFragment</b>
    <small>initialization / media</small>
  </div>
</div>

<div class="bottom-claim">左から右へ、撮影データを「HLSとして保存できるData」へ変える</div>

<!--
HLSSegmentRecorderは、CameraやMicを直接エンコードする巨大なAPIではありません。
Capture側の3 objectとWriter側の3 objectを、2種類のdelegate callbackでつないでいます。

左ではCaptureSessionがDataOutputへCMSampleBufferを流します。
中央ではReceiverからWriterへそのbufferを渡します。
右ではWriter delegateからfMP4のDataを受け取ります。
-->

---

<div class="kicker">AUDIO SESSION</div>

# カメラ構成前に、録音用AudioSessionを有効化

```swift
let audio = AVAudioSession.sharedInstance()
try audio.setCategory(
    .playAndRecord,
    mode: .videoRecording,
    options: [.defaultToSpeaker, .allowBluetoothHFP]
)
try audio.setActive(true)
```

<div class="contract-compare">
  <div><span>Capture</span><b>camera + microphone</b></div>
  <div><span>Route</span><b>speaker + Bluetooth HFP</b></div>
</div>

<!--
[Optional detail: 時間が厳しい場合は省略]

録音権限だけではなく、AVAudioSessionのcategoryとmodeを先に設定します。
Bluetooth HFPを含む入力routeを許可しつつ、端末側の再生はspeakerを既定にしています。
-->

---

<div class="kicker">CAPTURE TOPOLOGY</div>

# CaptureSessionが、Camera / MicをDataOutputへつなぐ

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

MovieFileOutputなら完成した録画ファイルを受け取れますが、ライブ中にWriterへ少しずつ渡せません。
DataOutputを使うことで、videoは約1 frame、audioは短いblockごとのCMSampleBufferを撮影中から受け取れます。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">ONE CAPTURE, THREE OUTPUTS</div>

# 1つのCaptureSessionを、3つの用途で使う

<div class="capture-branches">
  <div class="capture-branch-source"><span>Camera + Mic</span><b>CaptureSession</b></div>
  <div class="capture-branch-lines"><i></i><i></i><i></i></div>
  <div class="capture-branch-targets">
    <div class="preview"><span>画面表示</span><b>PreviewLayer</b><small>配信中も表示</small></div>
    <div class="hls"><span>ライブ配信</span><b>H.264 / AAC → fMP4</b><small>約2秒ごとにupload</small></div>
    <div class="local"><span>ローカル保存</span><b>HEVC / AAC → MP4</b><small>MomentNowのみ · 1080 × 1920</small></div>
  </div>
</div>

<div class="local-recording-result">
  <span>recording.mp4</span><i>→</i><b>元動画としてupload</b><i>+</i><b>設定時は写真ライブラリへ保存</b>
</div>

<div class="bottom-claim">公開サンプルはPreview + HLS。MomentNowは保存用Writerも並行する</div>

<!--
[Optional detail: 時間が厳しい場合は省略]

PreviewはAVCaptureVideoPreviewLayerへ同じCaptureSessionを接続するため、配信中も画面表示を続けられます。
MomentNowでは同じ元のCMSampleBufferを、HLS用とローカルMP4用の2つのAVAssetWriterへ分岐します。
停止時に両Writerをfinishし、ローカルMP4は元動画として別途uploadします。設定が有効なら写真ライブラリへも保存します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/CameraPreviewView.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveStreamView.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveSegmentRecorder.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveStreamViewModel.swift
-->

---

<div class="kicker">DATA OUTPUT DELEGATE</div>

# delegateを設定すると、CMSampleBufferが届き始める

<div class="delegate-setup-layout">

```swift
session.addOutput(videoOutput)
session.addOutput(audioOutput)

videoOutput.setSampleBufferDelegate(
    self, queue: writingQueue
)
audioOutput.setSampleBufferDelegate(
    self, queue: writingQueue
)
```

<div class="delegate-callback-flow">
  <div><b>VideoDataOutput</b><small>約1 frame</small></div>
  <div><b>AudioDataOutput</b><small>短いaudio block</small></div>
  <i>↓</i>
  <code>serial writingQueue</code>
  <i>↓</i>
  <strong>captureOutput(_:didOutput:from:)</strong>
</div>

</div>

<div class="bottom-claim">VideoとAudioを同じserial queueで受け取り、Writerの状態を1か所で更新する</div>

<!--
DataOutputへdelegateとcallback queueを設定すると、撮影のたびにcaptureOutputが呼ばれます。
AppleのAPIはVideoのcallback queueにserial queueを要求し、frameの到着順を保証します。

公開サンプルはVideoとAudioへ同じwritingQueueを指定します。
そのため、Writerの開始、時刻補正、appendを同じserial queue上で順番に処理できます。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avcapturevideodataoutput/setsamplebufferdelegate(_:queue:)
- https://developer.apple.com/documentation/avfoundation/avcaptureaudiodataoutput/setsamplebufferdelegate(_:queue:)
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">ORIENTATION</div>

# 縦向きは、video connectionへ90度を指定

<div class="orientation-visual">
  <div class="landscape-frame">1280 × 720</div>
  <div class="rotate-arrow">↻ <span>90°</span></div>
  <div class="portrait-frame">720 × 1280</div>
</div>

```swift
if let connection = videoOutput.connection(with: .video),
   connection.isVideoRotationAngleSupported(90) {
    connection.videoRotationAngle = 90
}
```

<!--
[Optional detail: 時間が厳しい場合は省略]

出力サイズはportraitで指定し、video connectionへ90度のrotation angleを設定します。
配信中の向き変更は扱わず、portrait 90度で固定します。
-->

---

<div class="kicker">CONFIG</div>

# 配信開始前に、上り回線に合う単一品質を選ぶ

```swift
let profile: (CGSize, Int) = switch quality {
case .high:   (.init(width: 720, height: 1280), 2_500_000)
case .medium: (.init(width: 720, height: 1280), 1_500_000)
case .low:    (.init(width: 480, height: 854),    900_000)
}
```

<div class="config-rail">
  <div><b>network状態</b><span>constrained / expensive</span></div>
  <div><b>API応答</b><span>配信準備の待ち時間</span></div>
  <div><b>test upload</b><span>上り速度</span></div>
  <div><b>2.0 sec</b><span>segment target</span></div>
</div>

<div class="source">LiveSegmentRecorder.Config.resolved / AutoStreamingQualityResolver</div>

<div class="bottom-claim">開始前に1品質を選ぶ。配信途中の自動画質切替（ABR）は行わない</div>

<!--
[本編必須: 品質・ビットレートの選択]

個人アプリではNWPathと小さなprobe PUTから、high、medium、lowの1品質を配信開始前に選びます。
途中でrenditionを切り替えるABRではなく、端末の上り回線に合わせた開始時の選択です。
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
公開サンプルは説明しやすい1.5 Mbps固定です。個人アプリは上り回線を優先し、品質設定ごとに0.9、1.5、2.5 Mbpsから選びます。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveSegmentRecorder.swift
-->

---

<div class="kicker">VIDEO SETTINGS</div>

# 配信用videoフォーマットは、互換性と上り帯域を優先する

<div class="settings-table">
  <div class="settings-head"><span>KEY</span><span>VALUE</span><span>INTENT</span></div>
  <div><code>AVVideoCodecKey</code><b>H.264</b><span>広い再生互換性</span></div>
  <div><code>Width / Height</code><b>480p / 720p</b><span>quality別に選択</span></div>
  <div><code>AverageBitRate</code><b>0.9–2.5 Mbps</b><span>上り回線へ追従</span></div>
  <div><code>ProfileLevel</code><b>High Auto</b><span>encoder profile</span></div>
  <div><code>FrameReordering</code><b>false</b><span>decode順を単純化</span></div>
</div>

<div class="source">LiveSegmentRecorder.makeHLSVideoSettings()</div>

<!--
[Optional detail: 時間が厳しい場合は省略]

個人アプリはH.264 portraitで、品質に応じて480pまたは720p、0.9から2.5Mbpsを選びます。
端末保存品質ではなく、ネットワークへ継続的に送れる配信品質として設定します。
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
家族向けの短い映像という用途に合わせ、stereoより送信量を優先しています。
-->

---

<div class="kicker">URL-LESS HLS WRITER · iOS 26+</div>

# HLS用のAVAssetWriterを構成する

<div class="writer-setup-layout">

```swift
let writer = AVAssetWriter(contentType: .mpeg4Movie)
writer.outputFileTypeProfile = .mpeg4AppleHLS
writer.preferredOutputSegmentInterval = .init(
    seconds: 2, preferredTimescale: 600
)
writer.initialSegmentStartTime = startTimeOffset
writer.delegate = self
```

<div class="writer-segment-output">
  <b>AVAssetWriter</b>
  <i>→ delegate →</i>
  <div class="writer-data-stack"><span>initialization Data</span><span>separable Data</span></div>
</div>

</div>

<div class="bottom-claim">URLへ書き込まず、delegateへsegment Dataを出す</div>

<!--
このサンプルのWriterコードはiOS 26以上が対象です。
通常の録画では出力先URLを指定しますが、segment delegateを使う構成ではcontentTypeだけでWriterを作ります。
Apple HLS profile、希望segment間隔、initial start time、delegateを設定します。

contentTypeのmpeg4MovieはMP4というcontainerの種類、outputFileTypeProfileのmpeg4AppleHLSはHLS向けfragment出力の指定です。
名前は似ていますが競合する設定ではありません。

このdelegate methodを実装すると通常のファイル書き込みは抑止され、Writerがsegment Dataをcallbackします。
各propertyの意味と2秒境界はChapter 03で詳しく見ます。
preferredTimescaleに 600 を指定した場合、1秒は 600/600 となり、1/600秒単位の細かい時間を表現できるようになります。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriter
- https://developer.apple.com/documentation/avfoundation/avfiletypeprofile/mpeg4applehls
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">INPUT → RECEIVER → WRITER · iOS 26+</div>

# CMSampleBufferの入口はReceiver

<div class="receiver-setup-layout">

```swift
let videoInput = AVAssetWriterInput(
    mediaType: .video,
    outputSettings: videoSettings()
)
self.videoReceiver = writer.inputReceiver(
    for: videoInput
)
```

<div class="receiver-connection">
  <div class="receiver-row video"><span>Video / H.264</span><b>videoReceiver</b></div>
  <div class="receiver-row audio"><span>Audio / AAC</span><b>audioReceiver</b></div>
  <i>↓ Inputを接続 ↓</i>
  <strong>AVAssetWriter</strong>
</div>

</div>

<div class="bottom-claim">Videoを代表例として表示。Audioも同じ手順でReceiverを保持する</div>

<!--
AVAssetWriterInputには、VideoならH.264、AudioならAACなどのencode設定を渡します。
inputReceiver(for:)は、そのInputをWriterへ接続すると同時に、CMSampleBufferを書き込むReceiverを返します。
画面ではVideo側を示しています。Audio側もmediaTypeとoutputSettingsを替え、同じ手順でaudioReceiverへ保持します。

これはiOS 26以上のSampleBufferReceiver APIです。
従来のwriter.add(input)とinput.append(sampleBuffer)に相当する接続と書き込みを、Receiver経由で行います。

Inputはsetup中のローカル変数で十分です。
一方ReceiverはcaptureOutputが呼ばれるたびに使うため、HLSSegmentRecorderのpropertyとして保持します。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriter
- https://developer.apple.com/documentation/avfoundation/avassetwriterinput/samplebufferreceiver
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">CMSAMPLEBUFFER ≠ SEGMENT</div>

# 多数のCMSampleBufferを、約2秒のsegmentへまとめる

<div class="sample-to-segment-flow">
  <div class="sample-flow-stack inputs">
    <div class="sample-flow-node video"><span>Camera</span><b>video CMSampleBuffer</b><small>約1 frame · 30fpsなら約33ms</small></div>
    <div class="sample-flow-node audio"><span>Microphone</span><b>audio CMSampleBuffer</b><small>短い音声block</small></div>
  </div>
  <div class="sample-flow-arrow">→</div>
  <div class="sample-flow-writer"><span>AVAssetWriter</span><b>圧縮<br>+ 時間順に格納<br>+ 約2秒に分割</b></div>
  <div class="sample-flow-arrow">→</div>
  <div class="sample-flow-stack outputs">
    <div class="sample-flow-node init"><span>1配信に1つ</span><b>initialization</b><small>圧縮方式 · 映像/音声の構成</small></div>
    <div class="sample-flow-node media"><span>約2秒ごと</span><b>separable</b><small>video frame + audio block本体</small></div>
  </div>
</div>

<div class="sample-term-grid">
  <div><b>video CMSampleBuffer</b><span>カメラから届く約1 frame分</span></div>
  <div><b>audio CMSampleBuffer</b><span>マイクから届く短い音声block</span></div>
  <div><b>separable Data</b><span>約2秒分の映像・音声本体</span></div>
  <div><b>initialization Data</b><span>再生開始に必要な設定</span></div>
</div>

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
  <div class="guard-input">captureOutput(_:didOutput:from:)</div>
  <div class="guard-step"><code>isWriting == true</code><span>配信中だけ</span></div>
  <div class="guard-step"><code>CMSampleBufferDataIsReady</code><span>利用可能なbufferだけ</span></div>
  <div class="guard-output">startWriterIfNeeded → append</div>
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

<div class="kicker">CAPTURE CALLBACK → RECEIVER · iOS 26+</div>

# 1回のcallbackで、1つのCMSampleBufferをReceiverへ渡す

<div class="append-pipeline">
  <div><span>DataOutput delegate</span><b>captureOutput</b></div>
  <i>→</i>
  <div><span>Writerへ渡す前</span><b>時刻を補正</b></div>
  <i>→</i>
  <div><span>CoreMedia</span><b>CMReadySampleBuffer</b></div>
  <i>→</i>
  <div class="primary"><span>SampleBufferReceiver</span><b>appendImmediately</b></div>
</div>

```swift
let readySampleBuffer = CMReadySampleBuffer(
    unsafeBuffer: transferableSampleBuffer
)
let didAppend = try receiver.appendImmediately(readySampleBuffer)
guard didAppend else { return }
```

<div class="append-outcomes">
  <div class="success"><b>true</b><span>Writerへ渡せた</span></div>
  <div class="warning"><b>false</b><span>待たずにdrop</span></div>
  <div class="danger"><b>throw</b><span>streamをerror終了</span></div>
</div>

<!--
captureOutputが呼ばれるたび、CMSampleBufferの時刻を補正し、CoreMediaの所有権を明示したCMReadySampleBufferへ変換します。
その後、VideoまたはAudioのReceiverへappendImmediatelyします。

appendImmediatelyは同期的に受け入れを試します。
trueならappend成功、falseならWriterがまだ受け入れられないため、そのCMSampleBufferを待たずに落とします。throwならWriter自体の失敗としてAsyncThrowingStreamをerrorで閉じます。
待つappendではなくappendImmediatelyを選ぶ理由は、Camera callbackへ古いframeを貯めないためです。
CMReadySampleBufferとappendImmediatelyもiOS 26以上のAPIです。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriterinput/samplebufferreceiver/appendimmediately(_:)
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">WRITER DELEGATE → FILE PATH</div>

# WriterのDataを、Publisherが保存する

<div class="fragment-routing">
  <div class="fragment-route init">
    <code>.initialization</code><i>→</i><b>HLSFragment.initialization</b><i>→</i><strong>init.mp4</strong>
  </div>
  <div class="fragment-route media">
    <code>.separable</code><i>→</i><b>HLSFragment.media</b><i>→</i><strong>seg/000001.m4s</strong>
  </div>
</div>

```swift
switch segmentType {
case .initialization:
    continuation.yield(.initialization(segmentData))
case .separable:
    segmentIndex += 1
    continuation.yield(.media(
        sequence: segmentIndex, data: segmentData, duration: duration
    ))
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

<div class="kicker">MINIMUM IMPLEMENTATION ORDER</div>

# HLS生成の8ステップ

<div class="generation-lifecycle">
  <div class="generation-phase">
    <h2>準備</h2>
    <div><b>01</b><span><strong>Captureを構成</strong>Camera / MicをSessionへ追加</span></div>
    <div><b>02</b><span><strong>DataOutputを接続</strong>delegateとserial queueを指定</span></div>
    <div><b>03</b><span><strong>Writerを構成</strong>HLS profileと約2秒の設定</span></div>
    <div><b>04</b><span><strong>Receiverを保持</strong>Video / Audio Inputを接続</span></div>
  </div>
  <div v-click class="generation-phase execution">
    <h2>実行・停止</h2>
    <div><b>05</b><span><strong>最初のVideoで開始</strong><code>start()</code> → <code>startSession</code></span></div>
    <div><b>06</b><span><strong>時刻を補正してappend</strong><code>appendImmediately</code></span></div>
    <div><b>07</b><span><strong>delegateでDataを受信</strong>initialization → separable</span></div>
    <div><b>08</b><span><strong>停止時にflush</strong><code>finishWriting</code>で最後のData</span></div>
  </div>
</div>

<div class="bottom-claim">05・06では、最初の映像を基準にWriterへ渡す時刻を調整する</div>

<!--
ここまでの実装順を、準備と実行・停止に分けてまとめます。
CaptureSessionとDataOutput、HLS設定済みWriter、InputとReceiverを用意します。
クリックで右側の実行・停止を表示します。
最初のvideo frameでWriterのsessionを開始し、同じ時刻補正をVideoとAudioへ適用してappendします。
Writer delegateから最初にinitialization Data、その後にseparable Dataが届きます。
停止時はfinishWritingまで待つことで、最後の短いsegmentも受け取れます。

このうち5番と6番の時刻の扱いを、次の数値例で確認します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---
class: timing-overview
---

<div class="kicker">WRITER TIMING</div>

# Writerへ渡す時刻の調整

<p class="timing-intro">時刻の扱いは、Apple公式サンプルの方針を採用</p>
<p class="timing-context">最初の映像を10秒に置き、映像と音声を同じ量だけずらす</p>

<table class="timing-comparison">
  <caption>両方から90秒を引いても、0.02秒の時間差は変わらない</caption>
  <thead><tr><th></th><th>調整前の時刻</th><th>Writerへ渡す時刻</th></tr></thead>
  <tbody>
    <tr><th>映像</th><td>100.00<span>秒</span></td><td>10.00<span>秒</span></td></tr>
    <tr><th>音声</th><td>100.02<span>秒</span></td><td>10.02<span>秒</span></td></tr>
  </tbody>
</table>

<div class="bottom-claim warning">今回の検証：調整を外すと、生成ファイルに映像・音声が入らなかった</div>
<p class="timing-detail-note">Writerの開始位置は10秒のまま、時刻調整だけを外した場合。詳細は補足へ</p>

<div class="source">Apple WWDC20: Author fragmented MPEG-4 content with AVAssetWriter</div>

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

<div class="kicker">HLS-SPECIFIC · FOUR KNOBS</div>

# 今回のHLS Writerで使う、4つの設定

<div class="setting-list">
  <div><b>01</b><code>outputFileTypeProfile</code><span>Apple HLS向けfMP4</span></div>
  <div><b>02</b><code>preferredOutputSegmentInterval</code><span>希望するsegment間隔</span></div>
  <div><b>03</b><code>initialSegmentStartTime</code><span>最初のsegment開始時刻</span></div>
  <div><b>04</b><code>delegate</code><span>生成されたDataの受け口</span></div>
</div>

<div class="source">HLSSegmentRecorder.setupWriterLocked()</div>

<!--
この4つが、ファイルURLを持たないHLS用Writerの核です。
特にintervalは「必ず」ではなく「preferred」です。
-->

---

<div class="kicker">PREFERRED INTERVAL</div>

# 2秒ぴったりではなく、2秒付近のIDRで切る

<div class="segment-boundary-demo">
  <div class="boundary-row preferred">
    <b>希望</b>
    <span>0.00s</span><i></i><span>2.00s</span><i></i><span>4.00s</span>
  </div>
  <div class="boundary-row actual">
    <b>実際</b>
    <span>IDR · 0.00s</span><i></i><span>IDR · 2.03s</span><i></i><span>IDR · 4.00s</span>
  </div>
</div>

<div class="actual-duration-flow">
  <div><small>segment 1</small><b>2.03s</b></div>
  <div><small>segment 2</small><b>1.97s</b></div>
  <span>→</span>
  <div class="playlist-duration"><small>playlist</small><b>実durationをEXTINFへ</b></div>
</div>

<div class="term-definition compact"><b>IDR</b><span>ほかのframeを参照せず、そこから単独で再生を始められるkeyframe</span></div>

<div class="bottom-claim warning">2秒は設定値。playlistにはAVAssetSegmentReportの実durationを書く</div>

<!--
preferredOutputSegmentIntervalのproperty名どおり、2秒は希望する間隔です。
HLSのvideo segmentは、途中のframeを参照せず単独でデコードを始められるIDR frameから開始する必要があります。
そのため希望位置が2.00秒でも、実際のIDRが2.03秒なら、segment境界は2.03秒まで前後します。
この例では1本目が2.03秒、次が1.97秒です。常に2.0秒固定とは限りません。
playlistのEXTINFには設定値の2秒ではなく、AVAssetSegmentReportが返すvideo trackの実durationを書きます。
サンプルではreportが取得できない、またはdurationが無効な場合だけ設定値の2秒へfallbackします。
次にplaylistへ書く実durationを確認し、その後で希望位置の近くにIDRを用意するencoder設定を見ます。

[Sources]
- Apple: AVAssetWriter.preferredOutputSegmentInterval
- Apple HLS Authoring Specification for Apple Devices
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLS/HLSManifest.swift
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

# Appleは、約2秒ごとのIDRを推奨

<div class="gop-timeline">
  <div class="frame-row">
    <span class="i-frame">I</span><span>P</span><span>P</span><span>P</span>
    <span class="i-frame">I</span><span>P</span><span>P</span><span>P</span>
    <span class="i-frame">I</span>
  </div>
  <div class="gop-brackets">
    <div>segment 1 · ~2s</div>
    <div>segment 2 · ~2s</div>
  </div>
</div>

```swift
AVVideoMaxKeyFrameIntervalDurationKey: config.segmentSeconds
```

<div class="term-definition"><b>IDR</b><span>ほかのframeを参照せず、そこから再生を始められるkeyframe</span></div>

<div class="source">Apple HLS Authoring Specification 1.13: IDRは2秒ごとを推奨</div>

<!--
Playerがsegmentの先頭からデコードするにはIDRが必要です。
segment targetと同じ2秒でmax keyframe interval durationを指定し、境界を作れるようにします。
AppleのHLS Authoring Specification 1.13にも「Key frames (IDRs) SHOULD be present every two seconds」とあります。
MUSTではなくSHOULDなので絶対必須ではありませんが、Apple端末向けHLSで特別な理由がなければ従う推奨です。
これはsegmentを必ず2.000秒にするという意味ではなく、約2秒ごとにIDRを用意するというencoder側の要件です。

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

<div class="kicker">SERVER · ATOMIC REPLACE</div>

# PUT中のファイルを、Viewerへ見せない

<div class="atomic-replace">
  <div><b>1 temporary file</b><span>request bodyを書き込む</span></div>
  <i>→</i>
  <div><b>2 ディスクへ書き切る</b><span>flush + fsync</span></div>
  <i>→</i>
  <div class="hot"><b>3 os.replace</b><span>完成品へ一気に置換</span></div>
</div>

<div class="bottom-claim">playlistの途中状態や、半分だけのm4sをGETさせない</div>

<!--
[Optional detail: 時間が厳しい場合は省略]

Macサーバーはrequest bodyをdestinationへ直接書きません。
同じdirectoryの一時ファイルへ書き切り、fsyncした後にos.replaceします。Viewerには古い完成品か新しい完成品だけが見えます。

[Sources]
- iosdc2026HLSSample/server/server.py
-->

---

<div class="kicker">VISIBLE OUTPUT</div>

# 再生開始は、init保存と最初のsegment公開の後

<div class="output-clock">
  <div class="clock-column"><span>start</span><b>init Data</b></div>
  <div class="clock-column"><span>network</span><b>PUT init</b></div>
  <div class="clock-column active"><span>~2s</span><b>PUT seg 1</b></div>
  <div class="clock-column active"><span>after PUT</span><b>playlist更新</b></div>
</div>

<div class="playable-equation">
  <span>init.mp4<br><small>保存済み</small></span><i>+</i>
  <span>000001.m4s<br><small>保存済み</small></span><i>+</i>
  <span>playlist<br><small>segment 1を掲載済み</small></span><i>=</i>
  <b>Viewer<br>再生開始</b>
</div>

<!--
AVAssetWriterがDataを返しただけでは、まだ視聴者は再生できません。
initと最初のm4sが保存先に存在し、playlistへ最初のsegmentが載った時点が最初の再生可能状態です。
-->

---

<div class="kicker">PUBLIC SAMPLE · LOCAL PLAYLIST</div>

# playlist公開後に、アプリ内の状態を確定する

```swift {1-3|4-6}
var nextManifest = manifest
nextManifest.addSegment(seq: seq, durationSec: durationSec)

try await client.putPlaylist(streamId: streamId, text: nextManifest.text)
manifest = nextManifest
```

<div class="transaction-visual">
  <div><span>current</span><b>segments 1...N</b></div>
  <i>copy</i>
  <div><span>candidate</span><b>+ segment N+1</b></div>
  <i>HTTP成功</i>
  <div class="hot"><span>状態を確定</span><b>current = candidate</b></div>
</div>

<div class="bottom-claim">seqはsegment番号。iPhoneはplaylist本文を更新し、PUT成功後に状態を確定する</div>

<!--
サンプルのHLSManifestはseqをkeyにしてsegmentを保持し、表示時には必ず番号順にします。
内部状態を先に進めません。次のmanifestをコピーで作り、そのPUTが成功した後だけcurrentへ代入します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSManifest.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HTTPHLSClient.swift
-->

---

<div class="kicker">PRODUCTION PUBLISH FLOW</div>

# 署名付きURL → S3 PUT → playlist公開

<div class="settings-table">
  <div class="settings-head"><span>APPの操作</span><span>MomentNow</span><span>結果</span></div>
  <div><b>S3への署名付きURLを受け取る</b><code>POST /presign</code><span>presigned PUT URL</span></div>
  <div><b>segment本体を保存する</b><code>PUT → S3</code><span>objectが存在する</span></div>
  <div><b>playlistへ載せる</b><code>POST /commit</code><span>Viewerから見える</span></div>
</div>

<div class="bottom-claim">S3は保存先の選択。HLSで重要なのは「保存してから公開」</div>

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

# サーバーが、S3へPUTするための<br>署名付きURLを発行する

<div class="choice-compare">
  <div class="choice muted"><span>DO NOT</span><b>AWS credentialを内包</b><small>漏えい範囲が広い</small></div>
  <div class="choice-arrow">→</div>
  <div class="choice selected"><span>PRESIGNED URL</span><b>S3へ1ファイルだけPUT</b><small>保存先 / 有効期限を署名</small></div>
</div>

<div class="bottom-claim">iPhoneは発行されたURLへDataをPUTするだけ。AWS credentialは持たない</div>

```swift
var request = URLRequest(url: presignedURL)
request.httpMethod = "PUT"
request.setValue(contentType, forHTTPHeaderField: "Content-Type")
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

<div class="kicker">PUBLISH AFTER BYTES</div>

# PUT成功より先に、playlistへseqを出さない

<div class="order-contrast">
  <div class="bad-order">
    <span>BAD</span>
    <div>commit</div><b>→</b><div>Viewer GET</div><b>→</b><div class="error">404</div>
  </div>
  <div class="good-order">
    <span>GOOD</span>
    <div>S3 PUT完了</div><b>→</b><div>commit</div><b>→</b><div>playlist更新</div>
  </div>
</div>

<!--
順序が重要です。
PUTの2xxを確認してからcommitします。逆なら、Playerがplaylistで見つけたURIをGETして404になります。
公開サンプルは同じファイルを最大3回まで再送し、それでも失敗した場合はplaylist更新へ進まずerrorとして残します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---

<div class="kicker">PUBLIC SAMPLE · RETRY POLICY</div>

# サンプルは、同じファイルを<br>最大3回まで再送する

<div class="retry-steps">
  <div><b>attempt 1</b><span>失敗</span><small>250 ms</small></div>
  <i>→</i>
  <div><b>attempt 2</b><span>失敗</span><small>500 ms</small></div>
  <i>→</i>
  <div><b>attempt 3</b><span>成功 / error</span><small>終了</small></div>
</div>

<div class="bottom-claim warning">segment PUTが失敗したら、そのsegmentをplaylistへ載せない</div>

<!--
[本編必須: アップロード失敗時の扱い]

サンプルアプリは各PUTを最大3回試します。
segment保存が3回とも失敗した場合、playlist PUTへ進まず、Publisherのerrorとして残します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSampleTests/HLSStreamPublisherTests.swift
-->

---

<div class="kicker">OUT-OF-ORDER RACE</div>

# upload完了順と、再生順は一致しないことがある

<div class="race-lanes">
  <div class="race-time-axis">時間 →</div>
  <div class="race-row"><b>seg 1</b><div class="race-track"><span class="race-bar medium">upload 1</span><em>commit 1</em></div></div>
  <div class="race-row"><b>seg 2</b><div class="race-track"><span class="race-bar fast">upload 2</span><em class="early">commit 2</em></div></div>
  <div class="race-row"><b>seg 3</b><div class="race-track"><span class="race-bar slow">upload 3</span><em>commit 3</em></div></div>
</div>

<div class="order-equation">
  <span>completion</span><code>2, 1, 3</code><b>≠</b><span>playback</span><code>1, 2, 3</code>
</div>

<div class="publication-status">
  <p><b>MomentNowの現状</b><span>sequence順へ整列。欠番は待たない</span></p>
  <p><b>改善案 · 未実装</b><span>欠番を待ち、連続した範囲だけ公開</span></p>
</div>

<!--
[Optional detail: 時間が厳しい場合は省略]

upload処理は待機中に完了順が入れ替わる可能性があります。
図は左から右を時間とし、seg 2、seg 1、seg 3の順でPUTが完了する例です。各commitは、そのsegmentのPUT完了後に始めます。
iOSはseqを必ず送ります。現在のcommit APIはplaylist内のsegmentをseq順へ並べ直しますが、欠番を待つ契約ではありません。
より安全にするなら、サーバーが連続したseqまでの「公開済み境界」を管理します。

[Sources]
- MomentNow-Lambda/src/commit.ts
-->

---

<div class="kicker">SERVER · OBJECT SEMANTICS</div>

# cache方針は、playlistとsegmentで分ける

<div class="cache-table server-cache-table">
  <div class="cache-head"><span>OBJECT</span><span>CACHE</span><span>HTTP</span></div>
  <div class="dynamic"><b>playlist.m3u8</b><span>no-store / no-cache</span><code>毎回最新を取得</code></div>
  <div class="immutable"><b>init.mp4 / m4s</b><span>max-age=31536000</span><code>Range / 206（部分取得）</code></div>
</div>

<div class="bottom-claim compact">S3 / CloudFrontでの設定</div>

<!--
[本編必須: キャッシュ制御]

playlistは同じURLの内容が増えるためキャッシュさせません。
initとm4sは一度置いたら変えず長期cacheし、PlayerのRange requestには206で返します。個人アプリもobjectの可変性で方針を分けます。

[Sources]
- iosdc2026HLSSample/server/server.py
- MomentNow-Lambda/src/create_stream.ts
- MomentNow-Lambda/src/presign.ts
-->

---
layout: center
class: chapter
---

<div class="chapter-no">05</div>
<div class="chapter-rule"></div>

# 生成を止めず、<br>配信の完了まで待つ
<p>AVFoundation callback → Swift Task → final playlist</p>

<!--
[Timing checkpoint: 30:00]

最後に、iOS上のcallback、Task、URLSessionを1本のライブ配信として整理します。
焦点は、メディア処理をHTTP待ちから切り離す境界です。
Writerが受け取れない問題と、生成済みsegmentが送信待ちになる問題を分けて説明します。
-->

---

<div class="kicker">CALLBACK TO TASK</div>

# メディア処理を、HTTP待ちから切り離す

<div class="async-boundary-flow">
  <div class="async-boundary-stage">
    <span>WRITING QUEUE · CALLBACK</span>
    <b>WriterのDataを受け取る</b>
    <small>initialization / mediaを受け取り、すぐcallbackを返す</small>
    <code>HLSSegmentRecorder</code>
  </div>
  <i>→</i>
  <div class="async-boundary-stage bridge">
    <span>BRIDGE</span>
    <b>Dataを順番に受け渡す</b>
    <small>同期callbackを、非同期で読める列へ変換</small>
    <code>AsyncThrowingStream&lt;HLSFragment&gt;</code>
  </div>
  <i>→</i>
  <div class="async-boundary-stage task">
    <span>SWIFT TASK</span>
    <b>HTTPをawaitする</b>
    <small>PUT、playlist更新、retryを順番に実行</small>
    <code>HLSStreamPublisher</code>
  </div>
</div>

<div class="async-boundary-types">
  <span><b>HLSFragment</b> = init / mediaのDataを表す値</span><i>·</i>
  <span><b>AsyncThrowingStream</b> = callbackとTaskをつなぐ通路</span>
</div>

<div class="bottom-claim warning">サンプルの送信待ちに上限はない。送信が生成より遅いとsegmentが滞留する</div>

<!--
AVAssetWriterDelegateはwritingQueue上で呼ばれます。
callbackではHLSFragmentをAsyncThrowingStreamへ渡してすぐ戻り、次のCMSampleBuffer処理を止めません。
HLSFragmentはinitializationまたはmediaのDataと付随情報を表す、サンプルアプリ内の値です。
AsyncThrowingStreamは同期callbackから届く値を、Task側がfor awaitで順番に読める形へ変換します。
HTTP処理はHLSStreamPublisherのTaskでawaitします。
AsyncThrowingStreamは既定の無制限bufferで作り、Publisherは各fragmentのHTTP処理を順番に待っています。
この分離でCapture callbackのHTTP待ちは避けられますが、継続的に上り回線が遅いと送信待ちDataと遅延が増えます。
次のページではWriter側のdropと、送信側に残る改善課題を区別します。
-->

---

<div class="kicker">BACKPRESSURE DECISION</div>

# Writerの詰まりと、送信待ちは別の問題

<div class="drop-timeline">
  <div class="drop-lane"><b>Capture</b><span>video 1</span><span>audio 1</span><span>video 2</span><span>audio 2</span></div>
  <div class="drop-lane writer"><b>Writer</b><span class="append">append</span><span class="append">append</span><span class="busy">busy</span><span class="dropped">drop</span></div>
</div>

<div class="drop-choice">
  <div class="wait-choice"><span>Writer側 · 実装済み</span><b>受け取れないCMSampleBufferをdrop<br>映像のカクつき・音声の欠けを許容</b></div>
  <div class="drop-choice-current"><span>送信側 · 今後の改善</span><b>送信待ち量に上限を設ける<br>上限超過時の停止判断を追加</b></div>
</div>

<div class="bottom-claim warning">Writer側のdropでは、生成済みsegmentの送信待ちは減らない</div>

<!--
VideoとAudioのDataOutputは一定間隔でCMSampleBufferをpushし続けます。
Writerが受け取れずappendImmediatelyがfalseを返した場合、この実装ではbufferを保留せずreturnします。
Writer側では受け入れ待ちのbufferを保留せず、映像のカクつきや音声の欠けを許容しています。
これは生成前のCMSampleBufferに対する判断で、生成済みsegmentの送信待ちは減らしません。
サンプルには送信待ち量の上限や上限超過時の停止判断は未実装です。今後の改善として分けて示します。
ライブ遅延全体や継続的な帯域不足を、このdropだけで解消できるわけではありません。
appendImmediatelyがthrowした場合はWriterの失敗としてstreamをerrorで閉じます。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriterinput/samplebufferreceiver/appendimmediately(_:)
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">WRITER FINISH</div>

# 最後に渡した時刻で、Writerを終了する

<div class="stop-flow">
  <div><b>lastAdjustedPTS</b><span>最後にappendした時刻</span></div>
  <i>→</i>
  <div><b>endSession</b><span>media rangeを閉じる</span></div>
  <i>→</i>
  <div><b>markAsFinished</b><span>video / audio</span></div>
  <i>→</i>
  <div><b>finishWriting</b><span>最後のsegmentをflush</span></div>
</div>

```swift
if lastAdjustedPTS.isValid {
    writer.endSession(atSourceTime: lastAdjustedPTS)
}
```

<!--
[Optional detail: 時間が厳しい場合は省略]

終了時刻も補正後のPTSを使います。
finishWritingによって最後のsegmentがdelegateへ届く可能性があるため、stopはその完了まで待ちます。
-->

---

<div class="kicker">DRAIN BEFORE FINISH</div>

# 撮影停止後も、最後のsegment公開まで待つ

<div class="drain-lanes">
  <div><b>1 Recorder</b><span>capture停止 → finishWriting</span><small>最後のfragmentが出る</small></div>
  <i>→</i>
  <div><b>2 Stream</b><span>continuation.finish()</span><small>fragment列を閉じる</small></div>
  <i>→</i>
  <div class="hot"><b>3 Publisher</b><span>uploadTask.value</span><small>ENDLIST PUTまで待つ</small></div>
</div>

<div class="bottom-claim">「撮影停止」と「配信完了」は、同じ瞬間ではない</div>

<!--
SampleHLSStreamer.stopRecordingの順序です。
Recorder.stopがcapture、Writer、streamを順に閉じ、PublisherがENDLIST付きplaylistをPUTするまでTaskを待ちます。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleHLSStreamer.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---

<div class="kicker">PUBLIC DEMO → PRODUCTION</div>

# MomentNowで採用した4つの運用制御

<div class="production-additions">
  <div><b>AUTH</b><span>user / group ownership</span></div>
  <div><b>PRESIGN</b><span>S3の1ファイル限定・署名付きURL</span></div>
  <div><b>COMMIT</b><span>公開順とplaylistの同時更新を制御</span></div>
  <div><b>DELIVERY</b><span>CloudFront / ticket / status</span></div>
</div>

<div class="bottom-claim">commit APIは必須ではない。サンプルはiOSでplaylistと公開順を管理する</div>

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

<div class="kicker">FIT, NOT UNIVERSAL</div>

# この構成が効くのは、用途を絞ったとき

<div class="fit-spectrum">
  <div class="fit-side good">
    <span>GOOD FIT</span>
    <b>少人数・短時間・単一品質</b>
    <small>新しいiPhone / 安定した上り回線</small>
  </div>
  <div class="fit-scale"><i></i></div>
  <div class="fit-side bad">
    <span>USE A MEDIA PLATFORM</span>
    <b>大規模・長時間・複数画質の自動切替</b>
    <small>厳しい可用性要件にも対応したい</small>
  </div>
</div>

<div class="bottom-claim">「端末でできる」と「端末でやるべき」は別の判断</div>

<!--
この構成は万能ではありません。
少人数・短時間・単一品質には合います。
大規模、長時間、複数画質を回線に合わせて自動切替するABR、厳しい可用性要件があるなら専用のmedia platformを選びます。
-->

---
layout: center
class: closing
---

<div class="kicker">TAKEAWAYS</div>

# 端末内で生成したHLSを、撮影中から公開する

<div class="takeaway-grid">
  <div><b>01</b><span><strong>生成 · Writer</strong>Camera / Micの入力からHLS用Dataを作る</span></div>
  <div><b>02</b><span><strong>区切り · Boundary</strong>IDRとsegment intervalをそろえる</span></div>
  <div><b>03</b><span><strong>公開 · Upload</strong>Dataを保存してからplaylistへ載せる</span></div>
  <div><b>04</b><span><strong>終了 · Finish</strong>全送信とENDLISTの公開を待つ</span></div>
</div>

<div class="takeaway-path"><b>端末内生成</b><span>AVCaptureSession → DataOutput → Receiver → AVAssetWriterDelegate</span></div>

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
    サンプルアプリのコード<br>https://github.com/HikaruSato/iosdc2026HLSSample
  </div>
</div>

<!--
ありがとうございました。
サンプルアプリは、このURLで公開しています。
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

# 10秒は、音声圧縮のための余白

<ul>
  <li>AACは圧縮の都合で、先頭に準備用の音声（priming）を加える</li>
  <li>Apple HLSでは、その分だけ音声の時刻を前へずらして補償する</li>
</ul>

<p class="lead">映像・音声の開始位置を後ろへずらし、<br>音声の時刻が負にならないようにする</p>

```swift
private let startTimeOffset = CMTime(value: 10, timescale: 1)
writer.initialSegmentStartTime = startTimeOffset
```

<div class="bottom-claim">10秒はAppleが示す設定例。再生開始まで10秒待つ意味ではない</div>
<div class="source">Apple WWDC20: Author fragmented MPEG-4 content with AVAssetWriter（16:16以降）</div>

<!--
AAC encoderは、正しくencode / decodeするために先頭へprimingを加えます。
Apple HLS profileはedit listを使わず、音声のbaseMediaDecodeTimeをpriming分だけ前へ移して補償します。
この値は符号なし整数なので、負にできません。そのためAppleは両方のmedia timeを同じ量だけ後ろへ移すことを勧めています。
initialSegmentStartTimeも同じ開始位置に合わせます。
10秒はHLS仕様の固定値ではありません。Appleのmediafilesegmenterと同じ値を選べる、という説明に合わせています。
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

# 時刻補正は、全サンプルの平行移動

<div class="formula-large">
  <span>delta</span>
  <b>=</b>
  <span class="formula-expression">10s − firstVideoPTS</span>
</div>

<div class="formula-large secondary">
  <span>adjustedPTS</span>
  <b>=</b>
  <span class="formula-expression">sourcePTS + delta</span>
</div>

<div class="code-caption">PTS：映像や音声を提示する時刻。移動量は最初の映像で一度だけ決める</div>

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

# 映像・音声は変えず、時刻情報だけを補正

```swift
let timingInfos = try sampleTimingInfos().map { info in
    var adjusted = info
    adjusted.presentationTimeStamp =
        info.presentationTimeStamp + offset
    if info.decodeTimeStamp.isValid {
        adjusted.decodeTimeStamp = info.decodeTimeStamp + offset
    }
    return adjusted
}
let copied = try CMSampleBuffer(
    copying: self, withNewTiming: timingInfos
)
```

<div class="code-caption">PTSと、有効なDTS（decode timestamp）を同じ量だけ動かす</div>
<div class="source">HLSSegmentRecorder.swift: offsettingTiming（時刻コピー部分の抜粋）</div>

<!--
CMSampleBufferの映像・音声データはそのままに、timing infoを差し替えたコピーを作ります。
PTSだけでなく、frameをdecodeする時刻であるDTSも、有効な場合は同じ量だけ補正します。
サンプルのoffsettingTimingでは、この後にoutputPresentationTimeStampも同じ量だけ補正しています。
ここは処理の抜粋で、エラー処理とoutput PTSの更新は省略しています。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->
