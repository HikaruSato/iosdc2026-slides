---
theme: default
title: 映像変換サーバーなしでiPhone端末内でHLSを生成してライブ配信
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
      <span class="cover-line"><span class="accent-coral">iPhone端末内で</span></span>
      <span class="cover-line"><span class="accent-coral">HLS</span>を生成してライブ配信</span>
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
layout: center
class: chapter
---

<div class="chapter-no">01</div>
<div class="chapter-rule"></div>

# まず、HLSで<br>「ライブ」になる仕組み
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

# HLSは、短い動画と<br><span class="accent-coral">更新されるplaylist</span>をHTTPで配る

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
    <b>ENDLIST</b>
    <small>もう増えない</small>
  </div>
</div>

<div class="bottom-claim">Playerは同じplaylist.m3u8を繰り返しGETする</div>

<div class="source">Apple: Event playlist construction / RFC 8216 §4.3.3</div>

<!--
配信側は短いsegmentを作り、そのURIをplaylistの末尾へ追加します。
Playerはplaylistを再取得し、新しく見つけたsegmentを順に取得します。この繰り返しがHLSのライブ配信です。停止時はENDLISTで閉じます。

[Sources]
- RFC 8216 §4.3.3: Media Playlist Tags
-->

---

<div class="kicker">FROM HLS TO THIS TALK</div>

# 今回は、segmentを作る場所をiPhoneへ移す

<div class="architecture-compare">
  <div class="arch-row muted">
    <div class="arch-label">一般的</div>
    <div class="arch-node">配信端末</div><div class="arch-arrow">→</div>
    <div class="arch-node server">映像サーバー<small>encode / segment</small></div><div class="arch-arrow">→</div>
    <div class="arch-node">CDN</div><div class="arch-arrow">→</div>
    <div class="arch-node">Viewer</div>
  </div>
  <div class="arch-row">
    <div class="arch-label">今回</div>
    <div class="arch-node phone">iPhone<small>capture / encode / segment</small></div><div class="arch-arrow">→</div>
    <div class="arch-node">S3</div><div class="arch-arrow">→</div>
    <div class="arch-node">CloudFront</div><div class="arch-arrow">→</div>
    <div class="arch-node">Viewer</div>
  </div>
</div>

<!--
HLSでライブになる仕組みはそのまま使い、一般的な構成からセグメント生成の場所だけを変えます。
iPhoneが生成済みのHLSファイルをオブジェクトストレージへ置くので、バックエンドは映像バイト列を変換しません。

バックエンドがゼロという意味ではありません。
API、保存、配信、状態管理は残ります。なくすのは、映像を受け取り続けて変換する役割です。
-->

---

<div class="kicker">WHY</div>
<div class="story-split">
  <div>
    <h1>撮影後に待つほど、<br>共有したい瞬間から遠ざかる</h1>
    <p class="lead">子どもの動画を家族へ送る。<br>撮影中から届けば、撮影後に待たせずに済む。</p>
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

# iPhoneが生成したHLSを<br><span class="accent-coral">撮影中からS3へ公開する</span>

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

# 仕組みが見えたところで、最小構成を動かす

<div class="live-demo-grid">
  <div class="live-demo-phone">
    <span>iosdc2026HLSSample</span>
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

# Macは「PUTされたファイルを残すだけ」

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
  <div><b>Camera + Mic</b><span>映像・音声sample</span></div>
  <i>→</i>
  <div class="hot"><b>AVAssetWriter</b><span>init.mp4 + m4s</span></div>
</div>

<div class="demo-observe-line">
  <span>elapsed <b>0.0 →</b></span>
  <span>segment <b>0 →</b></span>
  <span>pending upload <b>0 →</b></span>
</div>

<!--
配信開始を押します。
カメラとマイクのsample bufferをAVAssetWriterへ入れ、ファイルURLではなくdelegateからfMP4のDataを受け取ります。

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
UploadCoordinatorはinit.mp4の成功を覚え、これが済むまでmedia segmentを受け付けません。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSSegmentRecorder.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSUploadCoordinator.swift
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
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSUploadCoordinator.swift
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
  <div class="viewer-live"><b>LIVE</b><span>Safari / hls.js</span></div>
</div>

<div class="demo-observe-line">
  <span>STREAM <b>stream-...</b></span>
  <span>SEGMENTS <b>1 → 2 → 3</b></span>
  <span>PLAYLIST <b>/streams/.../playlist.m3u8</b></span>
</div>

