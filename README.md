# iOSDC Japan 2026 slides

「映像変換サーバーなしでiPhone端末内でHLSを生成してライブ配信」の発表スライドです。
全73枚で、40分想定です。補足ページも関係する章の中へ配置し、当日の残り時間に応じてスピーカーノートを見ながら省略できます。
最初に、撮影後のアップロードを待たずにURLで共有したいという動機と、
Apple公式サンプルをきっかけにiPhone内でHLSを生成できた経緯を説明します。続いて、短い動画ファイルと更新されるplaylistによって
HLSがライブ配信になる仕組みを説明し、その全体像へ今回の構成を当てはめたあと、`../iosdc2026HLSSample` を使い、
iPhoneでHLSを生成してMacのHTTPサーバーへPUTし、保存された `init.mp4`、`.m4s`、
`playlist.m3u8` をViewerで追従再生するデモを行います。その後、`AVCaptureSession`、DataOutput、
`AVAssetWriter`、iOS 26以降のSampleBufferReceiver、Writer delegateをどう接続するとHLS用Dataを取り出せるかを、
時刻補正と約2秒のfragment生成を含めて順番に説明します。最後に、保存できたsegmentだけをplaylistへ公開する順序と、
公開サンプルのMac保存をMomentNowのS3・署名付きURL・commit・CloudFrontへ置き換えた構成を扱います。

Chapter 02はCapture callbackからWriter delegateまでの必須導線を優先し、AudioSession、画面回転、品質選択、
詳細なencode設定、guard条件は時間に応じて省略できます。

## Commands

```sh
npm install
npm run dev
npm run build
npm run export -- --output iosdc2026-hls-on-iphone.pdf
```

発表内容は `slides.md`、共通スタイルは `styles/index.css` にあります。
