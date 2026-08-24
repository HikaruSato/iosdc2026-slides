# iOSDC Japan 2026 slides

「映像変換サーバーなしでiPhone端末内でHLSを生成してライブ配信」の発表スライドです。
本編63枚、付録16枚の全79枚で、40分想定です。最初に、短い動画ファイルと更新されるplaylistによってHLSがライブ配信に
なる仕組みを説明します。その全体像へ今回の構成を当てはめたあと、`../iosdc2026HLSSample` を使い、
iPhoneでHLSを生成してMacのHTTPサーバーへPUTし、保存された `init.mp4`、`.m4s`、
`playlist.m3u8` をViewerで追従再生するデモを行います。その後、`../../MomentNow-iOS` のiOS実装を掘り下げ、
ローカルのファイル保存をS3・presigned PUT・commit・CloudFrontへ置き換えた本番構成まで説明します。

## Commands

```sh
npm install
npm run dev
npm run build
npm run export -- --output iosdc2026-hls-on-iphone.pdf
```

発表内容は `slides.md`、共通スタイルは `styles/index.css` にあります。
