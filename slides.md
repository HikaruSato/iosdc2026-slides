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
0:00

今日は、iPhone自身をエンコーダー兼セグメンターにして、生成したHLSをS3へ順次アップロードし、ライブ配信にした実装を話します。
焦点はiOSです。サーバー側は、iOSから見えるpresignとcommitの契約だけを扱います。
-->

---
layout: center
class: statement
---

<div class="kicker">TODAY'S RESULT</div>

# iPhoneが生成したHLSを<br><span class="accent-coral">撮影中からS3へ公開する</span>

<div class="result-flow end-to-end">
  <div class="result-step"><b>Capture</b><span>CMSampleBuffer</span></div>
  <div class="result-arrow">→</div>
  <div class="result-step strong"><b>AVAssetWriter</b><span>init.mp4 + m4s</span></div>
  <div class="result-arrow">→</div>
  <div class="result-step"><b>presigned PUT</b><span>S3 object</span></div>
  <div class="result-arrow">→</div>
  <div class="result-step"><b>commit</b><span>playlist → Viewer</span></div>
</div>

<!--
0:35

録画開始後、init.mp4を1回、m4sを約2秒ごとに生成します。
各DataをS3へPUTし、成功したseqだけをcommitすると、視聴者のplaylistが伸びていきます。
-->

---

<div class="kicker">SAMPLE DEMO</div>

# デモでは、S3へ送るDataをローカルファイルで観察する

<div class="sample-demo-grid">
  <div class="sample-screen">
    <span>iOSDC HLS Sample</span>
    <div class="sample-camera">Camera Preview</div>
    <b><i></i> 録画中</b>
  </div>
  <div class="sample-demo-arrow">→<small>delegate</small></div>
  <div class="sample-files">
    <div><span>start</span><b>init.mp4</b></div>
    <div><span>+2s</span><b>seg/000001.m4s</b></div>
    <div><span>+4s</span><b>seg/000002.m4s</b></div>
    <pre>#EXTM3U
#EXT-X-MAP:URI="init.mp4"
#EXTINF:2.000,
seg/000001.m4s</pre>
  </div>
</div>

<!--
1:05

サンプルで見せるのは、S3送信直前のinit Dataとmedia Dataです。
ネットワークを外してファイルへ保存することで、AVFoundationが何を返したかを直接確認できるようにしています。
-->

---

<div class="kicker">WHY</div>
<div class="story-split">
  <div>
    <h1>撮影後に待つほど、<br>共有したい瞬間から遠ざかる</h1>
    <p class="lead">子どもの動画を家族へ送る。<br>撮影中から届き始めれば、終了後の待ち時間を減らせる。</p>
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
1:40

きっかけは、子どもの動画を家族へ送るときの待ち時間でした。
撮影し終わってから大きなファイルを送るのではなく、撮影中から小さく届けたい。それがHLSを選んだ理由です。
-->

---

<div class="kicker">CONSTRAINT</div>

# 消したいのは、映像を変換し続けるサーバー

<div class="constraint-line">
  <div class="constraint-item">
    <span class="index">01</span>
    <b>常時稼働</b>
    <small>配信がなくても運用対象</small>
  </div>
  <div class="constraint-item">
    <span class="index">02</span>
    <b>トランスコード</b>
    <small>CPU / GPUと監視が必要</small>
  </div>
  <div class="constraint-item selected">
    <span class="index">03</span>
    <b>端末で生成</b>
    <small>撮影中のiPhoneに任せる</small>
  </div>
</div>

<div class="bottom-claim">用途を「少人数・短時間・単一品質」に絞る</div>

<!--
2:10

個人開発では、映像変換サーバーの常時運用が重い。
そこで用途を絞り、カメラを持っているiPhone自身にエンコードとセグメント生成を任せます。
-->

---

<div class="kicker">ARCHITECTURE</div>

# 変えるのは、セグメントを作る場所だけ

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
2:40

一般的な構成と比べると、変わるのはセグメント生成の場所です。
iPhoneが生成済みのHLSファイルをオブジェクトストレージへ置くので、バックエンドは映像バイト列を変換しません。
-->

---
layout: center
class: statement ink-statement
---

<div class="kicker">PRECISE WORDING</div>

# 「サーバーなし」ではなく<br><span class="accent-coral">「映像変換サーバーなし」</span>

<div class="boundary-line">
  <span>API</span><span>S3</span><span>CDN</span><span>状態管理</span>
  <b>は残る</b>
</div>

<!--
3:10

バックエンドがゼロという意味ではありません。
API、保存、配信、状態管理は残ります。なくすのは、映像を受け取り続けて変換する役割です。
-->

---

<div class="kicker">ROUTE</div>

# 1つのsample bufferを、S3公開まで追う

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

<div class="repo-line">github / iosdc2026HLSSample</div>

<!--
3:35

HLSの最小形を確認し、Capture、Writer、S3 uploadの順に本番の処理を追います。
サンプルコードは、iOSの生成処理を拡大して読むために使います。
-->