<!--
MacのViewerは1秒ごとにstream一覧を取得し、更新時刻が最も新しい配信を選びます。
SafariはネイティブHLS、それ以外は同梱したhls.jsで再生します。

[Sources]
- iosdc2026HLSSample/server/static/app.js
-->

---
class: demo-step
---

<div class="kicker">DEMO RESULT · REAL OUTPUT</div>

# デモ後に残る、3種類のHLSファイル

<div class="real-output-tree">
  <div class="tree-root">server/data/streams/<b>stream-20260818-225315-C7A2AB11/</b></div>
  <div class="tree-columns">
    <div><code>├── init.mp4</code><small>1,136 bytes</small></div>
    <div><code>├── playlist.m3u8</code><small>682 bytes</small></div>
    <div><code>└── seg/</code><small>18 files</small></div>
    <div class="segment-list"><code>000001.m4s</code><code>000002.m4s</code><code>…</code><code>000018.m4s</code></div>
  </div>
</div>

<div class="bottom-claim">サーバーに残るのは、変換前の映像ではなく「再生可能なHLSオブジェクト」</div>

<!--
停止時は、Writerを閉じて最後のsegmentを受け取ったあと、AsyncStreamをfinishします。
すべてのイベントを処理した最後にENDLIST付きplaylistをPUTします。

これはリポジトリに残っている実際の出力です。
initが1つ、更新されるplaylistが1つ、2秒単位のm4sが18個あります。本番では同じ相対構造をS3 prefixへ置きます。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleHLSStreamer.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSUploadCoordinator.swift
- iosdc2026HLSSample/server/data/streams/stream-20260818-225315-C7A2AB11
-->

---

<div class="kicker">BIG PICTURE · FOUR BOUNDARIES</div>

# 今回の実装は、4つの境界を順番に越える

<div class="four-boundaries">
  <div><b>01 Capture</b><span>camera / mic → sample</span></div>
  <i>→</i>
  <div><b>02 Package</b><span>sample → fMP4</span></div>
  <i>→</i>
  <div><b>03 Publish</b><span>Data → object</span></div>
  <i>→</i>
  <div><b>04 Play</b><span>playlist → Viewer</span></div>
</div>

<div class="bottom-claim">今回の難所は、各境界で「順序」を失わないこと</div>

<!--
[Timing checkpoint: 10:00]

大枠は4段階です。
Capture、fMP4化、オブジェクト公開、playlist追従再生。境界を越えるたびに、時刻か順序の整合性が必要になります。
-->

---

<div class="kicker">ROUTE</div>

# sample bufferをS3公開まで追う

<div class="route">
  <div><b>01</b><span>HLSの最小形</span></div>
  <i></i>
  <div><b>02</b><span>iOSの責務分割</span></div>
  <i></i>
  <div><b>03</b><span>Captureと時刻</span></div>
  <i></i>
  <div><b>04</b><span>fMP4 segment</span></div>
  <i></i>
  <div><b>05</b><span>S3 uploadとcommit</span></div>
</div>

<div class="repo-line">Public at iOSDC 2026 / iosdc2026HLSSample</div>

<!--
HLSの最小形を確認し、Capture、Writer、S3 uploadの順に本番の処理を追います。
iosdc2026HLSSampleは、iOSDCのタイミングでpublic repositoryとして公開します。
発表では、iOSの生成処理を拡大して読むために使います。

映像バイト列のdata planeです。
iPhoneが完成済みのHLS断片をPUTし、Playerは同じオブジェクトをGETします。サーバーは再エンコードしません。
-->

---

<div class="kicker">SAME SHAPE, DIFFERENT DESTINATION</div>

# サンプルと本番は、同じHLSを別の場所へ置く

<div class="environment-map">
  <div class="environment-row sample">
    <b>Sample</b><span>iPhone</span><i>HTTP PUT</i><span>Mac filesystem</span><i>HTTP GET</i><span>Local Viewer</span>
  </div>
  <div class="environment-row production">
    <b>MomentNow</b><span>iPhone</span><i>presigned PUT</i><span>Amazon S3</span><i>CloudFront</i><span>Web Viewer</span>
  </div>
</div>

<div class="bottom-claim">init.mp4とm4sのバイト列は、どちらもiPhoneが生成する</div>

<!--
公開サンプルと本番の対応です。
サンプルはMacのfilesystemとViewer、本番はS3とCloudFrontです。端末が生成するHLSの構造は変わりません。

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

<div class="bottom-claim">「保存できた」と「再生リストに載せた」を分ける</div>

