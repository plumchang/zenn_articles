---
title: "Rust(Yew) vs JavaScript(React) — マンデルブロ集合で実測したWebAssemblyのリアルな速度差"
emoji: "🔬"
type: "tech"
topics:
  - "rust"
  - "yew"
  - "react"
  - "webassembly"
  - "performance"
published: false
publication_name: "milabo"
---

# はじめに

こんにちは！株式会社ミラボでエンジニアとして働いている 梅澤 です。

「WebAssembly は速い」「Rust + WASM ならフロントエンドのパフォーマンス問題は解決」——そんな話を一度は聞いたことがあるのではないでしょうか。私もその一人で、実際に手を動かして確かめてみたくなりました。

そこで、フラクタル図形の代表である**マンデルブロ集合**を題材に、**React + JavaScript** と **Yew + Rust(WebAssembly)** で同じ仕様のアプリを作り、Web Worker × 4 の並列計算で公平に性能比較してみました。

**結論を先に言うと、想定通りには行きませんでした。** むしろ、その「想定外」こそが本記事の核になります。

- ❌「Rust + WASM は JS より圧倒的に速い」とは限らない
- ✅ 同じ条件で比較すると **10〜20% 程度の差** に落ち着く
- 🪤 そして、**比較する前にハマった落とし穴が 3 つ**あった

なお、「ハマった落とし穴」は私（Rust 初学者）の経験ベースの話で、一般的な落とし穴を網羅したものではありません。「初心者がやらかすと、こうなる」程度の温度感で読んでいただければ。

実装したサンプルアプリは以下で公開しています。  
※ 公開期限は特に設定していませんが、将来的に公開停止する可能性があります旨はご了承ください。

- React 版
  - デモ: https://plumchang.github.io/react-fractal/
  - ソースコード: https://github.com/plumchang/react-fractal
- Yew 版
  - デモ: https://plumchang.github.io/yew-fractal/
  - ソースコード: https://github.com/plumchang/yew-fractal

## 対象読者

- 「WASM 速い」を鵜呑みにせず、自分の手で確かめたい方
- Rust + Yew でフロントエンド開発に踏み出したい方
- Web Worker による並列計算の実装パターンを知りたい方

# マンデルブロ集合と計測方針

## マンデルブロ集合とは

マンデルブロ集合は、複素数 $c$ に対して以下の漸化式を繰り返したとき、発散しない $c$ の集合です。

$$
z_{n+1} = z_n^2 + c, \quad z_0 = 0
$$

各ピクセル（複素平面上の1点）に対して独立に計算するため、**並列化しやすく、計算量も大きい** という性質があり、性能比較に適した題材です。

