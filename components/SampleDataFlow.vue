<template>
  <svg class="sample-data-flow" viewBox="0 0 1152 460" role="img" aria-labelledby="sample-flow-title sample-flow-desc">
    <title id="sample-flow-title">公開サンプルの撮影・HLS配信・写真保存のデータフロー</title>
    <desc id="sample-flow-desc">カメラとマイクからAVCaptureSessionを経てCMSampleBufferを受け取る。同じ撮影データを2つのWriterへ渡す。HLS用Writerはinit.mp4とm4sを生成し、HLSStreamPublisherがセグメント、プレイリストの順にMacへPUTする。ブラウザがGETして再生する。保存用Writerは別のMP4を生成し、停止後に写真ライブラリへ保存する。</desc>
    <defs>
      <marker id="sample-arrow-ink" markerWidth="9" markerHeight="9" refX="8" refY="4.5" orient="auto"><path d="M0 0 L9 4.5 L0 9" fill="var(--accent-secondary)" /></marker>
      <marker id="sample-arrow-blue" markerWidth="9" markerHeight="9" refX="8" refY="4.5" orient="auto"><path d="M0 0 L9 4.5 L0 9" fill="var(--accent-primary)" /></marker>
      <marker id="sample-arrow-green" markerWidth="9" markerHeight="9" refX="8" refY="4.5" orient="auto"><path d="M0 0 L9 4.5 L0 9" fill="var(--success)" /></marker>
    </defs>

    <g class="capture">
      <rect x="2" y="2" width="278" height="52" /><text x="141" y="36">カメラ・マイク</text>
      <path class="arrow" d="M280 28 H342" />
      <rect x="350" y="2" width="365" height="52" /><text x="532.5" y="36">AVCaptureSession</text>
      <path class="arrow" d="M715 28 H777" />
      <rect x="785" y="2" width="365" height="52" /><text x="967.5" y="36">CMSampleBuffer</text>
    </g>

    <g class="hls">
      <path class="arrow" d="M967.5 54 V80 H310 V103" />
      <rect x="30" y="110" width="560" height="74" />
      <text class="strong" x="310" y="141">HLS用 AVAssetWriter</text>
      <text x="310" y="171">H.264 ＋ AAC</text>
      <path class="arrow" d="M310 184 V205" />
      <rect x="30" y="212" width="560" height="46" />
      <text x="310" y="244">init.mp4 ／ .m4s</text>
      <path class="arrow" d="M310 258 V284" />
      <rect x="30" y="291" width="560" height="72" />
      <text class="strong" x="310" y="322">HLSStreamPublisher</text>
      <text x="310" y="351">HTTP PUT：segment → playlist</text>
      <path class="arrow" d="M310 363 V382 H146 V393" />
      <rect x="2" y="400" width="290" height="58" />
      <text x="147" y="437">Mac HTTPサーバー</text>
      <path class="arrow" d="M292 429 H348" />
      <text class="edge-label" x="324" y="414">GET</text>
      <rect x="356" y="400" width="295" height="58" />
      <text x="503.5" y="437">ブラウザでHLS再生</text>
    </g>

    <g class="local">
      <path class="arrow" d="M967.5 54 V103" />
      <rect x="758" y="110" width="392" height="74" />
      <text class="strong" x="954" y="141">保存用 AVAssetWriter</text>
      <text x="954" y="171">HEVC ＋ AAC</text>
      <path class="arrow" d="M954 184 V254" />
      <rect x="758" y="261" width="392" height="62" />
      <text class="strong" x="954" y="301">完成したMP4</text>
      <path class="arrow" d="M954 323 V393" />
      <text class="edge-label" x="1040" y="365">停止後に保存</text>
      <rect x="758" y="400" width="392" height="58" />
      <text class="strong" x="954" y="437">写真ライブラリ</text>
    </g>
  </svg>
</template>

<style scoped>
.sample-data-flow { display: block; width: 100%; margin-top: 20px; overflow: visible; }
text { fill: currentColor; font-family: "Hiragino Sans", "Noto Sans JP", sans-serif; font-size: 26px; text-anchor: middle; }
.strong { font-weight: 700; }
rect { fill: var(--paper); stroke: currentColor; stroke-width: 2; rx: 5; }
.arrow { fill: none; stroke: currentColor; stroke-width: 2; }
.capture { color: var(--accent-secondary); }
.capture .arrow { marker-end: url(#sample-arrow-ink); }
.hls { color: var(--accent-primary); }
.hls rect { fill: var(--accent-primary-soft); }
.hls .arrow { marker-end: url(#sample-arrow-blue); }
.local { color: var(--success); }
.local rect { fill: var(--success-soft); }
.local .arrow { marker-end: url(#sample-arrow-green); }
</style>