<!--
control planeはplaylist更新です。
segment objectが存在することを確認してから、playlistへURIを追加します。これが、途中の404を防ぐ公開契約です。

本番ではstreamIdごとのprefixへinitとm4sを置きます。
playlistはcommit APIが更新し、CloudFront経由のViewerは同じ相対URIをたどります。

本番のcommit APIが生成する初期playlistです。
この時点ではmedia segmentがなくても、playlistの種類、target duration、initの場所は決まっています。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSUploadCoordinator.swift
- MomentNow-Lambda/src/commit.ts
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

<div class="bottom-claim">ftyp / moov / moof / mdatというbox名はAppendixで扱う</div>

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

<div class="source">AVAssetWriterDelegate / LiveHLSStreamer / HLSUploadCoordinator</div>

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

# この仕組みを<br>iOSのコードへ分解する
<p>From a working stream to the AVFoundation pipeline</p>

<!--
[Timing checkpoint: 13:30]

HLSの全体像と冒頭で動かしたiosdc2026HLSSampleを、画面からHTTP PUTまで順に分解します。
本番固有の認証やAWS構成を外し、HLS生成と公開順序を追える形にしています。
-->

---

<div class="kicker">FOUR OWNERS</div>

# 本番は4つの型で、生成と公開を分ける

<div class="owner-lanes">
  <div><span>UI</span><b>LiveStreamViewModel</b><small>ready / streaming / completed</small></div>
  <div><span>ORCHESTRATE</span><b>LiveHLSStreamer</b><small>recorder callbackをuploadへ接続</small></div>
  <div><span>MEDIA</span><b>LiveSegmentRecorder</b><small>capture / writer / media clock</small></div>
  <div><span>UPLOAD</span><b>HLSUploadCoordinator</b><small>presign / PUT / commit</small></div>
</div>

<!--
画面は4領域です。
カメラ、サーバー接続、出力情報、playlist本文を同時に出します。
録画中にsegment数とpending uploadがどう変わるかを、実装を読む前に観察できます。

公開サンプルは、S3の代わりにMacのHTTPサーバーへ同じ形のオブジェクトをPUTします。
本番との差分は署名URL、Lambdaによるplaylist更新、CloudFrontです。iOSが作るinitとm4sは同じです。

UI状態、メディア状態、アップロード状態を分けています。
HLS固有の時刻処理もS3の公開順も、ViewModelへ漏らしません。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/ContentView.swift
- iosdc2026HLSSample/README.md
-->

---

<div class="kicker">ONE TAP</div>

# 配信開始タップで、生成とuploadをつなぐ

<div class="call-chain">
  <div><span>View</span><code>vm.start()</code></div>
  <b>→</b>
  <div><span>ViewModel</span><code>streamer.startRecording(stream:)</code></div>
  <b>→</b>
  <div><span>Streamer</span><code>onMediaSegment = { Task { ... } }</code></div>
  <b>→</b>
  <div><span>Uploader</span><code>presign → PUT → commit</code></div>
</div>

<div class="call-return">
  <span>AVAssetWriterDelegate callback</span>
  <i></i>
  <span>seq / Data / segmentReport</span>
</div>

<!--
タップはViewModelからStreamerへ入り、RecorderのcallbackをUploaderへ接続してからWriterを開始します。
以降はsegmentごとにTaskが作られ、presign、PUT、commitが進みます。

ViewModelのstreamingは、単なる端末録画ではありません。
segment生成とS3アップロードが続いている状態として扱い、stop後はpending uploadがなくなるまでcompletedへ進めません。
-->

---

<div class="kicker">CONCURRENCY MAP</div>

# Queueとactorは、守る状態が違う

<div class="concurrency-map">
  <div class="concurrency-row">
    <b>MainActor</b><span>ViewModelの表示状態</span><i class="cobalt"></i>
  </div>
  <div class="concurrency-row">
    <b>sessionQueue</b><span>AVCaptureSessionの構成・start / stop</span><i class="coral"></i>
  </div>
  <div class="concurrency-row">
    <b>writingQueue</b><span>sample順序・AVAssetWriter状態</span><i class="coral"></i>
  </div>
  <div class="concurrency-row">
    <b>HLSUploadCoordinator actor</b><span>upload数・commit状態・終了状態</span><i class="green"></i>
  </div>
</div>

<div class="bottom-claim">1本の巨大なロックではなく、責任ごとに直列化する</div>

