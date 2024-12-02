---
title: "Web Audio API + React でシンセサイザーを作ってみよう"
emoji: "🎹"
type: "tech"
topics: ["React", "WebAudioAPI", "AudioContext"]
published: false
---

## はじめに

こんにちは！今年（2024年）7月に株式会社ミラボに入社し、エンジニアとして働いている 梅澤 です。

弊社業務の一環で、Web Audio API のことを知り、興味を持ったので、私自身の知見を深める意味でも本記事を執筆してみることにしました。

本記事では、Web Audio API と React を使った簡単なシンセサイザーアプリを作成し、その仕組みを学んでいきたいと思います。

## Web Audio API

MDN Web Docs の [Web Audio API](https://developer.mozilla.org/ja/docs/Web/API/Web_Audio_API) のページを見ると、以下のように説明されています。

> ウェブオーディオ API はウェブ上で音声を扱うための強力で多機能なシステムを提供します。これにより開発者は音源を選択したり、エフェクトを加えたり、視覚効果を加えたり、パンニングなどの特殊効果を適用したり、他にもたくさんのいろいろなことができるようになります。

使い方のイメージとしては、上記ページにも記載されていますが、

1. Audio Context を作成する
2. オシレーター、ストリームの音源（Source Node）を作成する
3. 音声を加工するノード（Effect Node）を作成する
4. 音声の出力先（Destination Node）を作成する
5. 各ノードを接続する

という流れになります。

以下は、Web Audio API の全体イメージになります。

![Web Audio API のイメージ](/images/web_audio_api.drawio.png)

## サンプルアプリの作成を通して学ぶ

サンプルアプリの画面イメージです。

![サンプルアプリの画面イメージ](/images/react_simple_synthesizer.png)

サンプルアプリの完成版デモは、以下リンクでご覧いただけます。  
https://react-simple-synthesizer.vercel.app/

ソースコードは、GitHubで公開しています。  
https://github.com/plumchang/react-simple-synthesizer

サンプルアプリでは、以下の機能を実装しています。

- 音波形の選択（サイン波、矩形波など）
- 周波数の調整
  - 音階モードでは、音階とオクターブの選択
- 音量の調整
- ADSRエンベロープの適用
- リバーブエフェクトの追加

（余談ですが、本サンプルサプリは、ほぼほぼ Vercelの生成AIサービスである v0 を使って作成しています。）

### AudioContext の作成

`AudioContext`は、ブラウザでオーディオ処理を行うためのインターフェースです。音声の生成、加工、再生を管理します。

```typescript
// AudioContextの作成
const audioContext = new (window.AudioContext || window.webkitAudioContext)();
```

### オーディオノードの基本

- **OscillatorNode**：音波（サイン波、矩形波など）を生成します。
- **GainNode**：音量を調整します。


### オシレーターの実装とサウンド生成

#### オシレーターの作成

```typescript
const oscillator = audioContext.createOscillator();
oscillator.type = 'sine'; // 波形タイプを指定
oscillator.frequency.setValueAtTime(440, audioContext.currentTime); // 周波数を設定
```

#### 音量の調整

```typescript
const gainNode = audioContext.createGain();
gainNode.gain.setValueAtTime(0.5, audioContext.currentTime); // 音量を設定
```

#### ノードの接続と再生

```typescript
oscillator.connect(gainNode);
gainNode.connect(audioContext.destination);
oscillator.start();
```

### ADSRエンベロープの実装

#### ADSRとは

- **Attack**：音が出始めてから最大音量に達するまでの時間
- **Decay**：最大音量から持続音量に減衰するまでの時間
- **Sustain**：持続音量
- **Release**：鍵盤を離してから音が消えるまでの時間

#### エンベロープの適用

```typescript
const now = audioContext.currentTime;
envelopeGainNode.gain.cancelScheduledValues(now);
envelopeGainNode.gain.setValueAtTime(0, now);
envelopeGainNode.gain.linearRampToValueAtTime(1, now + attack);
envelopeGainNode.gain.linearRampToValueAtTime(sustain, now + attack + decay);
```


### リバーブなどのエフェクト追加

#### リバーブの仕組み

リバーブは、音の残響効果を再現するエフェクトです。`ConvolverNode`を使って実装します。

#### インパルスレスポンスの生成

```typescript
const impulseLength = 2 * audioContext.sampleRate;
const impulse = audioContext.createBuffer(2, impulseLength, audioContext.sampleRate);
const left = impulse.getChannelData(0);
const right = impulse.getChannelData(1);

for (let i = 0; i < impulseLength; i++) {
  left[i] = (Math.random() * 2 - 1) * Math.pow(1 - i / impulseLength, 2);
  right[i] = (Math.random() * 2 - 1) * Math.pow(1 - i / impulseLength, 2);
}

convolverNode.buffer = impulse;
```

#### エフェクトの適用

```typescript
envelopeGainNode.connect(dryGainNode);
envelopeGainNode.connect(convolverNode);
convolverNode.connect(wetGainNode);
dryGainNode.connect(gainNode);
wetGainNode.connect(gainNode);
```


### ユーザーインタラクションとUIの強化

#### ノートとオクターブの選択

```typescript
const notes: Note[] = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"];
```

- ノートボタンを作成し、クリック時に対応する周波数を計算して再生します。

#### マウスやタッチイベントの処理

```typescript
const handleNotePress = (noteToPlay: Note) => (event: React.MouseEvent | React.TouchEvent) => {
  event.preventDefault();
  setNote(noteToPlay);
  startSound(noteToPlay);
  
  const handleRelease = () => {
    stopSound();
    document.removeEventListener("mouseup", handleRelease);
    document.removeEventListener("touchend", handleRelease);
  };
  
  document.addEventListener("mouseup", handleRelease);
  document.addEventListener("touchend", handleRelease);
};
```


## まとめ

最後までお読みいただき、ありがとうございました！  
本記事では、Web Audio APIを使ってシンセサイザーアプリを開発し、ブラウザ上で様々な音声処理を実行できることを学べました。  
Web Audio API には、本記事でご紹介したもの以外にも、様々な音声処理機能が存在しているようですので、興味を持たれた方は、ぜひ MDNのドキュメント等で調べてみてください！

## 参考

- [Web Audio API - MDN Web Docs](https://developer.mozilla.org/ja/docs/Web/API/Web_Audio_API)
- [react-simple-synthesizer](https://github.com/plumchang/react-simple-synthesizer)