---
layout: center
class: chapter
---

<div class="chapter-no">01</div>
<div class="chapter-rule"></div>

# HLSは<br>ファイルと更新される目次
<p>HTTP Live Streaming in the smallest useful model</p>

<!--
4:00

最初にHLSの用語を、このサンプルで必要な範囲だけ揃えます。
-->

---

<div class="kicker">HLS IN ONE SENTENCE</div>

# 再生の入口は、動画ファイルではなくplaylist

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

<div class="source">RFC 8216: Media Playlist / Media Segment / EXT-X-MAP</div>

<!--
4:20

Playerはplaylistを取得し、そこに書かれた順でinitとsegmentを取得します。
ライブ中はplaylistの末尾へ新しいsegmentが増えます。
-->

---

<div class="kicker">THREE FILE TYPES</div>

# iPhoneが生成・公開するものは3種類

<div class="file-strip">
  <div class="file-type playlist-file">
    <span class="file-ext">M3U8</span>
    <b>playlist.m3u8</b>
    <small>更新される再生順</small>
  </div>
  <div class="file-type init-file">
    <span class="file-ext">MP4</span>
    <b>init.mp4</b>
    <small>codec / track情報</small>
  </div>
  <div class="file-type media-file">
    <span class="file-ext">M4S</span>
    <b>000001.m4s</b>
    <small>2秒前後の映像と音声</small>
  </div>
</div>

<div class="bottom-claim">playlistが、初期化情報とメディア断片を結びつける</div>

<!--
4:45

目次、初期化セグメント、メディアセグメントの3種類です。
initだけにも、m4sだけにも、完全な再生体験はありません。playlistが関係を定義します。
-->

---

<div class="kicker">OBJECT PREFIX = STREAM</div>

# S3では、1つのprefixが1配信

<div class="tree-layout">
  <div class="tree-root">live/hls/streams/<b>{streamId}</b>/</div>
  <div class="tree-branch">
    <div><span>├──</span><b>init.mp4</b><small>1回</small></div>
    <div><span>├──</span><b>playlist.m3u8</b><small>segmentごとに更新</small></div>
    <div><span>├──</span><b>status.json</b><small>配信状態</small></div>
    <div><span>└──</span><b>seg/</b></div>
    <div class="indent"><span>├──</span><b>000001.m4s</b></div>
    <div class="indent"><span>└──</span><b>000002.m4s</b></div>
  </div>
</div>

<div class="source">S3 object key contract / streamId prefix</div>

<!--
5:10

本番ではstreamIdごとのprefixへinitとm4sを置きます。
playlistはcommit APIが更新し、CloudFront経由のViewerは同じ相対URIをたどります。
-->

---

<div class="kicker">PLAYLIST ANATOMY</div>

# 5つのタグで、再生契約が決まる

<div class="manifest-annotated">
  <pre>#EXTM3U
#EXT-X-VERSION:7
#EXT-X-TARGETDURATION:2
#EXT-X-PLAYLIST-TYPE:EVENT
#EXT-X-MAP:URI="init.mp4"
#EXT-X-MEDIA-SEQUENCE:1</pre>
  <div class="annotation-list">
    <div><code>VERSION</code><span>使うタグの互換レベル</span></div>
    <div><code>TARGETDURATION</code><span>segment長の上限基準</span></div>
    <div><code>EVENT</code><span>末尾へ追加するplaylist</span></div>
    <div><code>MAP</code><span>fMP4の初期化セグメント</span></div>
    <div><code>MEDIA-SEQUENCE</code><span>最初の番号</span></div>
  </div>
</div>

<div class="source">RFC 8216 / commit APIが生成するMedia Playlist</div>

<!--
5:40

本番のcommit APIが生成する初期playlistです。
この時点ではmedia segmentがなくても、playlistの種類、target duration、initの場所は決まっています。
-->

---

<div class="kicker">EVENT LIFECYCLE</div>

# EVENTは「追記」され、最後に一度だけ閉じる

<div class="event-timeline">
  <div class="event-phase">
    <span class="time">t = 0</span>
    <b>header</b>
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

<div class="source">Apple: Event playlist construction / RFC 8216 §4.3.3</div>

<!--
6:05

EVENT playlistでは、既存segmentを消さずに末尾へ追加します。
停止時にENDLISTを付けると、Playerへこれ以上増えないことを伝えられます。
-->

---

<div class="kicker">FRAGMENTED MP4</div>

# fMP4は「設定」と「再生可能な断片」を分ける

<div class="box-anatomy">
  <div class="box-group init-group">
    <div class="iso-box"><b>ftyp</b><span>file type</span></div>
    <div class="iso-box wide"><b>moov</b><span>tracks / codec</span></div>
    <small>init.mp4</small>
  </div>
  <div class="box-plus">+</div>
  <div class="box-group media-group">
    <div class="iso-box"><b>moof</b><span>fragment metadata</span></div>
    <div class="iso-box wide"><b>mdat</b><span>encoded samples</span></div>
    <small>000001.m4s</small>
  </div>