<!--
CaptureSessionとWriterは別のserial queue、S3 uploadとcommitの状態はactorで守ります。
同じ「並行処理」でも、守る状態とAPIの制約が違うためです。

サンプルiOS側の責務分割です。
画面状態、配信の組み立て、AVFoundation、公開順序、HTTPを別の型にしています。ここから中央のRecorderを深掘りします。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample
-->

---

<div class="kicker">START ORDER</div>

# callbackを接続してから、Writerを開始する

```swift {3-8|10-13}
let channel = HLSUploadEventChannel()

recorder.onInitSegment = { [channel] data in
    channel.yield(.initialization(data))
}
recorder.onMediaSegment = { [channel, fragmentSeconds] seq, data, _ in
    channel.yield(.media(seq: seq, data: data, durationSec: fragmentSeconds))
}

uploadTask = Task {
    await coordinator.consume(channel.stream, channel: channel)
}
try await recorder.startRecording()
```

<!--
startRecordingでは、先にcallbackとconsumerを接続し、最後にRecorderを開始します。
順番を逆にすると、Writerがすぐ返したinit Dataを誰も受け取れない窓ができます。

Writer delegateは同期callbackです。
そこでDataをAsyncStreamへ渡してすぐ戻り、HTTP処理はconsumer側でawaitします。メディア処理とネットワーク待ちの境界です。

AsyncStream自体はqueue長を画面へ返しません。
サンプルではyield時に加算し、consumerが1イベントを処理するたびに減算してpending uploadを表示します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleHLSStreamer.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSUploadCoordinator.swift
-->

---
layout: center
class: chapter
---

<div class="chapter-no">03</div>
<div class="chapter-rule"></div>

# Captureが<br>メディアの時計を作る
<p>Camera, microphone, sample buffers, and one shared timeline</p>

<!--
[Timing checkpoint: 16:00]

ここからHLSSegmentRecorderを見ます。
最初の難所はエンコード設定ではなく、映像と音声を同じ時間軸へ載せることです。
-->

---

<div class="kicker">CAPTURE TOPOLOGY</div>

# AVCaptureSessionが、2つのdeviceを2つのDataOutputへつなぐ

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
    <div class="output-node"><b>VideoDataOutput</b><span>CMSampleBuffer</span></div>
    <div class="output-node"><b>AudioDataOutput</b><span>CMSampleBuffer</span></div>
  </div>
</div>

<div class="source">HLSSegmentRecorder.setupCaptureSessionLocked()</div>

<!--
カメラとマイクの権限をViewModelで揃えてから、プレビューを起動します。
権限の遷移をRecorderへ混ぜないことで、Recorderは許可済みの前提に集中できます。

PreviewはCaptureSession、RecordingはAVAssetWriterのライフサイクルです。
本番ではpreview準備とstreaming開始を分け、録画ボタンを押した時点でWriterとUploaderを動かします。

CaptureSessionには背面カメラとマイクを追加し、出力はDataOutputにします。
完成した動画ファイルではなく、フレームごとのCMSampleBufferを受け取るためです。
-->

---

<div class="kicker">WHY DATA OUTPUT</div>

# 完成ファイルではなく、sample bufferを使う

<div class="choice-compare">
  <div class="choice muted">
    <span>MovieFileOutput</span>
    <b>録画ファイルを作る</b>
    <small>高レベルで便利</small>
  </div>
  <div class="choice-arrow">→</div>
  <div class="choice selected">
    <span>Video / Audio DataOutput</span>
    <b>各sampleを受け取る</b>
    <small>Writerへ逐次append</small>
  </div>
</div>

<div class="sample-buffer-band">
  <span>video CMSampleBuffer</span>
  <span>audio CMSampleBuffer</span>
  <b>same writingQueue</b>
</div>

<!--
MovieFileOutputではなくDataOutputを使うのは、AVAssetWriterへsampleを逐次渡したいからです。
映像と音声のdelegateを同じwritingQueueへ載せます。
-->

---

<div class="kicker">AVFOUNDATION COMMON · INPUT FORMAT</div>

# CameraのNV12をWriterがH.264に圧縮

<div class="format-bridge">
  <div class="format-node raw"><span>Capture</span><b>420f / NV12</b><small>pixel buffer</small></div>
  <div class="format-arrow">append</div>
  <div class="format-node encoded"><span>Writer</span><b>H.264 High</b><small>fMP4 sample</small></div>
</div>

