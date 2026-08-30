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
      <p>MomentNow という「今この瞬間」の動画をHLSで配信し、<br>URLで共有できる iOSアプリ を個人開発してます</p>
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
  <div><b>01</b><span>HLSがライブになる仕組み</span></div>
  <div><b>02</b><span>端末内でHLSを生成</span></div>
  <div><b>03</b><span>映像と音声の時間軸調整</span></div>
  <div><b>04</b><span>約2秒のfMP4を生成</span></div>
  <div><b>05</b><span>playlistの更新</span></div>
  <div><b>06</b><span>配信の完了</span></div>
</div>

<!--
最初にHLSがライブになる仕組みを確認し、動くサンプルアプリを見ます。
その後、カメラとマイクの入力、時刻補正、fMP4生成、保存とplaylist公開、停止時の完了管理まで順番に追います。
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
[Timing checkpoint: 00:45]

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
  <span><code>EXT-X-MAP</code><b>最初に読む設定ファイル</b></span>
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

<div class="bottom-claim">Playerは同じplaylist.m3u8を繰り返しGETする</div>

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
    <div class="arch-node">API + S3<small>保存 / playlist更新</small></div><div class="arch-arrow">→</div>
    <div class="arch-node">CloudFront</div><div class="arch-arrow">→</div>
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

<div class="kicker">WHY</div>
<div class="story-split">
  <div>
    <h1>1ファイルだと撮影後、共有するためのアップロードに時間がかかる</h1>
    <p class="lead">撮影中からアップロードできていれば共有も楽</p>
  </div>
  <div class="why-files">
    <div class="why-file large"><span>撮影終了後</span><b>recording.mov</b><small>大きな1ファイルを送信</small></div>
    <div class="why-divider">↓</div>
    <div class="why-segments">
      <span>撮影中</span>
      <b>init.mp4</b><b>001.m4s</b><b>002.m4s</b>
      <small>小さく区切って順次送信</small>
    </div>
  </div>
</div>

<!--
この実装のきっかけは、子どもの動画を家族へ送るときの待ち時間でした。
撮影し終わってから大きなファイルを送るのではなく、撮影中から小さく届けたい。それがHLSを選んだ理由です。

個人開発では、映像変換サーバーの常時運用が重い。
そこで用途を絞り、カメラを持っているiPhone自身にエンコードとセグメント生成を任せます。
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
  <div class="result-step"><b>S3へ保存する</b><span>一時PUT URL</span></div>
  <div class="result-arrow">→</div>
  <div class="result-step"><b>再生順を公開する</b><span>commit API</span></div>
</div>

<!--
録画開始後、init.mp4を1回、m4sを約2秒ごとに生成します。
各DataをS3へPUTし、成功したseqだけをcommitすると、視聴者のplaylistが伸びていきます。
-->

---

<div class="kicker">LIVE DEMO · PUBLIC SAMPLE</div>

# 仕組みが見えたところで、公開サンプルアプリを動かす

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
iPhoneはHLSを生成してMacへHTTP PUTし、MacのViewerは同じファイルをHTTP GETして追従再生します。

会場Wi-Fiが不安定な場合はライブ操作を省略し、続く6枚を静止画デモとして説明します。

[Sources]
- iosdc2026HLSSample/README.md
-->

---
class: demo-step
---

<div class="kicker">DEMO · 1 / 5</div>

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
デモ用サーバーを起動します。
ここには映像変換処理がありません。受け取ったHTTP bodyをファイルとして置き換え、GETで返すだけです。

配信開始の前に接続確認を押します。
HTTPHLSClientがhealth endpointへGETし、2xxなら接続済みにします。デモ中にURL誤りを早く見つけるための一手です。

[Sources]
- iosdc2026HLSSample/README.md
- iosdc2026HLSSample/server/server.py
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HTTPHLSClient.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/ContentView.swift
-->

---
class: demo-step
---

<div class="kicker">DEMO · 2 / 5</div>

# 「配信開始」で、iPhoneがHLS生成を始める

<div class="tap-to-bytes">
  <div class="tap-button">● 配信開始</div>
  <i>→</i>
  <div><b>Camera + Mic</b><span>video / audio CMSampleBuffer</span></div>
  <i>→</i>
  <div class="hot"><b>AVAssetWriter</b><span>init.mp4 + m4s</span></div>
</div>

<div class="demo-observe-line">
  <span>elapsed <b>0.0 →</b></span>
  <span>segment <b>0 →</b></span>
</div>

<!--
配信開始を押します。
カメラとマイクのCMSampleBufferをAVAssetWriterへ入れ、ファイルURLではなくdelegateからfMP4のDataを受け取ります。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleHLSStreamer.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---
class: demo-step
---

<div class="kicker">DEMO · 3 / 5</div>

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
delegateの最初のcallbackはinitializationです。
HLSStreamPublisherはinit.mp4の成功を覚え、これが済むまでmedia segmentを受け付けません。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---
class: demo-step
---

<div class="kicker">DEMO · 4 / 5</div>

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
約2秒ごとにmedia segmentができます。
大事なのは、m4sをPUTしてからplaylistを置き換える順番です。Playerが404になる参照を先に公開しません。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---
class: demo-step viewer-demo-step
---

<div class="kicker">DEMO · 5 / 5</div>

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

<div class="kicker">BIG PICTURE · FOUR BOUNDARIES</div>

# CameraからViewerまで、4段階で考える

<div class="four-boundaries">
  <div><b>01 取り込む</b><span>Camera / Mic → CMSampleBuffer</span></div>
  <i>→</i>
  <div><b>02 分割する</b><span>約2秒分 → init.mp4 / .m4s</span></div>
  <i>→</i>
  <div><b>03 保存する</b><span>ファイル本体 → Mac / S3</span></div>
  <i>→</i>
  <div><b>04 公開する</b><span>playlistへ追加 → Viewer</span></div>
</div>

<div class="bottom-claim">今回の難所は、各境界で「順序」を失わないこと</div>

<!--
[Timing checkpoint: 10:00]

大枠は4段階です。
Capture、fMP4化、オブジェクト公開、playlist追従再生。境界を越えるたびに、時刻か順序の整合性が必要になります。
-->

---

<div class="kicker">ONE FRAME'S JOURNEY</div>

# 1 frameは、約2秒のsegmentへ<br>まとめてから届ける

<div class="frame-journey">
  <div class="capture"><span>01 CALLBACK</span><b>1 video frame</b><small>CMSampleBuffer</small></div>
  <i>→</i>
  <div class="writer"><span>02 WRITER</span><b>約60 frames</b><small>+ audio blocks</small></div>
  <i>→</i>
  <div class="fragment"><span>03 FRAGMENT</span><b>000001.m4s</b><small>約2秒分</small></div>
  <i>→</i>
  <div class="storage"><span>04 STORAGE</span><b>PUT → 2xx</b><small>保存できた</small></div>
  <i>→</i>
  <div class="viewer"><span>05 PUBLISH</span><b>playlistへ追加</b><small>ViewerがGET</small></div>
</div>

<div class="bottom-claim">frame単体は送らない。約2秒分を保存してから、playlistで見えるようにする</div>

<!--
Cameraから届いた1つのvideo CMSampleBufferを、そのまま1ファイルとして送るわけではありません。
Writerへ順次appendし、ほかのvideo frameとaudio blockを約2秒分まとめます。30fpsなら、おおよそ60 frameです。

IDR境界でmedia segmentが確定すると、delegateから000001.m4sのDataが届きます。
PublisherはまずそのDataをstorageへ保存し、2xxを確認してからplaylistへURIを追加します。
Viewerが更新後のplaylistを取得した時点で、このframeを含むsegmentが再生対象になります。