</div>

<div class="source">Apple WWDC20: Author fragmented MPEG-4 content with AVAssetWriter</div>

<!--
6:30

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
      <li>separable m4s</li>
    </ul>
  </div>
  <div class="ownership-divider">/</div>
  <div class="ownership-side app-side">
    <span>App</span>
    <b>seq + PUT + commit</b>
    <ul>
      <li>sequence番号</li>
      <li>S3 upload</li>
      <li>公開可能なseqを通知</li>
    </ul>
  </div>
</div>

<div class="source">AVAssetWriterDelegate / LiveHLSStreamer / HLSUploadCoordinator</div>

<!--
6:55

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

# サンプルで<br>iOSの生成処理だけを拡大する
<p>The same AVFoundation pipeline, with local files as a microscope</p>

<!--
7:20

ここからサンプルでiOSの生成処理を拡大します。
ローカル保存は最終構成ではなく、AVFoundationの出力を観察するための差し替えです。
-->

---

<div class="kicker">SAMPLE UI</div>

# 画面上で、カメラ・ファイル・playlistを同時に見る

<div class="sample-ui-map">
  <div class="ui-preview"><span>CameraPreviewView</span><b>capture preview</b></div>
  <div class="ui-controls"><span>start / stop</span><span>AVPlayer</span><span>WebView</span></div>
  <div class="ui-output">
    <div><b>streamId</b><span>output directory</span></div>
    <div><b>segment count</b><span>elapsed seconds</span></div>
  </div>
  <div class="ui-playlist"><code>#EXTM3U ...</code><span>playlist.m3u8</span></div>
</div>

<div class="source">ContentView.swift</div>

<!--
7:40

画面は4領域です。
カメラ、操作、出力情報、playlist本文を同時に出し、録画中にsegment数が増えることを確認できます。
-->

---

<div class="kicker">INTENTIONAL SCOPE</div>

# ローカル保存は、S3送信前のDataを見るためだけ

<div class="scope-ledger">
  <div class="scope-in">
    <b>IN</b>
    <span>camera / microphone</span>
    <span>AVAssetWriter</span>
    <span>init / media Data</span>
    <span>sequence番号</span>
    <span>playlist規則</span>
  </div>
  <div class="scope-out">
    <b>OUT</b>
    <span>presign / S3 PUT</span>
    <span>API / Lambda</span>
    <span>SwiftData</span>
    <span>purchase / sharing</span>
    <span>viewer metrics</span>
  </div>
</div>

<div class="bottom-claim">本番との差分は、callback後の保存先と公開処理</div>

<!--
8:05

サンプルはネットワーク失敗を混ぜず、AVFoundationから出たDataとplaylist規則を確認する教材です。
発表の完成形は、このDataをHLSUploadCoordinatorがS3へ送る本番実装です。
-->

---

<div class="kicker">SOURCE TREE</div>

# 8ファイルで、生成から再生まで完結する

<div class="source-tree">
  <div class="source-row core"><b>HLSSegmentRecorder.swift</b><span>capture / encode / segment</span></div>
  <div class="source-row core"><b>LocalHLSStreamer.swift</b><span>recorderとstoreを接続</span></div>
  <div class="source-row core"><b>LocalHLSStreamStore.swift</b><span>directory / file write</span></div>
  <div class="source-row core"><b>LocalHLSManifest.swift</b><span>playlist state</span></div>
  <div class="source-row"><b>SampleStreamViewModel.swift</b><span>permission / UI state</span></div>
  <div class="source-row"><b>ContentView.swift</b><span>操作と観測</span></div>
  <div class="source-row"><b>LocalHLSPlayerView.swift</b><span>AVPlayer</span></div>
  <div class="source-row"><b>LocalHLSWebPreview.swift</b><span>WKWebView</span></div>
</div>

<!--
8:30

中心は上の4ファイルです。
後半4つは、操作と再生確認を担当します。これ以降も、画面から下へ呼び出し順に追います。
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
8:55

UI状態、メディア状態、アップロード状態を分けています。
HLS固有の時刻処理もS3の公開順も、ViewModelへ漏らしません。
-->

---

<div class="kicker">ONE TAP</div>

# 配信開始タップで、RecorderとUploaderを接続する

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
9:20

タップはViewModelからStreamerへ入り、RecorderのcallbackをUploaderへ接続してからWriterを開始します。
以降はsegmentごとにTaskが作られ、presign、PUT、commitが進みます。
-->

---

<div class="kicker">UI STATE</div>

# streamingは「生成中かつアップロード中」

<div class="state-machine">
  <div class="state-node"><b>idle</b><small>未準備</small></div>
  <div class="state-arrow">permission</div>
  <div class="state-node"><b>ready</b><small>preview中</small></div>
  <div class="state-arrow">start</div>
  <div class="state-node live"><b>streaming</b><small>segment生成 + S3 upload</small></div>
  <div class="state-arrow">stop</div>
  <div class="state-node done"><b>completed</b><small>pending upload待機済み</small></div>