```swift
videoOutput.videoSettings = [
    kCVPixelBufferPixelFormatTypeKey as String:
        kCVPixelFormatType_420YpCbCr8BiPlanarFullRange
]
```

<div class="bottom-claim">ここはHLS固有ではなく、通常のcapture → encode処理</div>

<div class="source">HLSSegmentRecorder.setupCaptureSessionLocked()</div>

<!--
DataOutputからはNV12のpixel bufferを受け取ります。
この段階は未圧縮で、H.264への圧縮はAVAssetWriterInputのoutputSettingsが担当します。
-->

---

<div class="kicker">CALLBACK GATES</div>

# Writerへ届く前に、2つのguardで落とす

<div class="guard-funnel">
  <div class="guard-input">captureOutput(_:didOutput:from:)</div>
  <div class="guard-step"><code>isWriting == true</code><span>録画中だけ</span></div>
  <div class="guard-step"><code>CMSampleBufferDataIsReady</code><span>利用可能なsampleだけ</span></div>
  <div class="guard-output">startWriterIfNeeded → append</div>
</div>

```swift
guard isWriting else { return }
guard CMSampleBufferDataIsReady(sampleBuffer) else { return }
```

<!--
リアルタイム入力では、遅延したframeを無制限に保持するとメモリと遅延が増えます。
本番もlate frameを捨て、現在へ追いつく判断です。

CaptureSessionが動いていても、Writerへ渡すのは録画中だけです。
sample dataがreadyでない場合も早期returnし、Writerの状態遷移を単純に保ちます。
-->

---

<div class="kicker">SESSION START</div>

# 最初のvideo sampleで、全trackを開始

<div class="start-sequence">
  <div class="sequence-item audio"><span>audio sample</span><small>まだappendしない</small></div>
  <div class="sequence-line"></div>
  <div class="sequence-item video"><span>first video sample</span><small>startWriting</small></div>
  <div class="sequence-line active"></div>
  <div class="sequence-item session"><span>startSession(at: 10s)</span><small>video + audio共通</small></div>
</div>

```swift
guard output === videoOutput else { return }
guard !didStartSession, let writer else { return }
```

<!--
Writerのsessionは最初のvideo sampleで開始します。
先にaudioが届いても、共通の基準が決まるまではappendしません。
-->

---

<div class="kicker">PTS = SAMPLEの撮影時刻</div>

# sampleの撮影時刻を、Writerの開始時刻へ移す

<div class="dual-axis">
  <div class="axis-row">
    <b>Capture PTS<small>sampleに付く撮影時刻</small></b>
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

# 時刻補正は、全sampleへの平行移動

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
補正はレート変更ではなく、全sampleへの平行移動です。
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

<div class="kicker">APPEND READINESS</div>

# 詰まったsampleは、待たずに落とす

```swift
if output === videoOutput {
    guard videoInput.isReadyForMoreMediaData else { return }
    didAppend = videoInput.append(adjustedSampleBuffer)
} else {
    guard audioInput.isReadyForMoreMediaData else { return }
    didAppend = audioInput.append(adjustedSampleBuffer)
}
```

<div class="readiness-rule">
  <span>push source</span>
  <b>×</b>
  <span>real-time writer</span>
  <b>→</b>
  <span>bounded latency</span>
</div>

<div class="source">Apple: AVAssetWriterInput.isReadyForMoreMediaData</div>

<!--
DataOutputはpush型なので、Writer inputがreadyなときだけ直接appendします。
readyでなければ待機queueを作らず、そのsampleを落とします。品質よりリアルタイム性を優先する設計です。
-->

---
layout: center
class: chapter
---

<div class="chapter-no">04</div>
<div class="chapter-rule"></div>

# AVAssetWriterが<br>sampleをfMP4へ分ける
<p>Four HLS settings and a segment delegate</p>

<!--
[Timing checkpoint: 23:00]

Captureの時計が揃ったので、次はAVAssetWriterの設定とdelegate出力を見ます。
-->

---

<div class="kicker">URL-LESS WRITER</div>

# URLなしで、Dataをdelegateから受け取る

```swift
let writer = AVAssetWriter(contentType: .mpeg4Movie)
writer.shouldOptimizeForNetworkUse = true
writer.delegate = self
```

<div class="writer-output-model">
  <div class="writer-shape"><b>AVAssetWriter</b><span>MP4 content type</span></div>
  <div class="output-split">
    <span>init.mp4<br><small>再生準備</small></span>
    <span>media segment<br><small>約2秒のData</small></span>
  </div>
</div>