init.mp4は、この流れより先に1回だけ保存します。

この5段階は後続の章に対応します。
callbackとTaskの境界、CMSampleBufferの時刻、fMP4への分割、storage保存とplaylist公開の順に掘り下げます。
-->

---

<div class="kicker">SAME SHAPE, DIFFERENT DESTINATION</div>

# 同じHLSを、MacまたはS3へ置く

<div class="environment-map">
  <div class="environment-row sample">
    <b>公開サンプル</b><span>iPhone</span><i>HTTP PUT</i><span>Mac filesystem</span><i>HTTP GET</i><span>Local Viewer</span>
  </div>
  <div class="environment-row production">
    <b>MomentNow</b><span>iPhone</span><i>presigned PUT</i><span>Amazon S3</span><i>CloudFront</i><span>Web Viewer</span>
  </div>
</div>

<div class="bottom-claim">init.mp4とm4sのバイト列は、どちらもiPhoneが生成する</div>

<!--
公開サンプルと個人アプリの対応です。
サンプルはMacのfilesystemとViewer、個人アプリはS3とCloudFrontです。端末が生成するHLSの構造は変わりません。

[Sources]
- iosdc2026HLSSample/README.md
- MomentNow-Lambda/AGENTS.md
-->

---

<div class="kicker">CONTROL PLANE</div>

# 公開タイミングは、playlistが制御する

<div class="control-plane">
  <div><b>segment created</b><span>まだ非公開</span></div>
  <i>→</i>
  <div><b>segment PUT 2xx</b><span>objectが存在</span></div>
  <i>→</i>
  <div class="hot"><b>playlist PUT / commit</b><span>Playerから見える</span></div>
  <i>→</i>
  <div><b>Viewer reload</b><span>次のsegmentを取得</span></div>
</div>

<div class="bottom-claim">保存と公開を分ける · playlistはno-cache、init / m4sはlong-cache</div>

<!--
control planeはplaylist更新です。
segment objectが存在することを確認してから、playlistへURIを追加します。これが、途中の404を防ぐ公開契約です。

同じURLを更新するplaylistはno-store / no-cacheにし、一度置いたら変えないinit.mp4とm4sは長期cacheします。

個人アプリではstreamIdごとのprefixへinitとm4sを置きます。
playlistはcommit APIが更新し、CloudFront経由のViewerは同じ相対URIをたどります。

個人アプリのcommit APIが生成する初期playlistです。
この時点ではmedia segmentがなくても、playlistの種類、target duration、initの場所は決まっています。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
- iosdc2026HLSSample/server/server.py
- MomentNow-Lambda/src/commit.ts
- MomentNow-Lambda/src/create_stream.ts
-->

---

<div class="kicker">FRAGMENTED MP4</div>

# fMP4は「設定」と「再生可能な断片」を分ける

<div class="fmp4-beginner-model">
  <div class="fmp4-init"><span>1配信に1つ</span><b>init.mp4</b><small>Playerが再生を始めるための設定</small></div>
  <i>+</i>
  <div class="fmp4-media"><span>約2秒ごと</span><b>000001.m4s</b><small>実際の映像と音声</small></div>
  <i>=</i>
  <div class="fmp4-playable"><span>playlistが結ぶ</span><b>再生可能</b><small>Playerはこの組み合わせを取得</small></div>
</div>


<div class="source">Apple WWDC20: Author fragmented MPEG-4 content with AVAssetWriter</div>

<!--
init.mp4にはftypとmoov、各m4sにはmoofとmdatが入ります。
AVAssetWriterDelegateは、この単位のDataを返してくれます。
-->

---

<div class="kicker">RESPONSIBILITY</div>

# AVAssetWriterはsegmentを作る。playlistは作らない。

<div class="ownership-split">
  <div class="ownership-side writer-side">
    <span>AVFoundation</span>
    <b>encode + segment</b>
    <ul>
      <li>H.264 / AAC</li>
      <li>init.mp4</li>
      <li>約2秒のmedia segment</li>
    </ul>
  </div>
  <div class="ownership-divider">/</div>
  <div class="ownership-side app-side">
    <span>App</span>
    <b>順番を付けて公開</b>
    <ul>
      <li>再生順番号（seq）</li>
      <li>S3 upload</li>
      <li>公開可能なseqを通知</li>
    </ul>
  </div>
</div>

<div class="source">AVAssetWriterDelegate / HLSSegmentRecorder / HLSStreamPublisher</div>

<!--
この責任分界が今日の中心です。
AppleのAPIが返すのはfMP4の断片です。iOS側はseqを付けてS3へPUTし、成功後にcommitします。
playlist本文の生成はAPI側へ任せますが、公開順を決める入力はiOSが送ります。
-->

---
layout: center
class: chapter
---

<div class="chapter-no">02</div>
<div class="chapter-rule"></div>

# Camera / Micから、<br>fMP4 Dataを取り出す
<p>AVCaptureSession → DataOutput → SampleBufferReceiver → AVAssetWriterDelegate</p>

<!--
[Timing checkpoint: 13:30]

ここから公開サンプルのHLSSegmentRecorderへ入ります。
標準SDKのobjectをどう接続すると、CameraとMicからfMP4 Dataを取り出せるのかを順番に見ます。
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

<div class="kicker">URL-LESS HLS WRITER</div>

# HLS用Writer

<div class="writer-setup-layout">

```swift
let writer = AVAssetWriter(
    contentType: .mpeg4Movie
)
writer.outputFileTypeProfile = .mpeg4AppleHLS
writer.preferredOutputSegmentInterval = .init(
    seconds: 2, preferredTimescale: 600
)
writer.initialSegmentStartTime = startTimeOffset
writer.delegate = self
```

<div class="writer-segment-output">
  <div class="writer-shape"><b>AVAssetWriter</b><span>HLS profile · URLなし</span></div>
  <i>→ delegate →</i>
  <div class="writer-data-stack"><span>initialization Data</span><span>separable Data</span></div>
</div>

</div>

<div class="bottom-claim">Writer自身はファイルへ保存せず、生成したfMP4をDataで返す</div>

<!--
通常の録画では出力先URLを指定しますが、segment delegateを使う構成ではcontentTypeだけでWriterを作ります。
Apple HLS profile、希望segment間隔、initial start time、delegateを設定します。

このdelegate methodを実装すると通常のファイル書き込みは抑止され、Writerがsegment Dataをcallbackします。
各propertyの意味と2秒境界はChapter 04で詳しく見ます。
preferredTimescaleに 600 を指定した場合、1秒は 600/600 となり、1/600秒単位の細かい時間を表現できるようになります。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriter
- https://developer.apple.com/documentation/avfoundation/avfiletypeprofile/mpeg4applehls
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">INPUT → RECEIVER → WRITER</div>

# Receiverが、sampleの書き込み口になる

<div class="receiver-setup-layout">

```swift
let videoInput = AVAssetWriterInput(
    mediaType: .video,
    outputSettings: videoSettings()
)
let audioInput = AVAssetWriterInput(
    mediaType: .audio,
    outputSettings: audioSettings()
)

self.videoReceiver = writer.inputReceiver(
    for: videoInput
)
self.audioReceiver = writer.inputReceiver(
    for: audioInput
)
```

<div class="receiver-connection">
  <div class="receiver-row video"><span>video CMSampleBuffer</span><b>videoReceiver</b><code>H.264 Input</code></div>
  <div class="receiver-row audio"><span>audio CMSampleBuffer</span><b>audioReceiver</b><code>AAC Input</code></div>
  <i>↓ attached to ↓</i>
  <strong>AVAssetWriter</strong>