</div>

<div class="error-branch">どの非同期処理からも <code>error(String)</code> へ遷移</div>

<div class="source">LiveStreamViewModel.State</div>

<!--
9:45

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
10:10

CaptureSessionとWriterは別のserial queue、S3 uploadとcommitの状態はactorで守ります。
同じ「並行処理」でも、守る状態とAPIの制約が違うためです。
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
10:35

ここからHLSSegmentRecorderを見ます。
最初の難所はエンコード設定ではなく、映像と音声を同じ時間軸へ載せることです。
-->

---

<div class="kicker">CONFIG</div>

# 上り回線に合わせて、配信品質を3段階から選ぶ

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
10:55

本番ではNWPathと小さなprobe PUTから、high、medium、lowの1品質を配信開始前に選びます。
途中でrenditionを切り替えるABRではなく、端末の上り回線に合わせた開始時の選択です。
-->

---

<div class="kicker">PERMISSION FIRST</div>

# 権限がそろうまで、CaptureSessionを起動しない

<div class="permission-flow">
  <div class="permission-node"><b>video</b><span>authorized?</span></div>
  <div class="permission-and">AND</div>
  <div class="permission-node"><b>audio</b><span>authorized?</span></div>
  <div class="permission-arrow">→</div>
  <div class="permission-result"><b>startPreview()</b><span>state = ready</span></div>
</div>

```swift
let cameraAllowed = await requestCameraPermission()
let micAllowed = await requestMicrophonePermission()
guard cameraAllowed, micAllowed else { ... }
```

<div class="source">LiveStreamViewModel.requestPermissionsAndSetup()</div>

<!--
11:20

カメラとマイクの権限をViewModelで揃えてから、プレビューを起動します。
権限の遷移をRecorderへ混ぜないことで、Recorderは許可済みの前提に集中できます。
-->

---

<div class="kicker">PREVIEW ≠ RECORDING</div>

# カメラ起動とHLS生成は、別の操作

<div class="two-phase">
  <div class="phase-block">
    <span>PHASE 1</span>
    <b>startPreview()</b>
    <small>CaptureSessionを構成してstartRunning</small>
  </div>
  <div class="phase-gap">then</div>
  <div class="phase-block active">
    <span>PHASE 2</span>
    <b>startRecording()</b>
    <small>AVAssetWriterを準備してisWriting = true</small>
  </div>
</div>

<div class="bottom-claim">プレビューを見せたまま、Writerだけ開始・終了できる</div>

<!--
11:45

PreviewはCaptureSession、RecordingはAVAssetWriterのライフサイクルです。
本番ではpreview準備とstreaming開始を分け、録画ボタンを押した時点でWriterとUploaderを動かします。
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
12:10

CaptureSessionには背面カメラとマイクを追加し、出力はDataOutputにします。
完成した動画ファイルではなく、フレームごとのCMSampleBufferを受け取るためです。
-->

---

<div class="kicker">AUDIO SESSION</div>

# カメラ構成より先に、録音用AudioSessionを有効化する

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
12:55

録音権限だけではなく、AVAudioSessionのcategoryとmodeを先に設定します。
Bluetooth HFPを含む入力routeを許可しつつ、端末側の再生はspeakerを既定にしています。
-->

---

<div class="kicker">WHY DATA OUTPUT</div>

# 完成ファイルではなく、sample bufferが欲しい

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
13:20

MovieFileOutputではなくDataOutputを使うのは、AVAssetWriterへsampleを逐次渡したいからです。
映像と音声のdelegateを同じwritingQueueへ載せます。
-->

---

<div class="kicker">INPUT FORMAT</div>

# CameraのNV12を、WriterがH.264へ圧縮する

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

<div class="source">HLSSegmentRecorder.setupCaptureSessionLocked()</div>

<!--
13:45

DataOutputからはNV12のpixel bufferを受け取ります。
この段階は未圧縮で、H.264への圧縮はAVAssetWriterInputのoutputSettingsが担当します。
-->

---

<div class="kicker">REAL-TIME PRESSURE</div>

# 遅れたframeは、無制限に溜めない

<div class="pressure-visual">
  <div class="frame-train">
    <span>F1</span><span>F2</span><span>F3</span><span>F4</span><span>F5</span>
  </div>
  <div class="pressure-gate">writingQueue</div>
  <div class="frame-result">
    <span class="kept">append</span>
    <span class="dropped">late frameをdrop</span>
  </div>
</div>

```swift
videoOutput.alwaysDiscardsLateVideoFrames = true
```

<div class="source">Apple: alwaysDiscardsLateVideoFrames / TN2445</div>

<!--
14:10

リアルタイム入力では、遅延したframeを無制限に保持するとメモリと遅延が増えます。
本番もlate frameを捨て、現在へ追いつく判断です。
-->

---