<div class="source">Apple WWDC20: AVAssetWriter can output fragmented MP4 data without an output URL</div>

<!--
通常のAVAssetWriterと違い、ここでは出力URLを渡しません。
HLS profileとdelegateを設定すると、初期化セグメントと分離可能なセグメントがDataで返ります。
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

<div class="source">LiveSegmentRecorder.setupWriters_locked()</div>

<!--
この4つがHLS segmentationの核です。
それぞれを1枚ずつ見ます。特にintervalは「必ず」ではなく「preferred」です。
-->

---

<div class="kicker">PREFERRED INTERVAL</div>

# 2秒は「希望」。境界は前後する

<div class="interval-axis">
  <div class="interval-target">
    <span>0s</span><i></i><span>2s</span><i></i><span>4s</span><i></i><span>6s</span>
  </div>
  <div class="interval-labels">
    <span>preferred</span><span>preferred</span><span>preferred</span>
  </div>
</div>

```swift
writer.preferredOutputSegmentInterval = CMTime(
    seconds: config.segmentSeconds,
    preferredTimescale: 600
)
```

<div class="bottom-claim warning">現在は2.0秒固定。segmentReport反映は改善項目</div>

<!--
property名の通り、2秒は希望間隔です。
境界にはキーフレームなどの条件があるため、実際の長さをsegment reportから取り出すのが安全です。
-->

---

<div class="kicker">KEYFRAME ALIGNMENT</div>

# 2秒で切るなら、2秒以内にIDRを用意する

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

<div class="source">Apple HLS Authoring Specification: video segments start with an IDR frame</div>

<!--
Playerがsegmentの先頭からデコードするにはIDRが必要です。
segment targetと同じ2秒でmax keyframe interval durationを指定し、境界を作れるようにします。
-->

---

<div class="kicker">SEGMENT DELEGATE</div>

# segmentTypeでinitとmediaを分ける

```swift {1-6}
switch segmentType {
case .initialization:
    onInitSegment?(segmentData)
case .separable:
    segmentIndex += 1
    onMediaSegment?(segmentIndex, segmentData, segmentReport)
@unknown default:
    break
}
```

<div class="source">LiveSegmentRecorder: AVAssetWriterDelegate</div>

<!--
delegateではsegmentTypeをswitchするだけです。
initializationはinit用、separableはmedia用としてcallbackへ渡します。

separable Dataはmoofとmdatを含むmedia segmentです。
AVAssetWriterがすでにHLS向けに分けているため、アプリ側で再muxしません。

AVAssetWriterはアプリ用のseqを決めません。
delegateで1から採番し、同じseqをpresignとcommitの両方へ渡します。
-->

---

<div class="kicker">GENERATION TIMELINE</div>

# Captureを止めず、約2秒ごとにDataを出す

<div class="lane-timeline upload-lanes">
  <div class="lane"><b>Capture</b><span class="continuous">sample sample sample sample sample</span></div>
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

# Writerが詰まっても、Captureを待たせない

<div class="backpressure-map">
  <div class="bp-source"><b>DataOutput</b><span>real-time push</span></div>
  <div class="bp-gate"><code>isReadyForMoreMediaData</code></div>
  <div class="bp-branches">
    <div class="bp-yes"><b>true</b><span>append</span></div>
    <div class="bp-no"><b>false</b><span>return / drop</span></div>
  </div>
</div>

<div class="tradeoff-line">
  <span>長所: latencyとmemoryがbounded</span>
  <span>代償: frame dropを観測していない</span>
</div>

<!--
Writerが詰まったときにsampleを貯めるqueueはありません。
遅延とメモリを制限できる一方、drop数を記録していません。
S3 uploadのbacklogとは別の層なので、Writer dropとpending uploadは別々に観測します。
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

# サンプルは、playlistのPUT成功後に状態を確定

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
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSUploadCoordinator.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HTTPHLSClient.swift
-->

---

<div class="kicker">DRAIN BEFORE FINISH</div>

# stopは、3つの非同期処理を順番に閉じる

<div class="drain-lanes">
  <div><b>1 Recorder</b><span>finishWriting完了を待つ</span><small>最後のcallbackが出る</small></div>
  <i>→</i>
  <div><b>2 Channel</b><span>continuation.finish()</span><small>新規eventを閉じる</small></div>
  <i>→</i>
  <div class="hot"><b>3 Consumer</b><span>uploadTask.value</span><small>ENDLIST PUTまで待つ</small></div>
</div>