</div>

</div>

<div class="bottom-claim">Inputがencode設定を持ち、ReceiverがCMSampleBufferの入口になる</div>

<!--
AVAssetWriterInputには、VideoならH.264、AudioならAACなどのencode設定を渡します。
inputReceiver(for:)は、そのInputをWriterへ接続すると同時に、CMSampleBufferを書き込むReceiverを返します。

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
  <div class="sample-flow-writer"><span>AVAssetWriter</span><b>encode<br>+ interleave<br>+ segment</b></div>
  <div class="sample-flow-arrow">→</div>
  <div class="sample-flow-stack outputs">
    <div class="sample-flow-node init"><span>1配信に1つ</span><b>initialization</b><small>codec · trackなどの再生設定</small></div>
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

AVAssetWriterが多数のCMSampleBufferをH.264とAACへencodeし、2 trackを同じ時間軸でinterleaveして、約2秒分のfragmentへまとめます。
30fpsなら1つのsegmentにおよそ60 frameが入ります。実際の境界はkeyframeにより前後します。

最初にcodecやtrack設定を持つinitialization Dataが届き、その後に映像・音声本体を持つseparable Dataが繰り返し届きます。

[Sources]
- https://developer.apple.com/videos/play/wwdc2020/10011/
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">CAPTURE CALLBACK → RECEIVER</div>

# 1回のcallbackで、1つのsampleをReceiverへ渡す

<div class="append-pipeline">
  <div><span>DataOutput delegate</span><b>captureOutput</b></div>
  <i>→</i>
  <div><span>Chapter 03</span><b>時刻を補正</b></div>
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
trueならappend成功、falseならWriterがまだ受け入れられないため、そのsampleを待たずに落とします。throwならWriter自体の失敗としてAsyncThrowingStreamをerrorで閉じます。
待つappendではなくappendImmediatelyを選ぶ理由は、Camera callbackへ古いframeを貯めないためです。

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
    <code>.separable</code><i>→</i><b>HLSFragment.media(seq, …)</b><i>→</i><strong>seg/000001.m4s</strong>
  </div>
</div>

