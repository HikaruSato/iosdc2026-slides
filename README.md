# iOSDC Japan 2026 slides

「映像変換サーバーなしでiPhone端末内でHLSを生成してライブ配信」の発表スライドです。
全74枚、40分想定です。`../../MomentNow-iOS` のHLS生成・S3アップロード実装を本編とし、
`../iosdc2026HLSSample` はAVFoundationの生成処理をローカルで観察するための説明用サンプルとして扱います。

## Commands

```sh
npm install
npm run dev
npm run build
npm run export -- --output iosdc2026-hls-on-iphone.pdf
```

発表内容は `slides.md`、共通スタイルは `styles/index.css` にあります。
