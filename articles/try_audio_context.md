---
title: "Web Audio API + React でシンセサイザーを作ってみよう"
emoji: "🎹"
type: "tech"
topics: ["React", "WebAudioAPI", "AudioContext"]
published: false
publication_name: "milabo"
---

## はじめに

こんにちは！今年（2024年）7月に株式会社ミラボに入社し、エンジニアとして働いている 梅澤 です。

弊社業務の一環で、Web Audio API のことを知り、興味を持ったので、私自身の知見を深める意味でも本記事を執筆してみることにしました。

本記事では、Web Audio API と React を使った簡単なシンセサイザーアプリを作成し、その仕組みを学んでいきたいと思います。

:::message
誤解がないように細くしておくと、シンセサイザーアプリと弊社業務に直接関係はございません。あくまで、筆者の興味とWeb Audio API の学習のためのサンプルアプリとして作成した形となります🙇‍♂️
:::

## Web Audio API

MDN Web Docs の [Web Audio API](https://developer.mozilla.org/ja/docs/Web/API/Web_Audio_API) のページを見ると、以下のように説明されています。

> ウェブオーディオ API はウェブ上で音声を扱うための強力で多機能なシステムを提供します。これにより開発者は音源を選択したり、エフェクトを加えたり、視覚効果を加えたり、パンニングなどの特殊効果を適用したり、他にもたくさんのいろいろなことができるようになります。

### 使い方のイメージ
上記ページにも記載されていますが、

1. Audio Context を作成する
2. オシレーター、ストリームの音源（Source Node）を作成する
3. 音声を加工するノード（Effect Node）を作成する
4. 音声の出力先（Destination Node）を作成する
5. 各ノードを接続する

という流れになります。

以下は、Web Audio API の全体イメージになります。

![Web Audio API のイメージ](/images/web_audio_api.drawio.png)

### 各ブラウザの対応状況

Web Audio API の基本的な機能範囲であれば、モバイルブラウザも含め、ほとんどのモダンブラウザで利用できそうです。  
詳細は、[MDNドキュメント](https://developer.mozilla.org/ja/docs/Web/API/Web_Audio_API#%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%83%BC%E3%81%AE%E4%BA%92%E6%8F%9B%E6%80%A7)で確認できます。

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

### シンプルな音を鳴らしてみる
まずは、基本的なオシレーターのサイン波を鳴らしてみましょう。

#### OscillatorNodeの作成

`OscillatorNode`は、特定の波形で音を生成するノードです。

```typescript
// AudioContextの作成
const audioContext = new (window.AudioContext || window.webkitAudioContext)();

// OscillatorNodeの作成
const oscillator = audioContext.createOscillator();
```

- オシレーターノードはデフォルトでサイン波（sine wave）を生成します。

#### GainNodeを使った音量調整

音量を調整するために`GainNode`を作成します。

```typescript
// GainNodeの作成
const gainNode = audioContext.createGain();

// 初期音量の設定
gainNode.gain.setValueAtTime(0.5, audioContext.currentTime);
```

- `gainNode.gain.setValueAtTime`を使って、音量を設定します。
  - 第一引数は音量で、0.0から1.0の範囲で指定します。
  - 第二引数は時間で、`audioContext.currentTime`を指定すると、即座に音量が変更されます。

#### ノードの接続

作成したノードを接続し、最終的に音声を出力します。

```typescript
// オシレーターの出力をゲインノードに接続
oscillator.connect(gainNode);
// ゲインノードの出力をスピーカーなどの規定の出力デバイスに接続
gainNode.connect(audioContext.destination);
```

#### 音を鳴らす

最後に、オシレーターを開始して音を鳴らします。

```typescript
// オシレーターの開始
oscillator.start();

// オシレーターの停止
oscillator.stop();
```

#### オシレーターの周波数の変更

オシレーターの周波数を変更すると、音程が変わります。

```typescript
oscillator.frequency.setValueAtTime(440, audioContext.currentTime);
```

- `oscillator.frequency.setValueAtTime`を使って、音量を設定します。
  - 第一引数は周波数です。
  - 第二引数は時間で、`audioContext.currentTime`を指定すると、即座に音量が変更されます。

#### ここまでのサンプル

@[codesandbox](https://codesandbox.io/embed/y2n84w?view=editor+%2B+preview&module=%2Fsrc%2FReactSimpleSynthesizer1.tsx)


### 波形の種類の変更

オシレーターの波形を変更するには、`oscillator.type`を変更します。  
波形の種類を変更すると、音の鳴り方が変わります。

```typescript
oscillator.type = 'square'; // "sine" | "square" | "sawtooth" | "triangle"
```

#### ここまでのサンプル

波形の種類を変更できるようにしたサンプルです。  
サイン波の他に、矩形波（square）、鋸歯状波（sawtooth）、三角波（triangle）を選択できます。  
少々耳障りに聞こえる可能性もあるため、注意してください🙏

@[codesandbox](https://codesandbox.io/embed/73q2cd?view=editor+%2B+preview&module=%2Fsrc%2FReactSimpleSynthesizer2.tsx)

### 音階の選択

これまでは周波数を直接指定して音を鳴らしてきましたが、音楽的には**音階（ノート）**と**オクターブ**で音程を指定できると便利です。ここでは、音階とオクターブを選択して音を鳴らす機能を実装してみましょう。

音階とオクターブから対応する周波数を計算する必要があります。一般的に、音階ごとの周波数は以下の式で計算できるようです。

$$
\text{周波数} = 440 \times 2^{\frac{n}{12}}
$$

（ $n$ は基準音（A4、440Hz）からの半音の数になります）

参考までに、Wikipediaのリンクを貼っておきます。  
https://ja.wikipedia.org/wiki/%E5%B9%B3%E5%9D%87%E5%BE%8B

#### 音階から周波数への変換

上記の式を使って、音階とオクターブを入力として、対応する周波数を計算する関数を作成すると、以下のようになります。

```typescript
type Note = "C" | "C#" | "D" | "D#" | "E" | "F" | "F#" | "G" | "G#" | "A" | "A#" | "B";

const noteToFrequency = (note: Note, octave: number): number => {
  const notes: Note[] = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"];
  const baseFrequency = 440; // A4の周波数
  const baseOctave = 4;
  const baseNoteIndex = notes.indexOf("A");

  const noteIndex = notes.indexOf(note);
  const semitoneDifference = (octave - baseOctave) * 12 + (noteIndex - baseNoteIndex);

  return baseFrequency * Math.pow(2, semitoneDifference / 12);
};
```

#### 周波数の指定

計算した周波数をオシレーターに設定すれば、選択した音階とオクターブに基づいて音を鳴らすことができます。

```typescript
const frequency = noteToFrequency(selectedNote, selectedOctave);
oscillator.frequency.setValueAtTime(frequency, audioContext.currentTime);
```

#### ここまでのサンプル

以下のサンプルでは、音階とオクターブを選択して音を鳴らすことができます。

@[codesandbox](https://codesandbox.io/embed/6ply4f?view=editor+%2B+preview&module=%2Fsrc%2FReactSimpleSynthesizer3.tsx)


### ADSRエンベロープとリバーブエフェクトの追加

最後に、エフェクトとして、**ADSRエンベロープ**と**リバーブエフェクト**を追加してみます。

#### ADSRエンベロープ

**ADSR**とは、音の強弱や持続時間を制御するためのパラメータで、以下の4つの要素から構成されています。

- **Attack（アタック）**：音が出始めてから最大音量に達するまでの時間
- **Decay（ディケイ）**：最大音量から持続音量に減衰するまでの時間
- **Sustain（サステイン）**：持続音量（音が鳴っている間の音量）
- **Release（リリース）**：鍵盤を離してから音が消えるまでの時間

![ADSRエンベロープのイメージ](/images/adsr.drawio.png)

つまりは、音量を時間に応じて変化させるエフェクトになります。

##### エンベロープの実装

まず、エンベロープ用の`GainNode`を作成します。

```typescript
// エンベロープ用のGainNodeを作成
const envelopeGainNode = audioContext.createGain();
```

オシレーターからの出力を、エンベロープ用の`GainNode`に接続し、その後、音量調整用の`GainNode`に接続します。

```typescript
oscillator.connect(envelopeGainNode);
envelopeGainNode.connect(gainNode);
gainNode.connect(audioContext.destination);
```

エンベロープのパラメータを設定するために、`envelopeGainNode.gain`の値を時間に応じて変化させます。

```typescript
const now = audioContext.currentTime;

// Attackフェーズ
envelopeGainNode.gain.setValueAtTime(0, now);
envelopeGainNode.gain.linearRampToValueAtTime(1, now + attack);

// Decayフェーズ
envelopeGainNode.gain.linearRampToValueAtTime(sustainLevel, now + attack + decay);

// Sustainフェーズはそのまま維持

// Releaseフェーズ（音を止めるとき）
const stopSound = () => {
  const now = audioContext.currentTime;
  envelopeGainNode.gain.cancelScheduledValues(now);
  envelopeGainNode.gain.setValueAtTime(envelopeGainNode.gain.value, now);
  envelopeGainNode.gain.linearRampToValueAtTime(0, now + release);
  oscillator.stop(now + release);
};
```

#### リバーブエフェクト

**リバーブ**は、音の残響効果を再現するエフェクトです。

##### ConvolverNodeの使用

リバーブを実装するために、`ConvolverNode`を使用します。

`ConvolverNode`とは
- 音声信号に **畳み込み演算（Convolution）** を適用するノードです。
- 入力信号と**インパルスレスポンス**を合成します。

インパルスレスポンスとは
- 特定の空間や機器の音響特性を表現した音声データです。
- 瞬間的な音（パルス音）に対してどのように反応するかを示します。

```typescript
// ConvolverNodeの作成
const convolverNode = audioContext.createConvolver();

// インパルスレスポンスの生成
const createImpulseResponse = (duration, decay) => {
  const sampleRate = audioContext.sampleRate;
  const length = sampleRate * duration;
  // ステレオチャンネル（2チャンネル）のバッファを作成
  const impulse = audioContext.createBuffer(2, length, sampleRate);
  const impulseL = impulse.getChannelData(0);
  const impulseR = impulse.getChannelData(1);

  for (let i = 0; i < length; i++) {
    const n = length - i;
    impulseL[i] = (Math.random() * 2 - 1) * Math.pow(n / length, decay);
    impulseR[i] = (Math.random() * 2 - 1) * Math.pow(n / length, decay);
  }
  return impulse;
};

// インパルスレスポンスをConvolverNodeに設定
convolverNode.buffer = createImpulseResponse(2, 2);
```

- **duration**：インパルスレスポンスの長さ（秒単位）を指定します。長いほど残響が長くなります。
- **decay**：減衰率を指定します。値が大きいほど残響がゆっくり減衰します。
- ステレオチャンネル（左と右）に対して、ランダム性のある減衰効果を設定することで、自然な残響を再現します。
  - `(Math.random() * 2 - 1) * Math.pow(n / length, decay)`について
    - `Math.random() * 2 - 1`で、-1から1の間のランダムな値（ホワイトノイズ）を生成します。
    - `Math.pow(n / length, decay)`で、減衰させます。

##### wetGainNodeとdryGainNodeの説明

リバーブエフェクトを適用する際、元の音声（ドライ信号）とリバーブがかかった音声（ウェット信号）をミックスする必要があります。
- `dryGainNode（ドライゲインノード）`
  - 元の音声信号の音量を調整するためのノードです。
  - リバーブを適用せず、直接出力します。

- `wetGainNode（ウェットゲインノード）`
  - リバーブが適用された音声信号の音量を調整するためのノードです。
  - **ConvolverNode**から出力されたウェット信号を受け取り、音量を調整します。

##### ノードの接続とミキシング

ここまでのノード接続のイメージです。
![リバーブエフェクトのノード構成](/images/reverb_node_structure.drawio.png)

```typescript
// エンベロープノードからの出力をドライとウェットに分岐
envelopeGainNode.connect(dryGainNode); // ドライ信号
envelopeGainNode.connect(convolverNode); // ウェット信号（リバーブ適用）

// ConvolverNodeの出力をウェットゲインノードに接続
convolverNode.connect(wetGainNode);

// ドライとウェットのゲインノードをマスターゲインノードに接続
dryGainNode.connect(gainNode);
wetGainNode.connect(gainNode);

// マスターゲインノードを最終出力に接続
gainNode.connect(audioContext.destination);
```

##### リバーブ量の調整

ユーザーがリバーブの量を調整できるように、ウェットとドライのゲインを変更します。

```typescript
// リバーブ量を調整するスライダーの値を取得
const reverbLevel = 0.5; // 0.0から1.0の範囲で指定

// ゲインノードの音量を設定
wetGainNode.gain.setValueAtTime(reverbLevel, audioContext.currentTime);
dryGainNode.gain.setValueAtTime(1 - reverbLevel, audioContext.currentTime);
```

#### ここまでのサンプル

ADSRエンベロープとリバーブエフェクトを追加したサンプルです。

@[codesandbox](https://codesandbox.io/embed/8s8ptc?view=editor+%2B+preview&module=%2Fsrc%2FReactSimpleSynthesizer4.tsx)

これで、UIライブラリ以外は、以下の完成版デモと同じ状態になりました。  
※UIライブラリ：CodeSandboxのサンプルでは`MUI`を使用し、デモでは`shadcn/ui`を使用しています。
https://react-simple-synthesizer.vercel.app/

## まとめ

最後までお読みいただき、ありがとうございました！  
本記事では、Web Audio APIを使ってシンセサイザーアプリを開発し、ブラウザ上で様々な音声処理を実行できることを学べました。  
Web Audio API には、本記事でご紹介したもの以外にも、様々な音声処理機能が存在しているようですので、興味を持たれた方は、ぜひ MDNのドキュメント等で調べてみてください！

## 参考

https://developer.mozilla.org/ja/docs/Web/API/Web_Audio_API

https://github.com/plumchang/react-simple-synthesizer