```swift
switch segmentType {
case .initialization:
    continuation.yield(.initialization(segmentData))
case .separable:
    segmentIndex += 1
    continuation.yield(.media(
        sequence: segmentIndex,
        data: segmentData,
        duration: duration
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

<div class="kicker">SYNC CALLBACK → ASYNC TASK</div>

# 生成したDataだけを、公開Taskへ渡す

<div class="async-boundary-flow">
  <div class="async-boundary-stage callback">
    <span>SYNC CALLBACK</span>
    <b>fMP4 Dataを生成</b>
    <small>DataOutput → Receiver → Writer delegate</small>
    <code>writingQueue</code>
  </div>
  <i>→</i>
  <div class="async-boundary-stage bridge">
    <span>BRIDGE</span>
    <b>HLSFragment</b>
    <small>initialization / media</small>
    <code>AsyncThrowingStream</code>
  </div>
  <i>→</i>
  <div class="async-boundary-stage task">
    <span>ASYNC TASK</span>
    <b>保存してから公開</b>
    <small>HTTP PUT → playlist更新</small>
    <code>Publisher actor</code>
  </div>
</div>

<div class="async-boundary-types">HLSSegmentRecorder <i>→</i> HLSStreamPublisher</div>
<div class="bottom-claim">callbackはDataを渡して戻る。HTTPの完了はPublisher Taskが待つ</div>

<!--
ここまで見たHLSSegmentRecorderの流れを、非同期境界としてまとめます。
DataOutput callbackでReceiverへsampleを渡し、Writer delegateでfragment Dataを受け取ります。
RecorderはそのDataをHLSFragmentとしてAsyncThrowingStreamへyieldし、callbackからすぐ戻ります。

右側ではHLSStreamPublisherのTaskがstreamを消費し、HTTP PUTとplaylist更新をawaitします。
そのため、AVFoundation callbackはネットワークの応答を待ちません。公開順はPublisher actorが守ります。

Recorderはstreamのcontinuationを準備してからWriterを有効にします。具体的な開始コードはAppendixへ移しました。
次の章では、Receiverへ渡す直前に行っていた映像と音声の時刻補正を見ます。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
- Apple: https://developer.apple.com/documentation/avfoundation/avcapturevideodataoutput/setsamplebufferdelegate(_:queue:)
- Apple: https://developer.apple.com/documentation/avfoundation/avcaptureaudiodataoutput/setsamplebufferdelegate(_:queue:)
-->

---
layout: center
class: chapter
---

<div class="chapter-no">03</div>
<div class="chapter-rule"></div>

# 映像と音声を、<br>同じ時間軸でWriterへ渡す
<p>Camera / Mic → CMSampleBuffer → one shared timeline</p>

<!--
[Timing checkpoint: 18:30]

生成パイプラインがつながったので、Receiverへ渡す直前の時刻補正を詳しく見ます。
映像と音声を同じ時間軸へ載せることが、次の難所です。
-->

---

<div class="kicker">ONE CAPTURE · THREE OUTPUTS</div>

# 1つのCaptureSessionから、3つの用途へ分ける

<div class="capture-branches">
  <div class="capture-branch-source">
    <span>Camera + Mic</span>
    <b>CaptureSession</b>
  </div>
  <div class="capture-branch-lines"><i></i><i></i><i></i></div>
  <div class="capture-branch-targets">
    <div class="preview"><span>画面表示</span><b>PreviewLayer</b><small>配信中もそのまま表示</small></div>
    <div class="hls"><span>ライブ配信</span><b>H.264 / AAC → fMP4</b><small>約2秒ごとにupload</small></div>
    <div class="local"><span>ローカル保存(個人アプリのみ)</span><b>HEVC / AAC → MP4</b><small>1080 × 1920 · 停止時にfinish</small></div>
  </div>
</div>

<div class="local-recording-result">
  <span>recording.mp4</span><i>→</i><b>元動画としてupload</b><i>+</i><b>設定時は写真ライブラリへ保存</b>
</div>

<div class="bottom-claim">公開サンプルはPreview + HLS。MomentNowは保存用Writerも並行する</div>

<!--
PreviewはAVCaptureVideoPreviewLayerへ同じCaptureSessionを接続するため、配信中も画面表示を続けられます。
VideoDataOutputとAudioDataOutputから届く同じ元のCMSampleBufferを、MomentNowでは2つのAVAssetWriterへ分岐します。

HLS Writerは配信用のH.264/AACを約2秒のfMP4へ分けます。
Local WriterはHEVC/AAC、1080×1920、5Mbpsの1本のMP4を一時領域へ作ります。
停止時に両Writerをfinishし、ローカルMP4は元動画として別途uploadします。設定が有効なら写真ライブラリへも保存します。
公開サンプルには理解を絞るためLocal Writerを含めていません。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/CameraPreviewView.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveStreamView.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveSegmentRecorder.swift
- MomentNow-iOS/MomentNow-iOS/LiveStreamView/LiveStreamViewModel.swift
-->

---

<div class="kicker">AVFOUNDATION COMMON · INPUT FORMAT</div>

# CameraのNV12をWriterがH.264に圧縮

<div class="format-bridge">
  <div class="format-node raw"><span>Capture</span><b>420f / NV12</b><small>pixel buffer</small></div>
  <div class="format-arrow">append</div>
  <div class="format-node encoded"><span>Writer</span><b>H.264 High</b><small>encoded media</small></div>
</div>

```swift
videoOutput.videoSettings = [
    kCVPixelBufferPixelFormatTypeKey as String:
        kCVPixelFormatType_420YpCbCr8BiPlanarFullRange
]
```

<div class="bottom-claim compact">サンプルは1.5 Mbps · 個人アプリは上り回線に合わせて0.9-2.5 Mbps</div>

<div class="source">HLSSegmentRecorder.setupCaptureSessionLocked()</div>

<!--
DataOutputからはNV12のpixel bufferを受け取ります。
この段階は未圧縮で、H.264への圧縮はAVAssetWriterInputのoutputSettingsが担当します。
公開サンプルは説明しやすい1.5 Mbps固定です。個人アプリは上り回線を優先し、品質設定ごとに0.9、1.6、2.5 Mbpsから選びます。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- LiveSegmentRecorder.makeHLSVideoSettings()
-->

---

<div class="kicker">CALLBACK GATES</div>

# Writerへ届く前に、2つのguardで落とす

<div class="guard-funnel">
  <div class="guard-input">captureOutput(_:didOutput:from:)</div>
  <div class="guard-step"><code>isWriting == true</code><span>録画中だけ</span></div>
  <div class="guard-step"><code>CMSampleBufferDataIsReady</code><span>利用可能なbufferだけ</span></div>
  <div class="guard-output">startWriterIfNeeded → append</div>
</div>

```swift
guard isWriting else { return }
guard CMSampleBufferDataIsReady(sampleBuffer) else { return }
```

<!--
リアルタイム入力では、遅延したframeを無制限に保持するとメモリと遅延が増えます。
個人アプリもlate frameを捨て、現在へ追いつく判断です。

CaptureSessionが動いていても、Writerへ渡すのは録画中だけです。
CMSampleBufferのdataがreadyでない場合も早期returnし、Writerの状態遷移を単純に保ちます。
-->

---

<div class="kicker">SESSION START</div>

# 最初のvideo frameで、全trackを開始

<div class="start-sequence">
  <div class="sequence-item audio"><span>audio CMSampleBuffer</span><small>まだappendしない</small></div>
  <div class="sequence-line"></div>
  <div class="sequence-item video"><span>first video frame</span><small>writer.start()</small></div>
  <div class="sequence-line active"></div>
  <div class="sequence-item session"><span>startSession(at: 10s)</span><small>video + audio共通</small></div>
</div>

```swift
guard output === videoOutput else { return }
guard !didStartSession, let writer else { return }
```

<!--
Writerのsessionは最初のvideo frameで開始します。
先にaudioが届いても、共通の基準が決まるまではappendしません。
-->

---

<div class="kicker">PTS = CMSAMPLEBUFFER TIMESTAMP</div>

# CMSampleBufferの撮影時刻を、Writerの開始時刻へ移す

<div class="dual-axis">
  <div class="axis-row">
    <b>Capture PTS<small>CMSampleBufferに付く撮影時刻</small></b>
    <div class="axis-line"><span class="axis-value source-value">58342.31</span><i></i><i></i><i></i><em>…</em></div>
  </div>
  <div class="axis-transform">− firstVideoPTS + 10s</div>
  <div class="axis-row target">
    <b>Writer timeline<small>この配信内の時刻</small></b>
    <div class="axis-line"><span class="axis-value target-value">10.00</span><i></i><i></i><i></i><em>…</em></div>
  </div>
</div>

<div class="bottom-claim warning">10秒起点は今回のWriter設定。HLS仕様の固定値ではない</div>

<div class="source">CMSampleBuffer presentationTimeStamp / writer.initialSegmentStartTime</div>

<!--
Capture PTSはCaptureSessionの連続時間です。大きな値から始まることがあります。
一方、Writerは10秒起点へ明示的に揃えるため、両者を変換します。
-->

---
layout: center
class: formula-slide
---

<div class="kicker">ONE DELTA</div>

# 時刻補正は、全CMSampleBufferへの平行移動

<div class="formula-large">
  <span>delta</span>
  <b>=</b>
  <code>10s − firstVideoPTS</code>
</div>

<div class="formula-large secondary">
  <span>adjustedPTS</span>
  <b>=</b>
  <code>sourcePTS + delta</code>
</div>

<!--
補正はレート変更ではなく、全CMSampleBufferへの平行移動です。
最初のvideo PTSからdeltaを一度だけ決め、その後は映像と音声へ同じ値を足します。
-->

---

<div class="kicker">A/V SYNC</div>

# VideoとAudioを別々に補正しない

<div class="sync-diagram">
  <div class="sync-source">
    <div><b>video</b><span>58342.31</span></div>
    <div><b>audio</b><span>58342.29</span></div>
  </div>
  <div class="sync-delta">same delta</div>
  <div class="sync-target">
    <div><b>video</b><span>10.00</span></div>
    <div><b>audio</b><span>9.98</span></div>
  </div>
</div>

<div class="bottom-claim">相対差を保ったまま、Writerのtime rangeへ移す</div>

<!--
音声用の開始時刻を別に決めると、A/Vの相対差が変わります。
videoから決めたdeltaを両方へ適用し、同期を保ちます。
-->

---
layout: center
class: chapter
---

<div class="chapter-no">04</div>
<div class="chapter-rule"></div>

# 映像と音声を、<br>約2秒のfMP4へ分ける
<p>Four HLS settings and segment boundaries</p>

<!--
[Timing checkpoint: 23:00]

Captureの時計が揃ったので、次は約2秒のfragment境界を作るHLS固有設定を見ます。
-->

---

<div class="kicker">WHAT UNBLOCKED THE IMPLEMENTATION</div>

# macOS向けAppleサンプルが、iOS実装の突破口になった

<div class="implementation-story">
  <div class="attempt"><span>2025.03</span><b>AIで最初の試作</b><small>再生できるHLSとして<br>実用まで至らず</small></div>
  <i>→</i>
  <div class="reference"><span>Apple fmp4Writer</span><b>正しい生成手順を確認</b><small>Writer設定 · 時刻補正<br>segment delegate</small></div>
  <i>→</i>
  <div class="adapt"><span>iPhone</span><b>Camera / Micへ置換</b><small>AVAssetWriterの核心部分を<br>iOSで利用</small></div>
</div>

<div class="bottom-claim">macOSのmovie入力を、iOSのライブCapture入力へ置き換えた</div>

<div class="source">Apple: Writing fragmented MPEG-4 files for HTTP Live Streaming</div>

<!--
端末内でfMP4を生成するアイデアを思いつき、2025年3月ごろ最初はAIへ実装させてみました。
しかし、再生できるHLSとして実用まで到達できませんでした。

突破口になったのがApple公式のfmp4Writerです。
公式プロジェクトはmacOS 11以降向けのCommand Line Toolで、movie fileをAVAssetReaderで読み込みます。
一方、AVAssetWriterのHLS profile、URLなしWriter、10秒の時刻offset、segment delegateという核心部分はiOSでも利用できます。
入力だけをAVCaptureVideoDataOutputとAVCaptureAudioDataOutputへ置き換え、ライブCaptureからの生成へつなげました。

[Sources]
- https://developer.apple.com/documentation/avfoundation/writing-fragmented-mpeg-4-files-for-http-live-streaming
- https://developer.apple.com/videos/play/wwdc2020/10011/
-->

---

<div class="kicker">HLS-SPECIFIC · FOUR KNOBS</div>

# HLS出力は、4つのpropertyで有効になる

<div class="setting-list">
  <div><b>01</b><code>outputFileTypeProfile</code><span>Apple HLS向けfMP4</span></div>
  <div><b>02</b><code>preferredOutputSegmentInterval</code><span>希望するsegment間隔</span></div>
  <div><b>03</b><code>initialSegmentStartTime</code><span>最初のsegment開始時刻</span></div>
  <div><b>04</b><code>delegate</code><span>生成されたDataの受け口</span></div>
</div>

<div class="source">HLSSegmentRecorder.setupWriterLocked()</div>

<!--
この4つがHLS segmentationの核です。
それぞれを1枚ずつ見ます。特にintervalは「必ず」ではなく「preferred」です。
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
次のページでは、希望位置の近くにIDRを用意するためのencoder設定を見ます。

[Sources]
- Apple: AVAssetWriter.preferredOutputSegmentInterval
- Apple HLS Authoring Specification for Apple Devices
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLS/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLS/HLSManifest.swift
-->

---

<div class="kicker">KEYFRAME ALIGNMENT</div>

# 2秒境界には、2秒以内のIDRが必要

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
  <div class="lane"><b>S3</b><span class="init-mark">PUT init</span><span class="callback-mark">PUT 1</span><span class="callback-mark">PUT 2</span><span class="callback-mark">PUT 3</span></div>
</div>

<div class="bottom-claim">Writerのfinishを待たず、生成とS3 uploadを並行できる</div>

<!--
通常の録画ファイルと違い、finishWritingまで待ちません。
Captureが続く間にfragmentがdelegateへ届くため、生成と保存・uploadを並行できます。
-->

---

<div class="kicker">BACKPRESSURE DECISION</div>

# Writerが詰まったら、古いframeを捨てる

<div class="drop-timeline">
  <div class="drop-lane"><b>Camera</b><span>frame 1</span><span>frame 2</span><span>frame 3</span><span>frame 4</span></div>
  <div class="drop-lane writer"><b>Writer</b><span class="append">append</span><span class="append">append</span><span class="busy">busy</span><span class="dropped">drop</span></div>
</div>

<div class="drop-choice">
  <div class="wait-choice">
    <span>待って貯める</span>
    <b>古い映像が残り、遅延が増える</b>
  </div>
  <div class="drop-choice-current">
    <span>待たずに落とす</span>
    <b>少しカクつくが、現在へ追いつく</b>
  </div>
</div>

<div class="bottom-claim warning"><code>appendImmediately == false</code> は意図的なdrop</div>

<!--
DataOutputは一定間隔でCMSampleBufferをpushし続けます。WriterがbusyでappendImmediatelyがfalseを返したら、bufferを保留せずreturnします。
この場合は意図的なdropです。Writerが受け取れるようになった後の新しいframe / audio blockから、すぐappendを再開します。

CMSampleBufferをqueueへ貯めれば完全性は上がりますが、古い映像を後から送るためライブ遅延とメモリ使用量が増えます。
このサンプルは少しのframe dropを許容し、視聴者へ現在に近い映像を届ける方を優先します。

appendImmediatelyがthrowした場合は、意図的なdropではありません。Writerの失敗としてstreamをerrorで閉じます。

[Sources]
- https://developer.apple.com/documentation/avfoundation/avassetwriterinput/samplebufferreceiver/appendimmediately(_:)
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">VISIBLE OUTPUT</div>

# 再生開始は、init PUTと最初のcommitの後

<div class="output-clock">
  <div class="clock-column">
    <span>start</span><b>init Data</b>
  </div>
  <div class="clock-column">
    <span>network</span><b>PUT init</b>
  </div>
  <div class="clock-column active">
    <span>~2s</span><b>PUT seg 1</b>
  </div>
  <div class="clock-column active">
    <span>after PUT</span><b>commit seq 1</b>
  </div>
</div>

<div class="playable-equation">
  <span>init.mp4<br><small>保存済み</small></span><i>+</i>
  <span>000001.m4s<br><small>保存済み</small></span><i>+</i>
  <span>playlist<br><small>seq 1を公開済み</small></span><i>=</i>
  <b>Viewer<br>再生開始</b>
</div>

<!--
AVAssetWriterがDataを返しただけでは、まだ視聴者は再生できません。
initと最初のm4sがS3に存在し、seq 1のcommitでplaylistへ載った時点が最初の再生可能状態です。
-->

---

<div class="kicker">PUBLIC SAMPLE · LOCAL PLAYLIST</div>

# playlist公開後に、状態を確定する

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
  <i>PUT 2xx</i>
  <div class="hot"><span>local stateを確定</span><b>current = candidate</b></div>
</div>

<div class="bottom-claim">後半のcommit APIとは別。ここではiPhone自身がplaylist本文を書く</div>

<!--
サンプルのHLSManifestは文字列への追記ログではなく、seqをkeyにした値です。
同じseqを二度受けても重複せず、render時には必ず番号順になります。

内部状態を先に進めません。
次のmanifestをコピーで作り、その本文のPUTが成功した後だけcurrentへ代入します。失敗したsegmentをUI上で公開済みにしないためです。

HTTPHLSClientはobjectごとのpathとContent-Typeを組み立てます。
m4sのseqは6桁にzero paddingし、playlistだけをUTF-8 textからDataへ変換します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSManifest.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HTTPHLSClient.swift
-->

---

<div class="kicker">DRAIN BEFORE FINISH</div>

# stopは、3つの非同期処理を順番に閉じる

<div class="drain-lanes">
  <div><b>1 Recorder</b><span>capture停止 → finishWriting</span><small>最後のfragmentが出る</small></div>
  <i>→</i>
  <div><b>2 Stream</b><span>continuation.finish()</span><small>fragment列を閉じる</small></div>
  <i>→</i>
  <div class="hot"><b>3 Publisher</b><span>uploadTask.value</span><small>ENDLIST PUTまで待つ</small></div>
</div>

<div class="bottom-claim">「撮影停止」と「配信終了」は、同じ瞬間ではない</div>

<!--
SampleHLSStreamer.stopRecordingの順序です。
Recorder.stopがcapture、Writer、streamを順に閉じ、PublisherがENDLIST付きplaylistをPUTするまでTaskを待ちます。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleHLSStreamer.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---
layout: center
class: chapter
---

<div class="chapter-no">05</div>
<div class="chapter-rule"></div>

# 保存できたsegmentだけを、<br>playlistへ公開する
<p>Save bytes first, then make the segment visible to viewers</p>

<!--
[Timing checkpoint: 29:30]

ここから保存と公開の順序を見ます。
まずコード名を使わず、segment本体が保存されてからplaylistへ載るまでを捉えます。
その後、MomentNowのpresign、S3 PUT、commitへ対応づけます。
-->

---

<div class="kicker">THREE REQUESTS · THIS PRODUCTION DESIGN</div>

# 「保存先を受け取る → 保存する → 公開する」に分ける

<div class="settings-table">
  <div class="settings-head"><span>APPの操作</span><span>MomentNow</span><span>結果</span></div>
  <div><b>一時保存先を受け取る</b><code>POST /presign</code><span>短期PUT URL</span></div>
  <div><b>segment本体を保存する</b><code>PUT → S3</code><span>objectが存在する</span></div>
  <div><b>playlistへ載せる</b><code>POST /commit</code><span>Viewerから見える</span></div>
  <div><b>playlistを取得する</b><code>GET → CloudFront</code><span>再生が進む</span></div>
</div>

<div class="bottom-claim">S3は保存先の選択。HLSで重要なのは「保存してから公開」</div>

<!--
観客がMomentNowのコードを知らなくても追えるよう、まず4つの操作として説明します。
一時アップロード先を受け取り、segment本体を保存し、成功したあとにだけplaylistへ載せ、Viewerが取得します。

MomentNowではpresign API、S3 PUT、commit API、CloudFrontが担当します。
公開サンプルではMacのHTTPサーバーが保存とplaylist更新を担当します。S3自体はHLSの必須要素ではありません。

[Sources]
- MomentNow-Lambda/src/presign.ts
- MomentNow-Lambda/src/commit.ts
- iosdc2026HLSSample/server/server.py
-->

---

<div class="kicker">PREPARE BEFORE MEDIA</div>

# segment前にstreamIdと再生URLを準備

<div class="create-stream-flow">
  <div><b>createStream</b><span>streamId / prefix</span></div>
  <i>→</i>
  <div><b>ticket</b><span>playbackUrl</span></div>
  <i>→</i>
  <div><b>permission</b><span>camera / microphone</span></div>
  <i>→</i>
  <div><b>ready</b><span>previewSession</span></div>
</div>

```swift
if ticket == nil {
    ticket = try await apiClient.ticket(streamId: stream.streamId)
}
await requestPermissionsAndSetup()
try await streamer?.start()
```

<div class="source">LiveStreamViewModel.prepare()</div>

<!--
配信画面へ入る前にstreamは作成済みです。
ViewModelはticket、権限、previewを揃えてreadyへ進み、録画開始後すぐuploadできる状態を作ります。

Recorderは同期callbackを返し、StreamerがTaskを作ってUploaderへ渡します。
AVFoundationのwritingQueueをURLSessionの完了待ちで塞がない責務分割です。
-->

---

<div class="kicker">INITIALIZATION SEGMENT</div>

# init.mp4は、最初に1度だけPUT

```swift
guard !didUploadInit else { return }
let ticket = try await api.presign(
    streamId: stream.streamId, kind: "init", seq: nil
)
try await api.put(
    to: URL(string: ticket.putUrl)!, data: initData,
    contentType: "video/mp4"
)
didUploadInit = true
```

<div class="source">LiveHLSStreamer / HLSUploadCoordinator.uploadInitIfNeeded</div>

<!--
initialization callbackは1配信で最初に届きます。
Coordinator側でもdidUploadInitを持ち、重複PUTを防いでいます。
-->

---

<div class="kicker">MEDIA SEGMENT</div>

# mediaごとにpresign → PUT → commit

```swift
let ticket = try await api.presign(
    streamId: stream.streamId, kind: HLSUploadKind.segment,
    seq: seq
)
try await api.put(
    to: URL(string: ticket.putUrl)!, data: segmentData,
    contentType: HLSContentType.segmentM4S
)
_ = try await api.commit(
    streamId: stream.streamId, seq: seq, durationSec: dur,
    isLast: false, targetDurationSec: targetDurationSec
)
```

<!--
presign APIから、seqに対応する一時PUT URLを取得します。
S3 PUTが成功した後だけcommitし、サーバーへplaylistへ載せてよいseqを通知します。
-->

---

<div class="kicker">NO AWS CREDENTIAL ON DEVICE</div>

# iPhoneへ渡すのは、短期のPUT URLだけ

<div class="choice-compare">
  <div class="choice muted">
    <span>DO NOT</span>
    <b>AWS credentialを内包</b>
    <small>漏えい範囲が広い</small>
  </div>
  <div class="choice-arrow">→</div>
  <div class="choice selected">
    <span>PRESIGNED URL</span>
    <b>object単位のPUT権限</b>
    <small>streamId / kind / seqで制約</small>
  </div>
</div>

<div class="bottom-claim">サーバーが「この1ファイルだけPUTしてよいURL」を一時発行する</div>

```swift
var request = URLRequest(url: presignedURL)
request.httpMethod = "PUT"
request.setValue(contentType, forHTTPHeaderField: "Content-Type")
request.httpBody = data
```

<!--
iOSアプリはS3のcredentialを持ちません。
認証済みAPIからobject単位の一時URLを取得し、URLSessionでDataを直接PUTします。
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

<div class="bottom-claim">サンプルは最大3回retry · 失敗したsegmentはplaylistへ載せない</div>

<!--
順序が重要です。
PUTの2xxを確認してからcommitします。逆なら、Playerがplaylistで見つけたURIをGETして404になります。
公開サンプルは同じobjectを最大3回まで再送し、それでも失敗した場合はplaylist更新へ進まずerrorとして残します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
-->

---

<div class="kicker">OUT-OF-ORDER RACE</div>

# 現在のcommit APIはseq順へ並べ直すが、<br>欠番は待たない

<div class="race-lanes">
  <div class="race-row"><b>seg 1</b><span class="race-bar slow">upload 1</span><em>commit 1</em></div>
  <div class="race-row"><b>seg 2</b><span class="race-bar fast">upload 2</span><em class="early">commit 2</em></div>
  <div class="race-row"><b>seg 3</b><span class="race-bar medium">upload 3</span><em>commit 3</em></div>
</div>

<div class="order-equation">
  <span>completion</span><code>2, 1, 3</code>
  <b>≠</b>
  <span>playback</span><code>1, 2, 3</code>
</div>

<div class="bottom-claim warning">理想は、連続したseqまでだけを公開する「公開済み境界」を持つこと</div>

<!--
actorでもawait中は別segmentが進むため、seg 2がseg 1より先にPUT完了する可能性があります。
iOSはseqを必ず送ります。現在のcommit APIはplaylist内のsegmentをseq順へ並べ直しますが、連続番号だけに制限する契約ではありません。seg 2が先なら、一時的にseg 2だけが公開されます。
理想は、サーバーが連続したseqまでの「公開済み境界」を管理することです。
-->

---

<div class="kicker">ACTOR STATE</div>

# Actorは、uploadと終了の状態をまとめて守る

<div class="concurrency-map">
  <div class="concurrency-row">
    <b>didUploadInit</b><span>init.mp4の重複PUTを防ぐ</span><i class="green"></i>
  </div>
  <div class="concurrency-row">
    <b>lastCommittedSeq</b><span>commit成功済みの最大seq</span><i class="cobalt"></i>
  </div>
  <div class="concurrency-row">
    <b>inUploadSegmentCount</b><span>停止時に待つbacklog</span><i class="coral"></i>
  </div>
  <div class="concurrency-row">
    <b>isFinishing</b><span>遅れて完了したsegmentでもENDLISTを再通知</span><i class="coral"></i>
  </div>
</div>

<div class="bottom-claim warning">actor内のawaitは再入可能。完了順ではなくseqを正とする</div>

<!--
Coordinatorはactorですが、presignやPUTのawait中に別segmentの処理が進みます。
そのため到着順や完了順ではなく、seqとpending countを明示的な状態として持ちます。
lastCommittedSeqはcommit成功済みの最大seqであり、それ以前のseqがすべて成功したことは保証しません。
-->

---

<div class="kicker">STOP CONTRACT</div>

# 現状は、actorへ入った<br>uploadが0になるまで待つ

<div class="publish-order four-step">
  <div class="publish-step"><b>1</b><span>recorder.stop()</span></div>
  <div class="publish-arrow">→</div>
  <div class="publish-step"><b>2</b><span>uploader.finish()</span></div>
  <div class="publish-arrow">→</div>
  <div class="publish-step"><b>3</b><span>actor内 pending == 0</span></div>
  <div class="publish-arrow">→</div>
  <div class="publish-step"><b>4</b><span>state = completed</span></div>
</div>

```swift
while await streamer.hasPendingUploads() {
    try? await Task.sleep(for: .milliseconds(500))
}
```

<div class="source">LiveStreamViewModel.stop()</div>

<div class="bottom-claim warning">理想はupload Task／イベント列そのものとfinal commitをawaitすること</div>

<!--
録画停止は、ネットワーク送信完了と同義ではありません。
現在の個人アプリ実装はUploaderへisLastを通知し、actor内のpending uploadがゼロになるまでcompletedへ進めません。
ただし、onMediaSegmentで作ったTaskがactorに入る前はpending countへ反映されません。そのため、Taskが残っていてもpendingが0に見える余地があります。
公開サンプルはAsyncThrowingStreamのconsumer Taskを保持し、Recorderがstreamを閉じた後にTaskの終了までawaitします。完了契約としてはこちらのほうが明確です。
-->

---
layout: center
class: chapter
---

<div class="chapter-no">06</div>
<div class="chapter-rule"></div>

# Writerを待たせず、<br>最後の公開完了まで管理する
<p>Callback → Task → upload completion</p>

<!--
[Timing checkpoint: 34:00]

最後に、iOS上のcallback、Task、actor、URLSessionを1本のライブ配信として整理します。
焦点は、AVFoundationを止めずにネットワークの遅さを吸収する境界です。
-->

---

<div class="kicker">CALLBACK TO TASK</div>

# callbackでは待たず、TaskへDataを引き渡す

<div class="replacement-map">
  <div class="replacement-source">
    <b>LiveSegmentRecorder</b>
    <span>init Data / media Data</span>
  </div>
  <div class="replacement-branches">
    <div class="replacement-local">
      <span>WRITING QUEUE</span>
      <b>callbackを返す</b>
      <small>次のCMSampleBuffer処理へ</small>
    </div>
    <div class="replacement-prod">
      <span>SWIFT TASK</span>
      <b>Uploaderをawait</b>
      <small>presign / PUT / commit</small>
    </div>
  </div>
</div>

<div class="mapping-table">
  <span>initialization → uploadInitIfNeeded</span>
  <span>separable → uploadSegment</span>
  <span>segmentReport → duration改善余地</span>
</div>

<!--
AVAssetWriterDelegateはwritingQueue上で呼ばれます。
callback内ではTaskを作るだけにし、ネットワークawaitはSwift Concurrency側へ逃がします。
-->

---

<div class="kicker">PUBLIC DEMO → PRODUCTION</div>

# 個人アプリでは、HLSの外側に4つの制御を足す

<div class="production-additions">
  <div><b>AUTH</b><span>user / group ownership</span></div>
  <div><b>PRESIGN</b><span>object単位の短期PUT権限</span></div>
  <div><b>COMMIT</b><span>公開順とplaylistの同時更新を制御</span></div>
  <div><b>DELIVERY</b><span>CloudFront / ticket / status</span></div>
</div>

<div class="bottom-claim">AVAssetWriterの後ろを差し替えれば、同じiOS pipelineを使える</div>

<!--
iosdc2026HLSSampleはMacを小さなobject serverとして使います。
個人アプリは先にpresignしてS3へPUTし、Lambdaのcommitでplaylistを更新します。生成するDataと保存順序は共通です。

サンプルから個人アプリへ足すものです。
認証、署名URL、playlist競合制御、CDN配信。どれも重要ですが、映像の再エンコードではありません。

[Sources]
- iosdc2026HLSSample/README.md
- MomentNow-Lambda/src/presign.ts
- MomentNow-Lambda/src/commit.ts
- MomentNow-Lambda/src/create_stream.ts
- MomentNow-Lambda/src/ticket.ts
-->

---
layout: center
class: statement ink-statement
---

<div class="kicker">BACK TO THE DEMO</div>

# 冒頭のデモで増えていたのは<br><span class="accent-primary">「ファイル」と「再生可能な順序」</span>

<div class="demo-recap">
  <span>init.mp4</span><i>＋</i><span>000001.m4s</span><i>＋</i><span>playlist更新</span><i>＝</i><b>LIVE</b>
</div>

<!--
冒頭のデモへ戻ります。
ライブ配信に見えていたものは、iPhoneが作るファイルと、存在確認後にplaylistへ載せる順序の積み重ねでした。
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
    <b>大規模・長時間・ABR・厳しいSLA</b>
    <small>発熱、電池、回線変動を吸収したい</small>
  </div>
</div>

<div class="bottom-claim">「端末でできる」と「端末でやるべき」は別の判断</div>

<!--
この構成は万能ではありません。
少人数・短時間・単一品質には合います。大規模、長時間、ABR、厳しいSLAなら専用のmedia platformを選びます。
-->

---
layout: center
class: closing
---

<div class="kicker">TAKEAWAYS</div>

# HLSライブ配信は、4つの順序で成立する

<div class="takeaway-grid">
  <div><b>01</b><span><strong>時刻 · Clock</strong>Capture PTSをWriterのtimelineへ移す</span></div>
  <div><b>02</b><span><strong>区切り · Boundary</strong>IDRとsegment intervalをそろえる</span></div>
  <div><b>03</b><span><strong>公開 · Upload</strong>DataをS3へPUTしてからcommitする</span></div>
  <div><b>04</b><span><strong>終了 · Finish</strong>全送信とfinal commitの終了を待つ</span></div>
</div>

<div class="closing-footer">
  <div>
    <b>ありがとうございました</b>
  </div>
</div>

<!--
[Timing checkpoint: 37:30]

まとめです。
AVAssetWriterDelegateでiPhoneからfMP4を逐次取り出し、presigned PUTでS3へ送り、成功したseqをcommitします。
Clock、Boundary、Upload、Finishの順序が揃って初めて、録画ではなくライブ配信になります。

公開サンプルは、そのうちAVFoundationの生成処理を読みやすくした教材です。ありがとうございました。

本編はここで終了します。
-->

---
layout: center
class: chapter
---

<div class="chapter-no">APPENDIX</div>
<div class="chapter-rule"></div>

# 時間が余ったとき／Q&A用
<p>設定値・時刻補正・サーバー運用の詳細</p>

<!--
ここから先は本編では使用しません。
質問に応じて、設定値、時刻補正、サーバー運用の詳細を参照します。
-->


---

<div class="kicker">SOURCE TREE</div>

# 9ファイルで、生成から再生まで追える

<div class="source-tree">
  <div class="source-row core"><b>HLSSegmentRecorder.swift</b><span>capture / encode / segment</span></div>
  <div class="source-row core"><b>SampleHLSStreamer.swift</b><span>recorderとuploadを接続</span></div>
  <div class="source-row core"><b>HLSStreamPublisher.swift</b><span>順序 / retry / finish</span></div>
  <div class="source-row core"><b>HLSManifest.swift</b><span>playlist state</span></div>
  <div class="source-row core"><b>HTTPHLSClient.swift</b><span>PUT / health check</span></div>
  <div class="source-row"><b>SampleStreamViewModel.swift</b><span>permission / UI state</span></div>
  <div class="source-row"><b>ContentView.swift</b><span>操作と観測</span></div>
  <div class="source-row"><b>server.py</b><span>atomic replace / GET</span></div>
  <div class="source-row"><b>static/app.js</b><span>latest stream / hls.js</span></div>
</div>

<!--
中心は上の5ファイルです。
下の4つは操作、保存、再生確認を担当します。ここから、画面からdelegate callbackまで呼び出し順に追います。

[Sources]
- iosdc2026HLSSample repository tree
-->

---

<div class="kicker">START ORDER · APPENDIX</div>

# streamを準備してから、Publisherへ渡す

```swift {1|3-5|7-10}
let fragments = try await recorder.startRecording()

