# iOSDC Japan 2026 slides

「映像変換サーバーなしでiPhone端末内でHLSを生成してライブ配信」の発表スライドです。
全72枚（本編67枚・末尾の補足5枚）で、40分想定です。枚数削減は行わず、通常は本編を話します。
Optional detailは時間超過時だけの予備であり、省略を前提にした時間配分ではありません。
ライブデモ成功時は代替説明5枚だけを省略し、62枚とデモで進めます。失敗時はライブ操作を切り上げ、代替5枚で説明します。
最初に、撮影後のアップロードを待たずにURLで共有したいという動機と、
Apple公式サンプルをきっかけにiPhone内でHLSを生成できた経緯を説明します。続いて、短い動画ファイルと更新されるplaylistによって
HLSがライブ配信になる仕組みを説明し、その全体像へ今回の構成を当てはめたあと、`../iosdc2026HLSSample` を使い、
iPhoneでHLSを生成してMacのHTTPサーバーへPUTし、保存された `init.mp4`、`.m4s`、
`playlist.m3u8` をViewerで追従再生するデモを行います。その後、`AVCaptureSession`、DataOutput、
`AVAssetWriter`、iOS 26以降のSampleBufferReceiver、Writer delegateをどう接続するとHLS用Dataを取り出せるかを、
約2秒のfragment生成を含めて順番に説明します。時刻の扱いはApple公式サンプルの方針を参考にしており、
本編では「映像と音声を同じ量だけずらし、時間差を保つ」という数値例1枚にまとめています。
最後に、保存できたsegmentだけをplaylistへ公開する順序と、
公開サンプルのMac保存をMomentNowのS3・署名付きURL・commit・CloudFrontへ置き換えた構成を扱います。

品質・ビットレートの選択、アップロード失敗時のリトライ、キャッシュ制御は必ず話します。
SafariとAVPlayerでの再生確認にも、デモ代替ページを省略する場合を含めて触れます。
デモの前に写真への追加権限を許可します。配信停止後は保存成功の表示を待ち、写真アプリで保存動画を再生します。
公開サンプルも撮影データを2つのWriterへ渡し、配信用H.264（720×1280）と保存用HEVC（1080×1920・5 Mbps、AAC 96 kbps）を並行生成します。
Sampleは停止後に写真へ自動保存し、MomentNowは元動画uploadと設定に応じた写真保存を行います。
写真保存とHLSのENDLIST公開はそれぞれの結果を確認します。写真保存が失敗した場合はアプリから再試行します。
デモ直前の権限ページで、カメラ・マイク・写真への追加と、同じLANのMacへ接続する際のローカルネットワーク権限を説明します。
写真は追加のみを要求し、Sampleでは配信開始の必須条件です。用途を記すInfo.plistキーと権限要求APIはノートにまとめています。
AudioSessionの設定はCaptureSessionのページのノートで補足します。画面回転、詳細なencode設定、guard条件などのOptional detailは、時間超過時に限って省略できる予備です。
Writer側のCMSampleBufferのdropと、生成済みsegmentの送信待ちは別の問題として説明します。
サンプルには送信待ちの明示的な上限がなく、上限と超過時の停止判断は今後の改善です。
実durationの反映と公開順の説明では、公開サンプル・MomentNowの現状・未実装の改善案を区別します。
commit APIはMomentNowが選んだ責務配置であり、HLS必須のAPIとは扱いません。
時間軸の独立章は設けず、Writerの開始処理、10秒の余白とAAC priming、時刻の計算式、コピー処理のコードは
発表終了後の補足にまとめています。

## 通し練習

ノートのTiming checkpointはすべて練習用の仮目安であり、実測済みの所要時間ではありません。
デモ込みで35〜37分を目標に通して話し、実測後にチェックポイントを調整します。
40分に収まるかは枚数だけで判断せず、図を理解する間やコードの説明時間も含めて確認します。
デモが成功する場合と、失敗して代替説明へ切り替える場合の両方を練習します。

## Commands

```sh
npm install
npm run dev
npm run build
npm run export -- --output iosdc2026-hls-on-iphone.pdf
```

発表内容は `slides.md`、共通スタイルは `styles/index.css` にあります。