<div class="bottom-claim">「撮影停止」と「配信終了」は、同じ瞬間ではない</div>

<!--
SampleHLSStreamer.stopRecordingの順序です。
Writerを閉じて最後のsegmentを受け取り、channelをfinishし、consumerがENDLIST付きplaylistをPUTするまで待ちます。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleHLSStreamer.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSUploadCoordinator.swift
-->

---
layout: center
class: chapter
---

<div class="chapter-no">05</div>
<div class="chapter-rule"></div>

# 端末で生成したDataを<br>S3へ順番に公開する
<p>Presign, PUT, commit, and finish from the iOS point of view</p>

<!--
[Timing checkpoint: 29:30]

ここから本番のiOS実装です。
AVAssetWriterDelegateのDataを、HLSUploadCoordinatorがS3へ公開する流れを追います。
-->

---

<div class="kicker">THREE REQUESTS · THIS PRODUCTION DESIGN</div>

# 本番は、presign → PUT → commitに分担

<div class="settings-table">
  <div class="settings-head"><span>REQUEST</span><span>DESTINATION</span><span>ROLE</span></div>
  <div><code>POST /presign</code><b>API</b><span>seq用PUT URLを発行</span></div>
  <div><code>PUT putUrl</code><b>S3</b><span>video/mp4 Dataを保存</span></div>
  <div><code>POST /commit</code><b>API</b><span>seqをplaylistへ公開</span></div>
  <div><code>GET playbackUrl</code><b>CloudFront</b><span>Viewerがplaylistを取得</span></div>
</div>

<div class="bottom-claim">commit APIはHLSの必須要素ではなく、今回選んだ公開制御</div>

<!--
presignは権限発行、PUTはbytes保存、commitは公開可否です。
HLSとして必要なのは、segmentの保存後にplaylistへ載せる順序です。今回はその公開確定をサーバーのcommit APIへ任せました。
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

<div class="source">LiveHLSStreamer.startRecording / HLSUploadCoordinator.uploadInitIfNeeded</div>

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

<div class="bottom-claim">存在するsegmentだけを、再生可能として公開する</div>

<!--
順序が重要です。
PUTの2xxを確認してからcommitします。逆なら、Playerがplaylistで見つけたURIをGETして404になります。
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
現在の本番実装はUploaderへisLastを通知し、actor内のpending uploadがゼロになるまでcompletedへ進めません。
ただし、onMediaSegmentで作ったTaskがactorに入る前はpending countへ反映されません。そのため、Taskが残っていてもpendingが0に見える余地があります。
公開サンプルはAsyncStreamのconsumer Taskを保持し、channel.finish後にTaskの終了までawaitします。完了契約としてはこちらのほうが明確です。
-->

---
layout: center
class: chapter
---

<div class="chapter-no">06</div>
<div class="chapter-rule"></div>

# iOSの非同期境界を<br>ライブ配信として閉じる
<p>Callback, task, upload backlog, and publication order</p>

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
      <small>次のsample処理へ</small>
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

<div class="kicker">SAMPLE → PRODUCTION</div>

# 本番では、HLSの外側に4つの制御を足す

<div class="production-additions">
  <div><b>AUTH</b><span>user / group ownership</span></div>
  <div><b>PRESIGN</b><span>object単位の短期PUT権限</span></div>
  <div><b>COMMIT</b><span>公開順とplaylistの同時更新を制御</span></div>
  <div><b>DELIVERY</b><span>CloudFront / ticket / status</span></div>
</div>

<div class="bottom-claim">AVAssetWriterの後ろを差し替えれば、同じiOS pipelineを使える</div>

<!--
iosdc2026HLSSampleはMacを小さなobject serverとして使います。
本番は先にpresignしてS3へPUTし、Lambdaのcommitでplaylistを更新します。生成するDataと保存順序は共通です。

サンプルから本番へ足すものです。
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

# 冒頭のデモで増えていたのは<br><span class="accent-coral">「ファイル」と「再生可能な順序」</span>

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
    <span>zenn.dev/hs7/articles/080eac650f65ba</span>
  </div>
  <img src="/assets/zenn-qr.png" alt="QR code for the related Zenn article" />
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
  <div class="source-row core"><b>HLSUploadCoordinator.swift</b><span>順序 / retry / finish</span></div>
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
本番ではNWPathと小さなprobe PUTから、high、medium、lowの1品質を配信開始前に選びます。
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
  <div class="start-marker"><b>10.00s</b><span>first adjusted video sample</span></div>
  <div class="start-media"><i></i><i></i><i></i><span>media timeline</span></div>