<div class="kicker">ORIENTATION</div>

# 縦向きは、video connectionへ90度で指定する

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
14:30

出力サイズはportraitで指定し、video connectionへ90度のrotation angleを設定します。
配信中の向き変更は扱わず、portrait 90度で固定します。
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
14:55

CaptureSessionが動いていても、Writerへ渡すのは録画中だけです。
sample dataがreadyでない場合も早期returnし、Writerの状態遷移を単純に保ちます。
-->

---

<div class="kicker">SESSION START</div>

# 最初のvideo sampleが、全trackの開始を決める

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
15:20

Writerのsessionは最初のvideo sampleで開始します。
先にaudioが届いても、共通の基準が決まるまではappendしません。
-->

---

<div class="kicker">TWO CLOCKS</div>

# Capture PTSは、そのままHLSの開始時刻ではない

<div class="dual-axis">
  <div class="axis-row">
    <b>Capture PTS</b>
    <div class="axis-line"><span class="axis-value source-value">58342.31</span><i></i><i></i><i></i><em>…</em></div>
  </div>
  <div class="axis-transform">− firstVideoPTS + 10s</div>
  <div class="axis-row target">
    <b>HLS timeline</b>
    <div class="axis-line"><span class="axis-value target-value">10.00</span><i></i><i></i><i></i><em>…</em></div>
  </div>
</div>

<div class="source">CMSampleBuffer presentationTimeStamp / writer.initialSegmentStartTime</div>

<!--
15:45

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
16:10

補正はレート変更ではなく、全sampleへの平行移動です。
最初のvideo PTSからdeltaを一度だけ決め、その後は映像と音声へ同じ値を足します。
-->

---

<div class="kicker">COPY TIMING</div>

# Media bytesは変えず、timing metadataをコピーする

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

<div class="source">HLSSegmentRecorder.swift / CMSampleBuffer.offsettingTiming(by:)</div>

<!--
16:35

CMSampleBufferの映像・音声データはそのままに、timing infoを差し替えたコピーを作ります。
PTSだけでなく、有効なDTSも同じ量だけ補正します。
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
17:00

音声用の開始時刻を別に決めると、A/Vの相対差が変わります。
videoから決めたdeltaを両方へ適用し、同期を保ちます。
-->

---

<div class="kicker">APPEND READINESS</div>

# Writerが詰まったsampleは、待たずに落とす

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
17:25

DataOutputはpush型なので、Writer inputがreadyなときだけ直接appendします。
readyでなければ待機queueを作らず、そのsampleを落とします。品質よりリアルタイム性を優先する設計です。
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
17:50

終了時刻も補正後のPTSを使います。
finishWritingによって最後のsegmentがdelegateへ届く可能性があるため、stopはその完了まで待ちます。
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
18:15

Captureの時計が揃ったので、次はAVAssetWriterの設定とdelegate出力を見ます。
-->

---

<div class="kicker">URL-LESS WRITER</div>

# ファイルURLを渡さず、Dataをdelegateで受け取る

```swift
let writer = AVAssetWriter(contentType: .mpeg4Movie)
writer.shouldOptimizeForNetworkUse = true
writer.delegate = self
```

<div class="writer-output-model">
  <div class="writer-shape"><b>AVAssetWriter</b><span>MP4 content type</span></div>
  <div class="output-split">
    <span>initialization Data</span>
    <span>separable Data</span>
  </div>
</div>

<div class="source">Apple WWDC20: AVAssetWriter can output fragmented MP4 data without an output URL</div>

<!--
18:35

通常のAVAssetWriterと違い、ここでは出力URLを渡しません。
HLS profileとdelegateを設定すると、初期化セグメントと分離可能なセグメントがDataで返ります。
-->

---

<div class="kicker">FOUR HLS KNOBS</div>

# HLS出力は、4つのpropertyで有効になる

<div class="setting-list">
  <div><b>01</b><code>outputFileTypeProfile</code><span>Apple HLS向けfMP4</span></div>
  <div><b>02</b><code>preferredOutputSegmentInterval</code><span>希望するsegment間隔</span></div>
  <div><b>03</b><code>initialSegmentStartTime</code><span>最初のsegment開始時刻</span></div>
  <div><b>04</b><code>delegate</code><span>生成されたDataの受け口</span></div>
</div>

<div class="source">LiveSegmentRecorder.setupWriters_locked()</div>

<!--
19:00

この4つがHLS segmentationの核です。
それぞれを1枚ずつ見ます。特にintervalは「必ず」ではなく「preferred」です。
-->

---

<div class="kicker">PREFERRED INTERVAL</div>

# 2秒は「希望」。実際の境界はmedia条件にも依存する

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
19:45

property名の通り、2秒は希望間隔です。
境界にはキーフレームなどの条件があるため、実際の長さをsegment reportから取り出すのが安全です。
-->

---

<div class="kicker">INITIAL START TIME</div>

# Writerの最初のsegmentを、10秒起点へ固定する

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
20:10

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
20:35