let uploadTask = Task {
    await publisher.publish(fragments)
}

activeStream = ActiveStream(
    publisher: publisher,
    uploadTask: uploadTask
)
```

<!--
RecorderはAsyncThrowingStreamのcontinuationを保持してからWriterを有効にし、準備済みのstreamを返します。
そのため、PublisherのTaskが動き出す前にinit Dataが届いてもstream内にbufferされます。

Writer delegateは同期callbackです。
そこでDataをAsyncThrowingStreamへ渡してすぐ戻り、HTTP処理はPublisher側でawaitします。メディア処理とネットワーク待ちの境界です。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleHLSStreamer.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">CONFIG</div>

# 上り回線に合わせて、配信品質を選ぶ

```swift
let profile: (CGSize, Int) = switch quality {
case .high:   (.init(width: 720, height: 1280), 2_500_000)
case .medium: (.init(width: 720, height: 1280), 1_500_000)
case .low:    (.init(width: 480, height: 854),    900_000)
}
```

<div class="config-rail">
  <div><b>NWPath</b><span>constrained / expensive</span></div>
  <div><b>presign</b><span>API latency</span></div>
  <div><b>probe PUT</b><span>upload throughput</span></div>
  <div><b>2.0 sec</b><span>segment target</span></div>
</div>

<div class="source">LiveSegmentRecorder.Config.resolved / AutoStreamingQualityResolver</div>

<!--
個人アプリではNWPathと小さなprobe PUTから、high、medium、lowの1品質を配信開始前に選びます。
途中でrenditionを切り替えるABRではなく、端末の上り回線に合わせた開始時の選択です。
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

<div class="bottom-claim">AVCaptureSessionだけでなく、AVAudioSessionのrouteも配信品質に影響する</div>

<!--
録音権限だけではなく、AVAudioSessionのcategoryとmodeを先に設定します。
Bluetooth HFPを含む入力routeを許可しつつ、端末側の再生はspeakerを既定にしています。
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
出力サイズはportraitで指定し、video connectionへ90度のrotation angleを設定します。
配信中の向き変更は扱わず、portrait 90度で固定します。
-->

---

<div class="kicker">COPY TIMING</div>

# 映像・音声は変えず、時刻情報だけを補正

```swift {3-8}
let timingInfos = try sampleTimingInfos().map { info in
    var adjusted = info
    adjusted.presentationTimeStamp =
        info.presentationTimeStamp + offset
    if info.decodeTimeStamp.isValid {
        adjusted.decodeTimeStamp = info.decodeTimeStamp + offset
    }
    return adjusted
}
let copied = try CMSampleBuffer(copying: self, withNewTiming: timingInfos)
```

<div class="code-caption">PTSと、有効なDTSを同じ量だけ動かす</div>

<!--
CMSampleBufferの映像・音声データはそのままに、timing infoを差し替えたコピーを作ります。
PTSだけでなく、有効なDTSも同じ量だけ補正します。
-->

---

<div class="kicker">STOP IN THE SAME CLOCK</div>

# 開始と終了を、同じ補正後timelineで閉じる

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
終了時刻も補正後のPTSを使います。
finishWritingによって最後のsegmentがdelegateへ届く可能性があるため、stopはその完了まで待ちます。
-->

---

<div class="kicker">INITIAL START TIME</div>

# Writerの最初のsegmentは、10秒起点に固定

<div class="start-time-visual">
  <div class="start-empty"><span>0</span><i></i><i></i><i></i><i></i></div>
  <div class="start-marker"><b>10.00s</b><span>first adjusted video frame</span></div>
  <div class="start-media"><i></i><i></i><i></i><span>media timeline</span></div>
</div>

```swift
private let startTimeOffset = CMTime(value: 10, timescale: 1)
writer.initialSegmentStartTime = startTimeOffset
```

<!--
個人アプリのHLS Writerも初期segmentの開始を10秒へ設定します。
Capture PTSの補正は、この指定と入力CMSampleBufferを一致させるために必要でした。
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
音声はAAC、mono、44.1kHzで、low 48、medium 64、high 96kbpsです。
家族向けの短い映像という用途に合わせ、stereoより送信量を優先しています。
-->

---

<div class="kicker">PUBLIC SAMPLE · SEGMENT DURATION</div>

# 実測durationを使い、取れなければ2秒

<div class="duration-compare">
  <div class="duration-side sample">
    <span>REPORT</span>
    <b>video track duration</b>
    <small>EXTINFへ実際の長さ。とくに最後のsegmentは2秒にならないので必要</small>
  </div>
  <div class="duration-side production">
    <span>FALLBACK</span>
    <b>config.segmentSeconds</b>
    <small>無効・未取得なら2.0</small>
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
公開サンプルはAVAssetSegmentReportのvideo track durationをplaylistへ渡します。
reportがない、無効、0以下の場合だけ設定値の2秒へ戻します。frame reorderingは無効なので、このサンプルではvideo reportを採用しています。
現在の個人アプリ実装はまだfragmentSecondsの2.0固定なので、同じ取り込み方を適用できる改善点です。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
-->

---

<div class="kicker">PUBLIC SAMPLE · RETRY POLICY</div>

# 公開サンプルは、同じobjectを<br>最大3回まで再送する

<div class="retry-steps">
  <div><b>attempt 1</b><span>失敗</span><small>250 ms</small></div>
  <i>→</i>
  <div><b>attempt 2</b><span>失敗</span><small>500 ms</small></div>
  <i>→</i>
  <div><b>attempt 3</b><span>成功 / error</span><small>終了</small></div>
</div>

<div class="bottom-claim warning">segment PUTが失敗したら、そのsegmentをplaylistへ載せない</div>

<!--
サンプルは各PUTを最大3回試します。
segment保存が3回とも失敗した場合、playlist PUTへ進まず、Publisherのerrorとして残します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSStreamPublisher.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSampleTests/HLSStreamPublisherTests.swift
-->

---

<div class="kicker">SERVER · ATOMIC REPLACE</div>

# PUT中のファイルを、Viewerへ見せない

<div class="atomic-replace">
  <div><b>1 temporary file</b><span>request bodyを書き込む</span></div>
  <i>→</i>
  <div><b>2 flush + fsync</b><span>長さまで書き切る</span></div>
  <i>→</i>
  <div class="hot"><b>3 os.replace</b><span>完成品へ一気に置換</span></div>
</div>

<div class="bottom-claim">playlistの途中状態や、半分だけのm4sをGETさせない</div>

<!--
Macサーバーはrequest bodyをdestinationへ直接書きません。
同じdirectoryの一時ファイルへ書き切り、fsyncした後にos.replaceします。Viewerには古い完成品か新しい完成品だけが見えます。

[Sources]
- iosdc2026HLSSample/server/server.py
-->

---

<div class="kicker">SERVER · OBJECT SEMANTICS</div>

# cache方針は、playlistとsegmentで分ける

<div class="cache-table server-cache-table">
  <div class="cache-head"><span>OBJECT</span><span>CACHE</span><span>HTTP</span></div>
  <div class="dynamic"><b>playlist.m3u8</b><span>no-store / no-cache</span><code>毎回最新を取得</code></div>
  <div class="immutable"><b>init.mp4 / m4s</b><span>max-age=31536000</span><code>Range / 206対応</code></div>
</div>

<div class="bottom-claim compact">S3 / CloudFrontでの設定</div>

<!--
playlistは同じURLの内容が増えるためキャッシュさせません。
initとm4sは一度置いたら変えず長期cacheし、PlayerのRange requestには206で返します。個人アプリもobjectの可変性で方針を分けます。

[Sources]
- iosdc2026HLSSample/server/server.py
- MomentNow-Lambda/src/create_stream.ts
- MomentNow-Lambda/src/presign.ts
-->