</div>

```swift
private let startTimeOffset = CMTime(value: 10, timescale: 1)
writer.initialSegmentStartTime = startTimeOffset
```

<!--
本番のHLS Writerも初期segmentの開始を10秒へ設定します。
Capture PTSの補正は、この指定と入力sampleを一致させるために必要でした。
-->

---

<div class="kicker">VIDEO SETTINGS</div>

# 配信用videoは、互換性と上り帯域を優先する

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
本番はH.264 portraitで、品質に応じて480pまたは720p、0.9から2.5Mbpsを選びます。
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

<div class="kicker">DURATION CAVEAT</div>

# 現在は2秒固定。実測時間は次の改善点

<div class="duration-compare">
  <div class="duration-side sample">
    <span>CURRENT</span>
    <b>durationSec = 2.0</b>
    <small>fragmentSecondsをcommit</small>
  </div>
  <div class="duration-side production">
    <span>NEXT</span>
    <b>segmentReportから実測</b>
    <small>EXTINFへ正確な長さ</small>
  </div>
</div>

```swift
recorder.onMediaSegment = { [uploader, fragmentSeconds] seq, data, _ in
    try? await uploader.uploadSegment(
        stream: stream, seq: seq, segmentData: data,
        durationSec: fragmentSeconds
    )
}
```

<!--
現在の本番実装もreportを受け取りますが、commitにはfragmentSecondsの2.0を渡しています。
実segmentは必ずしもぴったり2秒ではないため、reportからdurationを取り出すのが次の精度改善です。
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
segment保存が3回とも失敗した場合、playlist PUTへ進まず、Coordinatorのerrorとして残します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/HLSUploadCoordinator.swift
- iosdc2026HLSSample/ios/iosdc2026HLSSampleTests/HLSUploadCoordinatorTests.swift
-->

---

<div class="kicker">UI OBSERVATION LOOP</div>

# ViewModelは300msごとに、配信状態を更新

```swift {2-10}
monitorTask = Task {
    while !Task.isCancelled {
        elapsedSeconds = streamer.recordedSeconds
        if let snapshot = await streamer.currentSnapshot() {
            apply(snapshot)
        }
        try? await Task.sleep(for: .milliseconds(300))
    }
}
```

<div class="monitor-strip">
  <span>elapsed</span><span>segmentCount</span><span>pendingUploadCount</span><span>playlistText</span><span>error</span>
</div>

<!--
UIはdelegate callbackへ直接結びつけません。
MainActorのViewModelが300msごとにsnapshotを取得し、elapsed、segment数、pending、playlist、errorをまとめて反映します。

[Sources]
- iosdc2026HLSSample/ios/iosdc2026HLSSample/SampleStreamViewModel.swift
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
  <div class="immutable"><b>static assets</b><span>max-age=86400</span><code>Viewer UI</code></div>
</div>

<div class="bottom-claim compact">この区別は、S3 + CloudFrontでもそのまま使う</div>

<!--
playlistは同じURLの内容が増えるためキャッシュさせません。
initとm4sは一度置いたら変えず長期cacheし、PlayerのRange requestには206で返します。本番もobjectの可変性で方針を分けます。

[Sources]
- iosdc2026HLSSample/server/server.py
- MomentNow-Lambda/src/create_stream.ts
- MomentNow-Lambda/src/presign.ts
-->

---

<div class="kicker">OBSERVE THE UPLINK</div>

# iPhone側で、upload backlogと速度を測る

<div class="cache-table">
  <div class="cache-head"><span>METRIC</span><span>MEANING</span><span>USE</span></div>
  <div class="dynamic"><b>pendingSegmentCount</b><span>未完了upload数</span><code>stop待機 / backlog</code></div>
  <div class="dynamic"><b>averageRoundTripSec</b><span>presign + PUT</span><code>segment間隔と比較</code></div>
  <div class="immutable"><b>averageUploadDurationSec</b><span>PUT所要時間</span><code>回線劣化を検知</code></div>
  <div class="immutable"><b>averageUploadThroughputBps</b><span>送信bitrate</span><code>品質選択の根拠</code></div>
</div>

<div class="bottom-claim">平均は smoothingFactor = 0.25 の指数移動平均</div>

<!--
Coordinatorは各PUTのbytes、upload時間、presignからのround tripを記録しています。
segment生成よりuploadが遅い状態を、pending countとthroughputで端末側から観測できます。
-->