本番はH.264 portraitで、品質に応じて480pまたは720p、0.9から2.5Mbpsを選びます。
端末保存品質ではなく、ネットワークへ継続的に送れる配信品質として設定します。
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

<div class="source">Apple HLS Authoring Specification: video segments start with an IDR frame</div>

<!--
21:00

Playerがsegmentの先頭からデコードするにはIDRが必要です。
segment targetと同じ2秒でmax keyframe interval durationを指定し、境界を作れるようにします。
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
21:20

音声はAAC、mono、44.1kHzで、low 48、medium 64、high 96kbpsです。
家族向けの短い映像という用途に合わせ、stereoより送信量を優先しています。
-->

---

<div class="kicker">SEGMENT DELEGATE</div>

# segmentTypeだけで、initとmediaを分けられる

```swift {7-12}
func assetWriter(
    _ writer: AVAssetWriter,
    didOutputSegmentData segmentData: Data,
    segmentType: AVAssetSegmentType,
    segmentReport: AVAssetSegmentReport?
) {
    switch segmentType {
    case .initialization:
        onInitSegment?(segmentData)
    case .separable:
        segmentIndex += 1
        onMediaSegment?(segmentIndex, segmentData, segmentReport)
    @unknown default:
        break
    }
}
```

<div class="source">LiveSegmentRecorder: AVAssetWriterDelegate</div>

<!--
21:45

delegateではsegmentTypeをswitchするだけです。
initializationはinit用、separableはmedia用としてcallbackへ渡します。
-->

---

<div class="kicker">SEPARABLE MEDIA</div>

# その後のcallbackは、番号付きm4sへ変わる

<div class="segment-output">
  <div class="segment-callback media-callback">
    <span>segmentType</span>
    <b>.separable</b>
  </div>
  <div class="segment-arrow">→</div>
  <div class="segment-data">
    <span>Data</span>
    <b>moof + mdat</b>
  </div>
  <div class="segment-arrow">→</div>
  <div class="segment-file">
    <span>presigned PUT</span>
    <b>S3 / seg / seq=1</b>
  </div>
</div>

<div class="bottom-claim">Dataを再変換せず、そのまま保存またはuploadできる</div>

<!--
22:30

separable Dataはmoofとmdatを含むmedia segmentです。
AVAssetWriterがすでにHLS向けに分けているため、アプリ側で再muxしません。
-->

---

<div class="kicker">APPLICATION SEQUENCE</div>

# 再生順の番号は、delegate callbackごとに自分で付ける

<div class="sequence-counter">
  <div class="counter-source">.separable</div>
  <div class="counter-op"><code>segmentIndex += 1</code></div>
  <div class="counter-values">
    <span>1</span><span>2</span><span>3</span><span>4</span>
  </div>
</div>

<div class="name-rule">
  <code>presign(kind: "seg", seq: seq)</code>
  <span>→</span>
  <code>commit(seq: seq)</code>
</div>

<!--
22:55

AVAssetWriterはアプリ用のseqを決めません。
delegateで1から採番し、同じseqをpresignとcommitの両方へ渡します。
-->

---

<div class="kicker">DURATION CAVEAT</div>

# 現在は2.0秒固定。segment reportは次の精度改善点

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

<div class="source">LiveHLSStreamer.startRecording / AVAssetSegmentReport</div>

<!--
23:20

現在の本番実装もreportを受け取りますが、commitにはfragmentSecondsの2.0を渡しています。
実segmentは必ずしもぴったり2秒ではないため、reportからdurationを取り出すのが次の精度改善です。
-->

---

<div class="kicker">GENERATION TIMELINE</div>

# Captureを止めず、2秒前後ごとにDataが外へ出る

<div class="lane-timeline upload-lanes">
  <div class="lane"><b>Capture</b><span class="continuous">sample sample sample sample sample</span></div>
  <div class="lane"><b>Writer</b><span></span><span class="fragment">seg 1</span><span class="fragment">seg 2</span><span class="fragment">seg 3</span></div>
  <div class="lane"><b>Delegate</b><span class="init-mark">init</span><span class="callback-mark">Data 1</span><span class="callback-mark">Data 2</span><span class="callback-mark">Data 3</span></div>
  <div class="lane"><b>S3</b><span class="init-mark">PUT init</span><span class="callback-mark">PUT 1</span><span class="callback-mark">PUT 2</span><span class="callback-mark">PUT 3</span></div>
</div>

<div class="bottom-claim">Writerのfinishを待たず、生成とS3 uploadを並行できる</div>

<!--
23:45

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
24:10

Writerが詰まったときにsampleを貯めるqueueはありません。
遅延とメモリを制限できる一方、drop数を記録していません。
S3 uploadのbacklogとは別の層なので、Writer dropとpending uploadは別々に観測します。
-->

---

<div class="kicker">VISIBLE OUTPUT</div>

# 最初の再生可能状態は、init PUTとsegment commitの後

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