![マンデルブロ集合の例](https://upload.wikimedia.org/wikipedia/commons/2/21/Mandel_zoom_00_mandelbrot_set.jpg)

（画像引用元：[Wikipedia マンデルブロ集合](https://ja.wikipedia.org/wiki/マンデルブロ集合)）

## 計測指標

両アプリとも、画面左上のオーバーレイに以下を表示しています。

- **Frame**: 描画リクエスト発行から `putImageData` 完了までの時間 [ms]
- **FPS**: 直近 1 秒間に完了したフレーム数

純粋な計算性能を測るのは **Frame ms** です。FPS はユーザーの操作頻度（ホイールをどれだけ回したか）に依存してしまうため、参考値として扱います。

# アーキテクチャ：両者を揃える

両アプリとも以下の構成にしました。「言語/ランタイムの差」だけが結果に効くようにするためです。

```
[UI スレッド]
   ↓ postMessage(width, startY, endY, zoom, ox, oy, maxIter)
[Worker × 4] 各スレッドで担当チャンクを計算
   ↑ postMessage(startY, chunkData: ArrayBuffer)
[UI スレッド] 4 チャンクが揃ったら ImageData を合成して putImageData
```

ポイント：

1. **Worker は使い回す**：毎フレーム `new Worker()` するのではなく、起動時に 4 つ作ったものをプールとして再利用
2. **連続描画は間引く**：描画中に新しいリクエストが来たら、最後の 1 つだけ `queued` に保留し、現フレームが完了したら次を発行
3. **計算ロジックも同等**：カルディオイド・周期2球の早期リターンも両方に入れる

## React 側の実装の要点

```ts
class FractalPool {
  private workers: Worker[] = [];
  private inFlight = false;
  private queued: DrawParams | null = null;

  constructor() {
    for (let i = 0; i < NUM_WORKERS; i++) {
      const worker = new Worker(FractalWorker, { type: "module" });
      worker.onmessage = (e) => this.onMessage(e.data);
      this.workers.push(worker);
    }
  }

  submit(params: DrawParams) {
    if (this.inFlight) {
      this.queued = params; // 描画中は最新パラメータだけ保留
      return;
    }
    this.inFlight = true;
    this.frameStart = performance.now();
    // 4 チャンクに分割して各 Worker に依頼
    for (let i = 0; i < NUM_WORKERS; i++) {
      this.workers[i].postMessage({ ...params, width, startY, endY });
    }
  }
}
```

Worker 側（`fractalWorker.ts`）はメインからのリクエストを `onmessage` で受け、マンデルブロ計算を回して RGBA バッファを返します。

```ts
// fractalWorker.ts （要点抜粋）
self.onmessage = (event) => {
  const { width, startY, endY, zoom, offsetX, offsetY, maxIter } = event.data;
  const imageData = new Uint8ClampedArray(width * (endY - startY) * 4);

  for (let y = startY; y < endY; y++) {
    for (let x = 0; x < width; x++) {
      const c_re = x / zoom + offsetX;
      const c_im = y / zoom + offsetY;
      const iter = mandelbrot(c_re, c_im, maxIter);
      const idx = ((y - startY) * width + x) * 4;
      // ... iter から RGBA を決定 ...
    }
  }

  // 第2引数の transferable リストに ArrayBuffer を入れると、
  // 構造化クローン（コピー）ではなく所有権の移譲になる → 大きなバッファでも高速に渡せる
  self.postMessage(
    { startY, chunkData: imageData.buffer },
    [imageData.buffer]
  );
};
```

ポイントは `postMessage` の **第 2 引数（transferable リスト）** に `ArrayBuffer` を渡している点です。これによりバッファが zero-copy でメインスレッドに移譲されます（Worker 側からは以後そのバッファにアクセスできなくなる代わりに、コピーが発生しない）。

## Yew 側の実装の要点

Trunk の Worker サポートを使い、メインバイナリと Worker バイナリを分離します。

```html
<!-- index.html -->
<link data-trunk rel="rust" href="Cargo.toml" data-bin="yew-fractal" data-type="main" />
<link data-trunk rel="rust" href="Cargo.toml" data-bin="fractal-worker" data-type="worker" />
```

```toml
# Cargo.toml
[[bin]]
name = "yew-fractal"
path = "src/main.rs"

[[bin]]
name = "fractal-worker"
path = "src/bin/fractal-worker.rs"
```

Trunk は `data-type="worker"` のバイナリを `<bin名>.js` / `<bin名>_bg.wasm` の固定名で出力してくれるので、メイン側からは `importScripts` する Blob を作って `Worker::new()` に渡します。

```rust
fn spawn_worker() -> Worker {
    let origin = window().unwrap().location().origin().unwrap();
    let script = Array::new();
    script.push(
        &r#"importScripts("/fractal-worker.js");wasm_bindgen("/fractal-worker_bg.wasm");"#.into(),
    );
    let bag = BlobPropertyBag::new();
    bag.set_type("text/javascript");
    let blob = Blob::new_with_str_sequence_and_options(&script, &bag).unwrap();
    let url = Url::create_object_url_with_blob(&blob).unwrap();
    Worker::new(&url).unwrap()
}
```

Worker 側（`bin/fractal-worker.rs`）は React 版と同じ流儀で、`DedicatedWorkerGlobalScope` の `onmessage` を立てて、計算結果を transferable で返します。

```rust
// bin/fractal-worker.rs （要点抜粋）
fn main() {
    let scope: DedicatedWorkerGlobalScope = JsValue::from(js_sys::global()).unchecked_into();
    let scope_clone = scope.clone();

    // Closure::wrap で Rust クロージャを JS の関数オブジェクトに変換
    let onmessage = Closure::wrap(Box::new(move |msg: MessageEvent| {
        let data = msg.data();
        // Reflect::get で JS オブジェクトのプロパティを取り出す
        let width = /* ... */;
        let start_y = /* ... */;
        // ... マンデルブロ計算 ...

        // Uint8ClampedArray にコピーして transferable で返す
        let array = Uint8ClampedArray::new_with_length(buf.len() as u32);
        array.copy_from(&buf);
        let buffer = array.buffer();
        let transfer = Array::new();
        transfer.push(&buffer);

        scope_clone
            .post_message_with_transfer(&response.into(), &transfer.into())
            .expect("post_message failed");
    }) as Box<dyn FnMut(MessageEvent)>);

    scope.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
    onmessage.forget();  // クロージャを Drop させない（JS 側に渡し続けるため）
}
```

JS の `self.onmessage = (e) => {...}` 1 行に相当する処理が、Rust ではこのくらいの記述量になります。`Closure::wrap` と `forget()` の組み合わせは、[wasm-bindgen のクロージャ仕様](https://wasm-bindgen.github.io/wasm-bindgen/examples/closures.html)で説明されている定石パターンです。

# 落とし穴①：Yew で「並列化したつもり」になっていた話

ここからが本題です。

私は当初、Yew 版を作ったときに以下のような構造体を `FractalWorker` と命名して使っていました。

```rust
// 旧実装（並列ではない）
pub struct FractalWorker {
    max_iter: u32,
}

impl FractalWorker {
    pub fn calculate_chunk(&self, ...) -> Vec<u8> {
        // 計算ロジック
    }
}
```

そしてメイン側でこう呼んでいました。

```rust
// 4 つのチャンクを「並列に」計算しているつもりだった
let worker = worker::FractalWorker::new(100);
for i in 0..4 {
    let chunk = worker.calculate_chunk(width, start_y, end_y, ...);
    image_data[..].copy_from_slice(&chunk);
}
```

お分かりいただけたでしょうか。**これはただの構造体メソッドの逐次呼び出し**です。Web Worker でも何でもありません。「Worker」という名前に騙されて、シングルスレッドで 4 チャンクを順番に処理していたわけです。

一方の React 版は最初から本物の `new Worker()` を 4 つ立てて、ブラウザの別スレッドで並列計算していました。

> 「Rust 版がなぜか React より遅い……」

その答えは、**Yew 版だけ並列度 1、React 版は並列度 4** だったから。Rust の単体性能の優位を、4 倍の並列度差が打ち消していたのです。

修正後、Yew 版も本物の Web Worker × 4 で並列処理する構成に書き換えました。これで初めて、両者を**公平に比較できる土台**ができました。

**反省**: 当初のサンプルアプリも AI に手伝ってもらって書いたものでしたが、出力されたコードを読み込まずに「動いている = 意図通り」と思い込んでいたのが原因でした。AI を活用するときは「**動くこと**」と「**自分の意図と合っていること**」を別軸で確認する必要があったな、と。本当に並列化されているかは、ブラウザの DevTools の Performance タブで複数のスレッドが見えるか目視するなどの確認が必要でした。

（ちなみに本記事も AI の助力を大いに借りて執筆していますが、こうやって自分の手で確かめながら進めると、確実に学びが残ります。）

# 落とし穴②：未最適化ビルド（debug ビルド）の WASM は JS より遅いこともある

並列化を直して計測してみると、今度は別の問題が出ました。

```
[計測結果（Yew 版・debug ビルド）]
深ズームシーン: Frame 500ms 超
[計測結果（React 版）]
深ズームシーン: Frame 170ms 程度
```

**Yew 版の方が圧倒的に遅い。**

原因は `trunk serve` のデフォルトが **debug ビルド**（=未最適化の開発用ビルド。`cargo build` の `--release` を付けない場合と同じ。Vite などフロント文脈での "dev ビルド" に近い位置付け）だったことです。Rust の debug ビルドは：

- 配列アクセスの境界チェックが有効
- インライン化が無効
- LLVM の最適化パスがほぼ無効
- `wasm-opt` も走らない

これでは V8 の JIT に勝てません。一方 React 側は Vite の dev サーバでも、ブラウザに渡る JS は V8 の JIT 最適化を受けます。**dev 同士の比較は対称ではない**のです。

`Cargo.toml` に最適化プロファイルを追加し、

```toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

`trunk serve --release` で起動し直すと、wasm のサイズも **174KB → 30KB（約 1/6）** に縮み、性能も大きく改善しました。

**教訓**: WASM のベンチマークを取るときは、必ず `--release` ビルドで。未最適化の debug ビルドの数値を持って「WASM 遅い」と判断してはいけない。これは [wasm-bindgen のドキュメント](https://wasm-bindgen.github.io/wasm-bindgen/) などでも繰り返し書かれていますが、実際にやらかすまで重大さに気づきにくい落とし穴です。

# 落とし穴③（おまけ）：dev で動く ≠ prod で動く

これは Rust/WASM 側ではなく React 側のデプロイ時にハマった話です。

GitHub Pages にデプロイした React 版を開いたところ、コンソールに以下のエラーが出て画面が真っ暗になりました。

```
Failed to load module script: The server responded with a non-JavaScript MIME type
of "video/mp2t". Strict MIME type checking is enforced for module scripts per HTML spec.
```

`video/mp2t`（MPEG Transport Stream）……？ なぜ動画の MIME type が？

原因は、**Vite の Worker 静的解析が効いていなかった** ことでした。元の React 実装ではこう書いていました：

```ts
// ❌ 変数経由だと Vite が Worker を検出できないことがある
const FractalWorker = new URL("./fractalWorker.ts", import.meta.url);
// ...
const worker = new Worker(FractalWorker);
```

### Vite が裏でやっていること

Vite は本番ビルド時、ソースを実行せずに AST（抽象構文木）を読んで `new Worker(new URL("./xxx.ts", import.meta.url))` というパターンを探します。見つかると：

1. `xxx.ts` を **別の chunk としてバンドル**（`.ts` → `.js` に変換、依存もバンドル）
2. 成果物の名前を `xxx-XXXXXX.js` で `dist/assets/` に出力
3. 元のコードの `new URL(...)` を **ビルド後のパスに書き換え**

つまりビルド後のコードは内部的にこう変換されます。

```ts
// ビルド後（Vite が自動変換）
new Worker(new URL("/assets/fractalWorker-DgqTrZoM.js", import.meta.url), { type: "module" });
```

### 変数経由だと「実行時にしか分からない」

ところが上のように `URL` インスタンスを変数に格納すると、Vite は `new Worker(変数)` を見ても **「変数が実行時に何を指すか分からない」** ため、Worker としてのバンドル処理を発動できません。

結果として：
- `fractalWorker.ts` は `.js` にバンドルされない
- 本番では `.ts` 拡張子のファイルが直接配信を試みられる
- GitHub Pages は `.ts` 拡張子を **MPEG Transport Stream（`video/mp2t`）として配信**
- ブラウザは「JS モジュールなのに `video/mp2t` が返ってきた」と判定してロードを拒否

ちなみに [Vite 公式ドキュメントの Web Workers セクション](https://vite.dev/guide/features#web-workers) でも、

> you must instantiate the Worker directly inside the new Worker(...) call for the Worker constructor to be detected by Vite.

と「**Worker コンストラクタの引数として直接書く**」ことが明示的に要求されています。私は当初この記述を読み飛ばしており、変数経由でも動いていたので（dev サーバでは on-the-fly トランスパイルされるので問題が表面化しない）、本番デプロイで初めてエラーに遭遇した、という流れでした。

### 修正は `new Worker` の引数を直接書くだけ

```ts
// ✅ 直書きすれば Vite が Worker を検出して .js として bundle してくれる
const worker = new Worker(
  new URL("./fractalWorker.ts", import.meta.url),
  { type: "module" }
);
```

これで Vite は AST 上で「`new Worker` の第 1 引数が `new URL` 式そのもの」というパターンを検出でき、正しく `.js` にバンドルしてくれます。

`npm run dev` では Vite が `.ts` を on-the-fly でトランスパイルしてくれるため、開発時にはこの問題は表面化しません。**本番ビルドで初めて顕在化する**典型的な「dev で動く ≠ prod で動く」のパターンでした。

**教訓**: フロントの本番ビルド結果は必ず実環境で動作確認する。dev サーバの便利機能に頼った書き方が、本番で牙を剥くことがある。

# 実測結果

両者を同条件（Worker × 4 プール再利用、release ビルド、同じ画面サイズ）で揃えた上で、改めて計測した結果がこちらです。

| シーン | React 版 Frame | Yew 版 Frame | 比 |
|---|---|---|---|
| Zoom 1.9x（初期画面） | **12.0 ms** | **10.0 ms** | Yew 1.2x |
| Zoom 約 380x（中ズーム） | **170.4 ms** | **156.6 ms** | Yew 1.09x |
| Zoom 約 72,000x（深ズーム） | **170.3 ms** | **153.3 ms** | Yew 1.11x |

参考までに、初期画面と深ズーム時のスクリーンショットを添えておきます。

| | React 版 | Yew 版 |
|---|---|---|
| 初期画面（Zoom 1.9x） | ![React 初期画面](/images/rust_yew_vs_react_fractal/react-initial.png) | ![Yew 初期画面](/images/rust_yew_vs_react_fractal/yew-initial.png) |
| 深ズーム（Zoom ~72,000x） | ![React 深ズーム](/images/rust_yew_vs_react_fractal/react-deep-zoom.png) | ![Yew 深ズーム](/images/rust_yew_vs_react_fractal/yew-deep-zoom.png) |

画面左上のオーバーレイで実測値を確認できます。両者の差は数値・見た目の両方とも、わずかな違いに留まっています。

**Yew が 10〜20% 速い程度**——「圧倒的な差」とは言えない結果になりました。

正直、計測前は「Rust + WASM なら 5 倍は速いだろう」と思っていたので、この結果はかなり意外でした。

# 小ネタ：同じロジックを書いたつもりが、色合いが違った話

両アプリの動作確認をしていて、深ズーム時の色合いが微妙に違うことに気づきました。

- **React 版**：全体的に緑がかった、落ち着いたグラデーション
- **Yew 版**：紫や黄色が突然混ざる、派手な色合い

| React 版 | Yew 版 |
|---|---|
| ![React 色合い](/images/rust_yew_vs_react_fractal/react-colors.png) | ![Yew 色合い](/images/rust_yew_vs_react_fractal/yew-colors.png) |

計算ロジックは完全に同じはずなのに、なぜ？

原因は **整数オーバーフローの扱い** でした。

色付けのコードは両者でほぼ同じです。

```ts
// React 版（fractalWorker.ts）
imageData[idx + 1] = iter * 5;  // imageData は Uint8ClampedArray
```

```rust
// Yew 版（bin/fractal-worker.rs）
buf[idx + 1] = (iter * 5) as u8;  // buf は Vec<u8>
```

`iter` は最大 100 程度の値なので、`iter * 5` は最大 500。これを 1 バイト（0〜255）に収める必要がありますが、両者の挙動が **正反対** です。

| `iter * 5` の値 | React (`Uint8ClampedArray`) | Yew (`as u8`) |
|---|---|---|
| 250 | 250 | 250 |
| 255 | 255 | 255 |
| 260 | **255**（クランプ） | **4**（260 % 256） |
| 500 | **255**（クランプ） | **244**（500 % 256） |

`Uint8ClampedArray` は範囲外を **頭打ち（クランプ）** にしますが、Rust の `as u8` は **モジュロ（256 で割った余り）** を取ります。

iter が 51 を超えた瞬間、Yew 版では緑チャネルが「255 → 4」へとガクッと落ちる挙動になり、各チャネル（R, G, B）が独立にぐるぐる回ることで予測不能な色が出現します。

これは「**同じロジックを書いたつもりが、整数型の境界処理の違いで結果が変わる**」という、言語間移植で踏みやすい落とし穴の一例です。意図して揃えるなら以下のいずれかが必要：

```rust
// クランプを明示する
buf[idx + 1] = (iter * 5).min(255) as u8;
```

今回のサンプルアプリでは「Yew 版の色合いの方が派手で見栄えが良い」という主観的な好みもあって、あえて修正せずそのままにしています。両方のデモを見比べてみてください。

> **教訓**: 「型が同じ動きをする」と思い込まない。`Uint8ClampedArray`、`as u8`、`u8::saturating_add`、`u8::wrapping_add` はそれぞれ違う挙動。境界条件のテストを意識する。

# なぜ差が小さいのか

「WASM はネイティブに近い速度で動く」——これ自体は嘘ではありません。ではなぜ、JS との差がここまで縮まるのか。考察してみます。

## 1. V8 JIT の優秀さ

今回ホットパスになっているのは、こんな単純な数値ループです。

```ts
while (i < maxIterations) {
  const z_re2 = z_re * z_re;
  const z_im2 = z_im * z_im;
  if (z_re2 + z_im2 > bailout) break;
  z_im = 2 * z_re * z_im + c_im;
  z_re = z_re2 - z_im2 + c_re;
  i++;
}
```

これは V8 の JIT が **最も得意とするコード**です：

- 変数の型が常に `number`（V8 内部では `double` か `smi`）で安定している
- 制御フローが単純で、最適化を阻害する分岐がない
- 関数呼び出しが少なく、ホット関数の機械語化が効きやすい

V8 はこのような関数を **TurboFan** で最適化し、結果としてネイティブに近い機械語を吐きます。WebAssembly も最終的には機械語になるので、両者の差はそれほど大きくならないわけです。

### 補足：JIT 最適化と「deopt（最適化解除）」

V8 の JIT 最適化はあくまで「**実行時に観測した型の前提**」に依存しています。例えば「`z_re` は常に `number` 型」という前提で機械語を吐く。

ところが、もしループの途中で型の前提が崩れると（例：途中で `z_re` に文字列を代入する、配列に違う型の値が混ざる、など）、V8 は「観測した前提が崩れた」と判断して、**最適化済みコードを破棄してインタプリタに戻ります**。これが **deopt（deoptimization、最適化解除）** と呼ばれる現象で、起きるとそのコードのパフォーマンスが大きく落ちます。

今回のマンデルブロループは：

- ループ内の変数が常に `number`（型変化なし）
- 関数呼び出しなし
- 例外もなし
- 動的なオブジェクト操作なし

という「**deopt のリスクが極めて低い理想形**」になっており、最適化が外れることなく走り切れます。逆に言えば、ここから少しでもパターンを外すと、JS の性能は不安定になりやすい——これが後述の「Rust の魅力」の章で触れる **「パフォーマンスの予測可能性」** の文脈に繋がります。

この deopt の仕組みについては [V8 公式ブログの Sparkplug 解説記事](https://v8.dev/blog/sparkplug) や、[V8 公式ブログ](https://v8.dev/blog) の各種 TurboFan 関連記事が詳しいです。

## 2. Worker 境界のオーバーヘッド

ボトルネックは「計算」だけではありません。

- `postMessage` のシリアライズ
- `ArrayBuffer` の transfer（zero-copy ですが境界処理は走る）
- メイン側で `Uint8ClampedArray` を集約 → `ImageData` 生成 → `putImageData`

これらは **両者共通の "JS 世界の処理"** で、Rust では速くなりません。むしろ、wasm-bindgen 経由のデータコピーが追加で入る分、Rust 側がやや不利になる場面すらあります。

## 3. WASM が活きる領域 / 活きない領域

整理すると、WASM の優位が出やすいのは以下のような場面です：

| 優位が出やすい | 優位が出にくい |
|---|---|
| 複雑な制御フロー（コンパイラ実装、画像処理アルゴリズム等） | 単純な数値ループ（V8 が完全に最適化できる） |
| 文字列処理・パース | postMessage で大量データを往復するワークロード |
| メモリレイアウトを細かく制御したい処理 | DOM 操作主体の処理 |
| 既存 C/C++/Rust 資産の流用 | JS と密に相互作用する処理 |

マンデルブロ集合は不運にも「JIT が完全に最適化できる単純な数値ループ」のど真ん中に入ってしまっていたわけです。

……正直に言うと、**題材選びをミスりました**。「計算量が大きくて並列化できる」という条件だけで選んだのですが、まさに V8 の得意分野でもあった、というオチです。「Rust + WASM がド派手に勝つ」絵を期待していた身としては苦笑いものの結果でした。

### では、どんな題材なら WASM が活きそうだったか

仮に WASM の優位を強く示したかったなら、以下のような題材の方が向いていたと思います。

- **画像処理（フィルタ、リサイズ、ピクセル単位の変形）**：制御フローが複雑で分岐が多く、JIT が deopt しがち。`image` クレートなど Rust 側の充実したエコシステムも使える
- **動画コーデック / 音声処理**：FFmpeg や Opus の Rust ポートを WASM 化して持ち込める
- **物理シミュレーション（剛体・流体・粒子）**：状態遷移の複雑さ、メモリレイアウトの最適化が効く
- **パーサ / コンパイラ / 構文解析**：複雑なステートマシン。`tree-sitter` の WASM ビルドが好例
- **暗号処理（ハッシュ、署名検証、PQC など）**：定数時間計算を厳密に保証したい場面で WASM が向く
- **既存 C/C++/Rust 資産の Web 移植**：SQLite (`sql.js`)、OpenCV、PDFium、Blender の Python など、既に C/C++/Rust で書かれた巨大資産を WASM 化することで活用できる

これらは「JIT に苦手なパターンを混ぜる」「JS で書き直すのが現実的でない既存資産を使う」という性質を持っています。

「次回のネタ」として何か書く機会があれば、このあたりから題材を選んでみたいと思います。

# Rust の魅力（私が実装を通して感じた範囲）

ここまでの結果から「速度差が 10〜20% なら Rust を採用するメリットは無いのでは？」と思うかもしれません。私自身も Rust 初学者の立場なので大それたことは言えませんが、**今回の実装を通じて私が感じた Rust の良さ**を、いくつか共有します。

## 1. 型安全性

```rust
let zoom: f64 = 200.0;
zoom = "200"; // ❌ コンパイルエラー
```

JS（や TS の `any`）でありがちな「実行時まで気づかない型エラー」がありません。今回も Rust 側ではコンパイラが型ミスを片っ端から指摘してくれました。

詳しくは [Rust の所有権・借用・型システム（公式 The Rust Programming Language）](https://doc.rust-jp.rs/book-ja/title-page.html) が定番の入門資料です。

## 2. パフォーマンスの予測可能性

V8 JIT は最適化されている間は速いものの、いわゆる「**deopt（最適化が外れる）**」の条件が複雑だ、という話を「なぜ差が小さいのか」の章で触れました。

一方 WASM は JIT に依存せず最初から機械語なので、「**最悪ケースが安定して速い**」と言われています。私自身まだここを定量的に検証できているわけではないので、詳しくは [WebAssembly 公式の High-level Goals](https://webassembly.org/docs/high-level-goals/) や [V8 公式ブログ - Sparkplug](https://v8.dev/blog/sparkplug) などを参照してください。

## 3. GC stop-the-world がない

JS の GC は近年かなり優秀ですが、それでも 60fps を狙うアニメーションでは数 ms の停止が問題になることがあります。Rust は **所有権モデル** で GC を持たないため、この種のジッタが原理的に起きません。

参考：[Rust の所有権（公式ドキュメント）](https://doc.rust-jp.rs/book-ja/ch04-00-understanding-ownership.html)

## 4. 既存 Rust 資産の Web 展開

これは私も今回の調査で改めて気づいた点です。画像処理（[`image`](https://crates.io/crates/image)）、暗号（[`ring`](https://crates.io/crates/ring)）、パーサ（[`tree-sitter`](https://tree-sitter.github.io/tree-sitter/)）、シミュレーション系などのクレートを、**JS で書き直さずに** Web に持ち込める可能性があるというのは、業務の選択肢を広げる意味でも魅力的に感じました。

（このあたりは私もまだ実践できていない領域なので、語気を強めずに「魅力に見える」と書くにとどめます。）

# まとめ

本記事では、マンデルブロ集合の描画を題材に、React + JavaScript と Yew + Rust(WASM) を **Web Worker × 4 で並列化した上で** 比較してみました。

私（Rust 初学者）が今回の実装と検証を通じて得た学びは以下の通りです。

1. **AI が出したコードを「動く = 意図通り」と思い込まない**：本物の Web Worker で並列化されているか、DevTools で目視確認する
2. **WASM のベンチマークは必ず release ビルドで**：dev ビルドは JS より遅いことすらある
3. **dev で動く ≠ prod で動く**：本番ビルド結果を必ず実環境で動作確認する
4. **同条件で比較すると、差は 10〜20% 程度に落ち着いた**：JIT が得意とする単純な数値ループでは、WASM の優位は限定的だった
5. **Rust の魅力は速度以外にもある**：型安全性、パフォーマンスの予測可能性、GC stop-the-world の不在、既存資産の流用——記事を書く中で改めて感じた点です

「WASM は速い」というメッセージ自体は事実ですが、**自分のユースケースのボトルネックがそこに当たっているか** は別問題なんだな、ということを今回身をもって学べました。本当に WASM が劇的な効果を発揮するのは、JIT が苦戦する複雑な処理や、既存 Rust 資産を活用したい場面でしょう。

もし「WASM 化を検討している」という方がいたら、ぜひ採用判断の前に、**自分のホットパスで実測してみる**ことをお勧めします。私のように「想定と違った」ことに気づけるかもしれません。

最後になりますが、本記事は私自身が Rust + WebAssembly + Yew を学びながら書いたもので、誤りや表現の不適切な点があるかもしれません。お気づきの点があればコメントなどで教えていただけると嬉しいです。

# サンプルコード・参考資料

## 本記事のサンプル

- React 版
  - デモ: https://plumchang.github.io/react-fractal/
  - ソースコード: https://github.com/plumchang/react-fractal
- Yew 版
  - デモ: https://plumchang.github.io/yew-fractal/
  - ソースコード: https://github.com/plumchang/yew-fractal

## Rust + WebAssembly

- [The Rust Programming Language（日本語訳）](https://doc.rust-jp.rs/book-ja/title-page.html) — Rust 公式入門書
- [Rust and WebAssembly](https://rustwasm.github.io/docs/book/) — Rust + WASM の公式ガイド
- [wasm-bindgen Book](https://wasm-bindgen.github.io/wasm-bindgen/) — JS と Rust を繋ぐ仕組み
- [js-sys API ドキュメント](https://wasm-bindgen.github.io/wasm-bindgen/api/js_sys/index.html) — JS 標準オブジェクトの Rust バインディング
- [web-sys ガイド](https://wasm-bindgen.github.io/wasm-bindgen/web-sys/index.html) — ブラウザ Web API の Rust バインディング

## Yew / Trunk

- [Yew 公式](https://yew.rs/) / [Yew チュートリアル](https://yew.rs/docs/getting-started/introduction)
- [Trunk 公式リポジトリ](https://github.com/trunk-rs/trunk) / [Trunk Web Worker サンプル](https://github.com/trunk-rs/trunk/tree/main/examples/webworker)

## Vite / Web Worker

- [Vite - Web Workers](https://vite.dev/guide/features#web-workers)
- [MDN - Web Workers API](https://developer.mozilla.org/ja/docs/Web/API/Web_Workers_API)

## V8 / WebAssembly の仕組み

- [V8 公式ブログ](https://v8.dev/blog) — TurboFan、Sparkplug、Ignition の解説記事多数
- [WebAssembly High-Level Goals](https://webassembly.org/docs/high-level-goals/)

## マンデルブロ集合

- [Wikipedia - マンデルブロ集合](https://ja.wikipedia.org/wiki/マンデルブロ集合)
- [Mandelbrot set - Wikipedia (English)](https://en.wikipedia.org/wiki/Mandelbrot_set) — メインカルディオイド・周期2の球の数式の出典