<div class="manifest-preview">
  <code>init object exists</code><code>seg 1 object exists</code>
  <code>playlist contains seq 1</code><code>Viewer can start</code>
</div>

<!--
24:35

AVAssetWriterがDataを返しただけでは、まだ視聴者は再生できません。
initと最初のm4sがS3に存在し、seq 1のcommitでplaylistへ載った時点が最初の再生可能状態です。
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
25:00

ここから本番のiOS実装です。
AVAssetWriterDelegateのDataを、HLSUploadCoordinatorがS3へ公開する流れを追います。
-->

---

<div class="kicker">PREPARE BEFORE MEDIA</div>

# segmentが来る前に、streamIdと再生URLを準備する

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
25:25

配信画面へ入る前にstreamは作成済みです。
ViewModelはticket、権限、previewを揃えてreadyへ進み、録画開始後すぐuploadできる状態を作ります。
-->

---

<div class="kicker">PRODUCTION COMPONENTS</div>

# iOS側は「生成」「接続」「公開」を分ける

<div class="owner-lanes">
  <div><span>MEDIA</span><b>LiveSegmentRecorder</b><small>Data callbackを生成</small></div>
  <div><span>BRIDGE</span><b>LiveHLSStreamer</b><small>callbackをTaskへ変換</small></div>
  <div><span>UPLOAD</span><b>HLSUploadCoordinator</b><small>presign / PUT / commit</small></div>
  <div><span>HTTP</span><b>HLSAPIClient</b><small>URLSessionとAPI contract</small></div>
</div>

<div class="bottom-claim">AVFoundationのqueueへネットワーク待ちを持ち込まない</div>

<!--
25:50

Recorderは同期callbackを返し、StreamerがTaskを作ってUploaderへ渡します。
AVFoundationのwritingQueueをURLSessionの完了待ちで塞がない責務分割です。
-->

---

<div class="kicker">INITIALIZATION SEGMENT</div>

# init.mp4は、最初のcallbackで1回だけPUTする

```swift
recorder.onInitSegment = { [weak self] data in
    Task {
        try? await self?.uploader.uploadInitIfNeeded(
            stream: stream, initData: data
        )
    }
}

guard !didUploadInit else { return }
let presign = try await api.presign(
    streamId: stream.streamId, kind: "init", seq: nil
)
try await api.put(to: URL(string: presign.putUrl)!, data: initData,
                  contentType: "video/mp4")
didUploadInit = true
```

<div class="source">LiveHLSStreamer.startRecording / HLSUploadCoordinator.uploadInitIfNeeded</div>

<!--
26:40

initialization callbackは1配信で最初に届きます。
Coordinator側でもdidUploadInitを持ち、重複PUTを防いでいます。
-->

---

<div class="kicker">MEDIA SEGMENT</div>

# media Dataごとに、presign → PUT → commit

```swift
let presign = try await api.presign(
    streamId: stream.streamId,
    kind: HLSUploadKind.segment,
    seq: seq
)
try await api.put(
    to: URL(string: presign.putUrl)!,
    data: segmentData,
    contentType: HLSContentType.segmentM4S
)
_ = try await api.commit(
    streamId: stream.streamId, seq: seq,
    durationSec: dur, isLast: false,
    targetDurationSec: targetDurationSec
)
```

<!--
27:05

presign APIから、seqに対応する一時PUT URLを取得します。
S3 PUTが成功した後だけcommitし、サーバーへplaylistへ載せてよいseqを通知します。
-->

---

<div class="kicker">NO AWS CREDENTIAL ON DEVICE</div>

# iPhoneへ渡すのは、短時間だけ有効なPUT URL

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

```swift
var request = URLRequest(url: presignedURL)
request.httpMethod = "PUT"
request.setValue(contentType, forHTTPHeaderField: "Content-Type")
request.httpBody = data
```

<!--
27:30

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
27:55

順序が重要です。
PUTの2xxを確認してからcommitします。逆なら、Playerがplaylistで見つけたURIをGETして404になります。
-->

---

<div class="kicker">ACTOR STATE</div>

# Actorは、uploadと終了の状態をまとめて守る

<div class="concurrency-map">
  <div class="concurrency-row">
    <b>didUploadInit</b><span>init.mp4の重複PUTを防ぐ</span><i class="green"></i>
  </div>
  <div class="concurrency-row">
    <b>lastCommittedSeq</b><span>公開済み最大seq</span><i class="cobalt"></i>
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
28:20

Coordinatorはactorですが、presignやPUTのawait中に別segmentの処理が進みます。
そのため到着順や完了順ではなく、seqとpending countを明示的な状態として持ちます。
-->

---

<div class="kicker">STOP CONTRACT</div>

# stop後は<br>pending uploadが0になるまで完了にしない

<div class="publish-order four-step">
  <div class="publish-step"><b>1</b><span>recorder.stop()</span></div>
  <div class="publish-arrow">→</div>
  <div class="publish-step"><b>2</b><span>uploader.finish()</span></div>
  <div class="publish-arrow">→</div>
  <div class="publish-step"><b>3</b><span>pending == 0を待つ</span></div>
  <div class="publish-arrow">→</div>
  <div class="publish-step"><b>4</b><span>state = completed</span></div>
</div>

```swift
while await streamer.hasPendingUploads() {
    try? await Task.sleep(for: .milliseconds(500))
}
```

<div class="source">LiveStreamViewModel.stop()</div>

<!--
28:45

録画停止は、ネットワーク送信完了と同義ではありません。
UploaderへisLastを通知し、pending uploadがゼロになるまで画面をcompletedへ進めません。
-->

---

<div class="kicker">WHY THE SAMPLE IS LOCAL</div>

# ローカルサンプルは、本番のupload手前を可視化する

<div class="playback-compare">
  <div class="playback-path">
    <span>SAMPLE</span>
    <b>Data.write</b>
    <small>init / m4sを直接観察</small>
  </div>
  <div class="playback-path">
    <span>PRODUCTION iOS</span>
    <b>presigned PUT</b>
    <small>同じDataをS3へ送る</small>
  </div>
  <div class="playback-path production">
    <span>VIEWER</span>
    <b>CloudFront URL</b>
    <small>commit済みseqを再生</small>
  </div>
</div>

<div class="bottom-claim">サンプルのローカル保存は、最終アーキテクチャではない</div>

<!--
29:10

サンプルはS3やAPIを再現するものではありません。
本番と共通なのはRecorderまでで、callback後はローカル保存ではなくUploaderへ接続します。
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
29:35

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
30:00

AVAssetWriterDelegateはwritingQueue上で呼ばれます。
callback内ではTaskを作るだけにし、ネットワークawaitはSwift Concurrency側へ逃がします。
-->

---

<div class="kicker">THREE REQUESTS</div>

# presign・PUT・commitは、役割が違う

<div class="settings-table">
  <div class="settings-head"><span>REQUEST</span><span>DESTINATION</span><span>ROLE</span></div>
  <div><code>POST /presign</code><b>API</b><span>seq用PUT URLを発行</span></div>
  <div><code>PUT putUrl</code><b>S3</b><span>video/mp4 Dataを保存</span></div>
  <div><code>POST /commit</code><b>API</b><span>seqをplaylistへ公開</span></div>
  <div><code>GET playbackUrl</code><b>CloudFront</b><span>Viewerがplaylistを取得</span></div>
</div>

<div class="bottom-claim">iOSはplaylist本文を書かず、「公開してよいseq」をcommitする</div>

<!--
30:30

presignは権限発行、PUTはbytes保存、commitは公開可否です。
責任を分けることで、iOSへAWS credentialもplaylist更新競合も持ち込みません。
-->

---

<div class="kicker">OUT-OF-ORDER RACE</div>

# 2秒ごとのuploadは、完了順まで保証しない

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

<div class="bottom-claim warning">commit APIは、欠番より先をplaylistへ出さない契約が必要</div>

<!--
31:00

actorでもawait中は別segmentが進むため、seg 2がseg 1より先にPUT完了する可能性があります。
iOSはseqを必ず送り、commit APIは連続した番号だけをplaylistへ出す契約にします。
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
31:30

Coordinatorは各PUTのbytes、upload時間、presignからのround tripを記録しています。
segment生成よりuploadが遅い状態を、pending countとthroughputで端末側から観測できます。
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
32:00

この構成は万能ではありません。
少人数・短時間・単一品質には合います。大規模、長時間、ABR、厳しいSLAなら専用のmedia platformを選びます。
-->

---
layout: center
class: closing
---

<div class="kicker">TAKEAWAYS</div>

# 成立条件は、4つの順序をそろえること

<div class="takeaway-grid">
  <div><b>01</b><span><strong>Clock</strong>Capture PTSをWriterのtimelineへ移す</span></div>
  <div><b>02</b><span><strong>Boundary</strong>IDRとsegment intervalをそろえる</span></div>
  <div><b>03</b><span><strong>Upload</strong>DataをS3へPUTしてからcommitする</span></div>
  <div><b>04</b><span><strong>Finish</strong>pendingが0になるまで完了にしない</span></div>
</div>

<div class="closing-footer">
  <div>
    <b>ありがとうございました</b>
    <span>zenn.dev/hs7/articles/080eac650f65ba</span>
  </div>
  <img src="/assets/zenn-qr.png" alt="QR code for the related Zenn article" />
</div>

<!--
32:40 - 39:30（デモ・質疑バッファを含む）

まとめです。
AVAssetWriterDelegateでiPhoneからfMP4を逐次取り出し、presigned PUTでS3へ送り、成功したseqをcommitします。
Clock、Boundary、Upload、Finishの順序が揃って初めて、録画ではなくライブ配信になります。

ローカルサンプルは、そのうちAVFoundationの生成処理を見える形にした教材です。ありがとうございました。
-->
