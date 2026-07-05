# tinkerbox 設計書 (v1)

- Repository: `github.com/Saber5656/tinkerbox`
- Tagline: "A playful physics sandbox for water, sand, gears, and tinkering."
- License: MIT（前提）/ クラウド送信なし / 個人 OSS・最小実装で早期リリース
- 作成日: 2026-07-05

---

## 1. コンセプトと既存 OSS との位置づけ

tinkerbox は、ブラウザで開いた瞬間に砂・水・火を指で描いて遊べる falling sand（粉体）サンドボックスである。
URL を開いて 3 秒で遊べる・スマホのタッチで遊べる・コードを読めば午後のうちに自分の元素を足せる、
の 3 点だけに集中し、完全クライアントサイドの静的 Web アプリとして GitHub Pages で配布する。

falling sand というジャンル自体は 2000 年代からある古典で、名作 OSS がすでに存在する。
tinkerbox の存在理由は機能量ではなく「小ささと読みやすさ」に置く。

**既存 OSS との位置づけ（車輪の再発明にならない理由）**

| 既存 | 実装 / 形態 | 強み | tinkerbox との違い |
|---|---|---|---|
| sandspiel (MaxBittker) | Rust + WASM + WebGL、Web | 約 20 元素、流体・風圧、投稿ギャラリー。300×300 を 60fps で駆動 | tinkerbox は **純 TypeScript のみ**。Rust ツールチェーン不要で fork → 改造の敷居が一桁低い。ギャラリー用バックエンド（Firebase）も持たず**ネットワーク通信ゼロ** |
| The Powder Toy | C++ + SDL、デスクトップアプリ | 気圧・熱・電子回路まで再現する本格派。歴史と深さは別格 | tinkerbox はインストール不要の Web。**スマホで動く**。深さでは戦わない |
| Coding Train #180 や小規模 JS クローン群 | p5.js / 素の JS、チュートリアル | 学習用として優秀。実装例が豊富 | 多くは砂 1 種のデモ止まり。tinkerbox は複数元素の相互作用・タッチ UI・配布までを**プロダクトとして仕上げる** |

つまり「sandspiel の面白さの核を、oneko.js 的な"読める小ささ"で再実装し、PR で元素を足せる形にして出す」ことが存在理由。
sandspiel が Rust の講演資料になったように、tinkerbox は **TypeScript + Canvas 2D の教材にもなる遊び場**を狙う。

**リポジトリ説明の "gears" について**: タグラインの歯車は v2 以降の方向性として残す。
歯車は剛体回転 + 動力伝達であり、セルオートマトンとは別ジャンルの物理になるため v1 には入れない（§2 で理由を明記）。

---

## 2. v1 スコープ

| 区分 | 項目 | 備考 |
|---|---|---|
| 入れる | 元素 7 種（砂 / 水 / 壁 / 火 / 植物 / 油 + 副産物の煙） | §5 の相互作用表。選択可能は 6 種 + 消しゴム |
| 入れる | ブラシ描画（サイズ 3 段階）・消しゴム・クリア・一時停止 | 道具はこれだけ。タッチ / マウス / ペン共通（Pointer Events） |
| 入れる | プラガブルな元素レジストリ | 「1 ファイル追加 + 登録 1 行」で元素を足せる構造。v2 拡張と外部 PR の受け皿 |
| 入れる | 初期デモシーン + 操作ヒント 1 行 | 開いた瞬間に動いている状態を見せる（§6） |
| 入れる | 決定的シミュレーション（シード付き PRNG） | 画素スナップショットで回帰テスト可能にする |
| 入れる | GitHub Pages 自動デプロイ + 英語 README | 1 クリック URL 配布 |
| 入れない | **歯車（gears）・剛体物理** | タグライン由来だが、剛体回転 + トルク伝達は CA と別ジャンルの規模（The Powder Toy ですら歯車は持たない）。v1 の「早く出す」に反する。エンジンは CA 層と描画層を分離し、v2 で別レイヤーとして足せる構造だけ担保する |
| 入れない | WASM（Rust / AssemblyScript）化 | sandspiel 級の 300×300+ 流体を狙わない限り不要（§4 調査）。性能不足が実測されたときの v2 手段として温存 |
| 入れない | WebGL 描画・GPU シミュレーション | putImageData 1 回/フレームで足りる見込み（§5）。依存と難読化はコンセプトに反する |
| 入れない | Web Worker / OffscreenCanvas | グリッド予算内なら main thread で 60fps 可能な想定。スパイク（#1）で崩れた場合のみ再検討 |
| 入れない | 温度・気圧・電気などの物理拡張 | The Powder Toy の領分。元素間の局所ルールだけで面白さを出す |
| 入れない | セーブ / ロード・共有 URL・GIF 書き出し | 砂遊びは揮発性でよい。v2 候補 |
| 入れない | PWA / オフラインキャッシュ | 依存ゼロの静的 1 ページは素で軽い。SW の更新不全リスクを v1 で背負わない |
| 入れない | アンドゥ・音・i18n | UI 文言は 10 語未満のアイコン主体。翻訳対象がほぼない |

---

## 3. 対応プラットフォームと優先順位

| 優先度 | 対象 | 判断 | 理由 |
|---|---|---|---|
| 1 (v1) | モバイルブラウザ（iOS Safari 16+ / Android Chrome） | 第一級対応 | 「開いて 3 秒で遊べる」の主戦場はスマホ。The Powder Toy が届かない層に届く差別化点。タッチ前提で UI 設計する |
| 1 (v1) | デスクトップブラウザ（Chrome / Edge / Firefox / Safari 最新 2 メジャー） | 対応 | 開発・デモ環境。大画面ではセル数が増えるため性能上はこちらが厳しい（§5） |
| 対象外 | ネイティブアプリ（iOS / Android / デスクトップ） | 見送り | 署名・ストア・配布の維持コストが個人 OSS に見合わない。Web で全プラットフォームに届く |
| 対象外 | IE・旧 WebView | 非対応表明 | Pointer Events / ES2020 前提 |

必要 API は Canvas 2D / `ImageData` / Pointer Events / `requestAnimationFrame` のみ。いずれも上記ブラウザで安定しており、権限ダイアログも一切出ない。

---

## 4. 技術選定

### シミュレーションコアの比較

| 候補 | 性能 | 実装・保守コスト | 読みやすさ（存在理由への適合） | 判定 |
|---|---|---|---|---|
| **純 TypeScript + typed array（採用）** | 60fps で〜12 万セル級は現実的（下記調査） | 最小。ツールチェーンは Vite のみ | ◎ ブラウザで読めて fork できる | ✅ |
| Rust + WASM（sandspiel 方式） | ◎ 300×300 + 流体を 60fps で実証済み | Rust + wasm-bindgen の 2 層。コントリビュート敷居が激増 | △ 本家 sandspiel が既にある | ❌ v1 では過剰。性能不足時の v2 手段 |
| GPU シミュレーション（WebGL fragment shader CA） | ◎ | シェーダデバッグが難。元素追加のたびにシェーダ改修 | × 「1 ファイルで元素追加」と両立しない | ❌ |

### 描画方式の比較

| 候補 | 性能 | 判定 |
|---|---|---|
| **`ImageData` に直接書き込み → `putImageData` 1 回/フレーム、CSS で整数倍拡大（採用）** | セル=1px で書けば転送は 1 回。sandspiel も「描画は 16ms 予算中 1ms 未満」を CPU→GPU テクスチャ転送で達成しており、同じ発想の Canvas 2D 版 | ✅ |
| セル毎に `fillRect` | 数万セルで破綻する定番アンチパターン | ❌ |
| WebGL テクスチャ描画 | 最速だが依存と複雑さが増す | ❌ v1 では不要 |

### 性能調査の結果（グリッドサイズ決定の根拠）

- sandspiel は **300×300（9 万セル）を Rust+WASM+WebGL で 60fps** 駆動。ただし風圧・流体場など tinkerbox が持たない重い計算を含む。作者の 2015 年の**素の JS 試作は 100×100** だった（最適化以前の実装）
- 素の JS + `Uint8Array` の実装例（perymimon 氏の解説）では **400×400（16 万セル）** をダブルバッファ + typed array で駆動しており、単純な局所ルールなら純 JS で 10 万セル級が現実的であることを示している
- tinkerbox の 1 セル更新は「switch + 近傍数セルの読み書き」で、12 万セル × 60fps ≒ 720 万セル更新/秒。現代の JS エンジンで数 ms に収まる見込み
- **結論: 論理セルの表示サイズ 4 CSS px を基準にグリッドを動的決定し、総セル数の上限を約 12 万セルとする**（例: 1280×800 の描画領域 → 320×200 = 6.4 万セル。4K 全画面では上限を超えるためセル表示サイズを 5〜6px に上げて吸収）。スマホは画面が小さくセル数は 2〜4 万に収まるため、実は大画面デスクトップの方が性能上厳しい。この予算の妥当性は **Issue #1 のスパイクで実測確認する（P0）**

### 採用スタック

| 層 | 技術 | 理由 |
|---|---|---|
| 言語 | TypeScript (strict) | 元素定義・エンジン API の契約を型で固定。PR を受けやすい |
| ビルド | Vite | 静的サイト出力が軽量・高速。GitHub Pages と相性がよい（8bitme と同系統） |
| シミュコア | 依存ゼロの純 TS モジュール（`core/`） | `Uint8Array` グリッド + 古典 CA アルゴリズム。DOM 非依存で単体テスト可能 |
| 描画 | Canvas 2D + `ImageData`（`Uint32Array` ビューでパレット書き込み） | 上記比較のとおり。`image-rendering: pixelated`（非対応時は `imageSmoothingEnabled = false` の drawImage）で拡大 |
| UI 殻 | Preact + 素の CSS | ツールバー程度の UI に React 互換 API を ~4KB で。状態管理ライブラリ不要（8bitme と同系統） |
| 入力 | Pointer Events + `touch-action: none` | マウス / タッチ / ペンを 1 経路で処理。スクロール・ピンチ誤爆を根元で止める |
| 乱数 | シード付き xorshift32（自前 10 行） | `Math.random` 禁止。決定的シミュレーションで回帰テストを成立させる |
| テスト | Vitest + グリッドスナップショット比較 | コアが純関数 + 整数演算なのでプラットフォーム差なく画素比較できる |
| CI/CD | GitHub Actions → GitHub Pages | push で build + deploy。手作業ゼロ |
| ランタイム依存 | Preact のみ（コアはゼロ） | 「読める小ささ」を数字で担保する |

---

## 5. アーキテクチャ

### 全体構成

```
┌────────────────────────────────────────────────────────┐
│ app/ (Preact UI 殻)                                     │
│  ┌───────────┐   選択元素/ブラシ径   ┌────────────────┐  │
│  │ Toolbar    │────────────────▶│ InputController │  │
│  │ (元素/道具) │                  │ (Pointer Events │  │
│  └───────────┘                  │  → 線分をセル座標 │  │
│        │ pause/clear            │  へ量子化・塗布)   │  │
│        ▼                        └───────┬────────┘  │
│  ┌─────────────────────────────────────▼─────────┐  │
│  │ GameLoop (requestAnimationFrame)                │  │
│  │   1. input を grid へ反映                        │  │
│  │   2. engine.step()                              │  │
│  │   3. renderer.blit()                            │  │
│  └───────┬──────────────────────────┬─────────────┘  │
├──────────┼──────────────────────────┼────────────────┤
│ core/ (依存ゼロ・DOM 非依存の純 TS)     │                │
│  ┌───────▼────────┐        ┌───────▼─────────────┐  │
│  │ Engine          │        │ Renderer             │  │
│  │  Uint8Array×2   │ 読取専用 │  palette: Uint32[]   │  │
│  │  (type + aux)   │───────▶│  ImageData へ書込 →   │  │
│  │  bottom-up 走査  │        │  putImageData 1回     │  │
│  └───────┬────────┘        └─────────────────────┘  │
│          │ update(cell, api)                          │
│  ┌───────▼────────────────────────────┐              │
│  │ ElementRegistry                     │              │
│  │  elements/sand.ts  water.ts  fire.ts │ ← 1元素=1ファイル │
│  │  wall.ts plant.ts oil.ts smoke.ts   │   登録1行で追加   │
│  └────────────────────────────────────┘              │
└────────────────────────────────────────────────────────┘
```

### データ構造と走査（CA の定石を採用）

- グリッドは `Uint8Array` を 2 本: `type[i]`（元素 ID）と `aux[i]`（寿命・色ゆらぎシード・**更新済みフラグ 1 bit**）
- 走査は**下の行から上へ**（落下物が 1 フレームに 1 セルずつ自然に落ちる）。**行内の走査方向は毎フレーム乱数で左右反転**し、斜め移動の左右バイアスを消す（falling sand の定石）
- 上昇する元素（煙・火）は bottom-up 走査だと同一フレームで多重移動するため、`aux` の**フレームパリティビット**で「このフレーム更新済み」を弾く
- 元素の振る舞いは sandspiel と同じ「element = 近傍を読み書きする 1 関数」パターン。`update(cell, api)` に渡す `api` は `get(dx,dy)` / `set(dx,dy, cell)` / `rand()` だけの最小 API とし、境界チェックは api 側で吸収する

```ts
// 元素定義の契約（実装イメージ。1 ファイル = 1 元素）
interface Element {
  id: number;            // 1..255
  name: string;          // "Sand"
  color: number;         // 基本色 (ABGR)。描画時に aux シードで明度±6% ゆらぎ
  selectable: boolean;   // ツールバーに出すか（Smoke は false）
  update(api: CellApi): void;
}
```

### v1 元素と相互作用

| 元素 | 分類 | 挙動 |
|---|---|---|
| Sand | 粉体 | 下 → 空きがなければ斜め下（左右ランダム）。水・油より重く沈む |
| Water | 液体 | 落下 + 左右へ水平拡散（dispersion 3〜5 セル/フレーム）。火を消す |
| Wall | 固体 | 不動・不燃。仕切りや器を作る道具 |
| Fire | 反応 | 寿命 30〜90 tick。隣接する Plant / Oil に引火。ゆらぎながら上昇し Smoke を残す。Water に接すると消える |
| Plant | 固体 | 不動。隣接する Water を確率で取り込んで成長（sandspiel 型の名物挙動）。可燃 |
| Oil | 液体 | Water より軽い液体（浮く）。強可燃で火が走る |
| Smoke | 気体 | 上昇し短寿命で消滅。Fire の副産物（選択不可） |

液体の比重（Sand > Water > Oil）は「下のセルがより軽い液体なら入れ替わる」だけの単純規則で表現する。

### フレーム処理（1 tick）

1. 入力キューを反映（前フレームからのポインタ軌跡を線分補間しブラシ径で塗る。タップ 1 点でも塗れる）
2. `engine.step()`: bottom-up 走査で全セルの `update()` を実行（更新済みビットで多重更新防止）
3. `renderer.blit()`: type + aux → パレット + 明度ゆらぎで `ImageData` に書き込み、`putImageData` 1 回
4. `document.hidden` 時はループごと停止（バッテリー配慮）。一時停止中は 2 をスキップし描画のみ

### 重要な設計判断

- **コア（core/）と殻（app/）の完全分離**: core は DOM も Preact も知らない。テスト容易性に加え、v2 で歯車レイヤーや WASM 差し替えを検討する際の切り離し面になる
- **決定性**: シード固定なら同じ操作列は同じ画になる（整数演算のみ）。「初期シーン + N tick 後のグリッド」をスナップショットテストにする
- **リサイズ / 画面回転はグリッド再生成（= 盤面クリア）で割り切る**: 砂遊びは揮発性。状態移植のコードを v1 に持ち込まない

---

## 6. UI/UX

### 画面構成（1 画面完結・タッチ最優先）

```
┌─────────────────────────────┐
│                             │
│        <canvas>             │← 全面がキャンバス。指 / マウスで描く
│     （初期デモシーン再生中）      │
│                             │
│  "Draw with your finger ✏"  │← 初回のみ。最初のストロークで消える
├─────────────────────────────┤
│ [🟨Sand][💧Water][🧱Wall]     │← 元素ボタン（色スウォッチ + 選択リング）
│ [🔥Fire][🌱Plant][🛢Oil][◻消] │   横スクロール可。ElementRegistry から自動生成
│  ブラシ ●/●●/●●●  ⏸  🗑  ℹ   │← ブラシ径 3 段階 / 一時停止 / クリア / About
└─────────────────────────────┘
```

- ツールバーは**画面下部固定**（スマホの親指到達圏）。デスクトップも同一レイアウトで分岐を作らない
- 元素ボタンはレジストリの `selectable` から自動生成。**元素を足すと UI にも勝手に並ぶ** = コントリビュート体験の一部
- クリアは誤爆防止に長押し or 確認なしの undo なし……ではなく「クリア直後 5 秒だけ Restore ボタン表示」も検討したが、**v1 は確認なし即クリア**で割り切る（砂遊びに惜しむ状態はない）
- 設定画面・保存機能なし。`localStorage` は選択元素とブラシ径の記憶のみ（画像・個人情報は保存しない）

### 初回起動体験（勝負は 3 秒）

1. URL を開く → **すでに動いているデモシーン**（砂山 + 注がれる水 + 植物、数秒後に火花が油に引火）が迎える。「何ができるか」を操作ゼロで伝える
2. ヒント 1 行「Draw with your finger」（デスクトップは "Click and drag"）。最初のストロークでフェードアウトし、以後表示しない
3. 既定選択は Sand（いちばん気持ちいい元素）。2 タップ目で Water にすれば相互作用まで 10 秒で体験できる

### エッジケースの割り切り（v1）

- 画面回転・ウィンドウリサイズで盤面クリア（§5）。頻度と実害が低い
- ピンチズーム・ダブルタップズームは `touch-action: none` + viewport meta で抑止（キャンバス上のみ）
- マルチタッチは第 1 ポインタのみ有効（2 本指同時描画は v2 候補）
- 端末がスリープ → 復帰時はそのまま再開（状態はメモリ上にある）。タブ非表示中はループ停止

---

## 7. プライバシー設計

- **外部通信ゼロを構造で保証する**: バックエンドなし・API なし・アナリティクスなし・外部フォント / CDN なし。実行時に発生するネットワークリクエストは GitHub Pages からの静的アセット取得のみ
- CSP を meta タグで宣言（`default-src 'self'`）し、外部への fetch をブラウザレベルでブロック
- `localStorage` の用途は UI 設定 2 キーのみ。描いた絵・画像・識別子は保存も送信もしない
- README に検証手順を明記: 「DevTools → Network タブを開いて遊ぶ → ロード後のリクエストが 0 件であることを確認」

---

## 8. 配布方法

| 項目 | v1 の方針 | 理由 |
|---|---|---|
| 一次配布 | GitHub Pages（`https://saber5656.github.io/tinkerbox/`） | 1 クリック URL。インストール不要が最大の売り |
| デプロイ | GitHub Actions: main へ push → Vite build → Pages deploy | 手作業ゼロ。fork した人も Actions 有効化だけで自分の Pages に出せる |
| バージョニング | git タグ + GitHub Releases（CHANGELOG のみ、バイナリなし） | 静的アプリのため「更新 = 再訪で最新」 |
| 自動更新 | **不要**（静的 Web の性質上、リロードが最新版） | SW キャッシュを持たない（§2）ので更新不全も起きない |
| ライセンス | MIT。`LICENSE` を初回リリース前に追加 | 前提どおり |
| ストア / パッケージ | なし（npm 公開も v1 ではしない） | core/ の npm 切り出しは需要が見えたら v2 で判断 |

---

## 9. README 構成案（英語）

```markdown
<バナー画像: ドット絵の砂時計から砂がこぼれて "tinkerbox" のロゴを形作る横長 PNG>

# tinkerbox 🧪
> A playful physics sandbox for water, sand, and tinkering — right in your browser.

<デモ GIF: スマホ画面で砂→水→油→火と描いて延焼するまでの 10 秒ループ>

**👉 Play now: https://saber5656.github.io/tinkerbox/** — no install, no sign-up,
and nothing ever leaves your browser.

[license badge] [pages deploy badge] [release badge]   ← バッジは 3 個まで

## What it is
- A classic falling-sand toy: paint sand, water, oil, fire, and plants, then watch them interact
- Works on your phone — draw with your finger
- ~zero dependencies, readable TypeScript, 60fps on a plain <canvas>

## Elements
| Sand | Water | Wall | Fire | Plant | Oil | の 1 行紹介表

## Add your own element
An element is one small TypeScript file:（update(api) の 6 行スケッチ）
Register it in one line and it appears in the toolbar. PRs welcome — see CONTRIBUTING.md.

## Why another falling sand game?
sandspiel is gorgeous (Rust + WASM). The Powder Toy is deep (C++ desktop).
tinkerbox is the small, readable TypeScript one — built to be forked, hacked, and learned from.

## Privacy
100% client-side. No servers, no analytics, no requests after load
(open DevTools → Network and see for yourself).

## Development
git clone / npm i / npm run dev の 3 行

## Roadmap
Gears and contraptions (the "tinkering" in the name), save/share, more elements — see issues.

## License
MIT
```

ポイント: デモ GIF と Play now の 1 行導線をファーストビューに収める。
"gears" はタグラインとの整合のため Roadmap で言及する。

---

## 10. リスクと実装前検証項目

| 優先度 | 項目 | 内容 | 検証方法 |
|---|---|---|---|
| **P0** | **セル予算 12 万で 60fps が本当に出るか** | 純 TS + putImageData で、砂 + 水が大量に動く最悪ケースの実測。デスクトップ 4K（セル上限時）と 3〜4 年落ちのミドルレンジ Android / iPhone で確認。崩れた場合はセル表示サイズ引き上げ（グリッド縮小）→ dirty-rect → WASM の順にフォールバック方針を決める | **Issue #1 のスパイク**。UI なしのベンチページで fps と step()/blit() の ms を計測し結果を Issue に記録 |
| **P0** | 液体（水）の見栄え | 素朴な CA の水は「階段状に固まる」失敗が定番。dispersion（横方向の走査距離）のチューニングで自然に流れるかをスパイクで同時確認 | 同上（スパイクに水を含める理由） |
| P1 | iOS Safari のジェスチャ干渉 | `touch-action: none` でもダブルタップズーム・画面端スワイプが割り込む既知の癖。実機で描画が途切れないか確認 | #6 実装時に iPhone 実機テスト |
| P1 | 決定性の担保 | `Math.random` 混入や走査順の非決定性があるとスナップショットテストが崩壊する | #3 で lint ルール（core/ での Math.random 禁止）+ シード固定テスト |
| P2 | 連続 60fps 駆動の発熱・電池 | 放置タブの電力消費。`document.hidden` で停止 + 静止検知での間引きは v1.x | #7 で visibilitychange 停止のみ実装 |
| P2 | 非整数倍拡大のにじみ | devicePixelRatio が半端な端末で pixelated でもモアレが出る場合がある | スパイクの実機確認に含める |
| P3 | 名称衝突 | "tinkerbox" の既存プロダクト・npm パッケージの簡易確認 | リリース前に検索 |

**最重要リスクは P0 の 2 件**。グリッド予算（＝遊べる広さ）と水の手触りはプロダクトの核であり、
ここが崩れると技術選定（§4）の前提が変わるため、**本実装前に捨てられるスパイク（Issue #1）で必ず実測する**。

---

## 11. v1 Issue 分割案（8 個）

- **#1 `Spike: benchmark falling-sand performance on plain Canvas 2D`** — ラベル: `spike`, `design`
  UI なしのベンチページを作り、Uint8Array グリッド + bottom-up 走査 + putImageData の最小実装（砂 + 水のみ）で P0 リスクを実測する。セル数 6 万 / 12 万 / 20 万での fps と step()/blit() の所要 ms を、デスクトップとミドルレンジのスマホ実機で計測し、既定セル予算と水の dispersion 値を決定する。
  受け入れ条件: 計測結果表と決定値（セル予算・セル表示 px・dispersion）が Issue コメントに記録され、フォールバック方針（縮小 → dirty-rect → WASM）の要否が判断されている。コードは捨ててよい。

- **#2 `Set up Vite + TypeScript + Preact scaffold with Pages deploy`** — ラベル: `infra`
  Vite + TS strict + Preact の雛形、GitHub Actions による main push → GitHub Pages 自動デプロイ、CSP meta（`default-src 'self'`）、LICENSE (MIT) を整備する。以後の Issue は常にデプロイ可能な状態で進める。
  受け入れ条件: `https://saber5656.github.io/tinkerbox/` にプレースホルダページが公開され、push で自動更新される。外部リクエストが 0 件である。

- **#3 `Implement core engine: grid, element registry, deterministic step loop`** — ラベル: `enhancement`
  `core/` に Uint8Array×2 のグリッド、ElementRegistry、bottom-up + 行方向ランダム反転の走査、更新済みビット、シード付き xorshift32、CellApi（get/set/rand）を DOM 非依存で実装する。#1 の決定値を反映する。
  受け入れ条件: 砂 1 元素がヘッドレスで動き、シード固定の N tick スナップショットテストが Vitest で通る。core/ に DOM 参照と Math.random がない（lint で強制）。

- **#4 `Implement ImageData renderer with palette and per-cell shading`** — ラベル: `enhancement`
  パレット（Uint32）+ aux シードによる明度ゆらぎで ImageData に書き込み、putImageData 1 回/フレームで描画する。`image-rendering: pixelated`（フォールバック: imageSmoothingEnabled=false）による整数倍拡大、リサイズ時のグリッド再生成を実装する。
  受け入れ条件: 砂が粒感のある見た目で 60fps 描画され、blit() が 2ms 以内（#1 計測環境比）。ウィンドウリサイズで破綻しない。

- **#5 `Implement the v1 element set and interactions`** — ラベル: `enhancement`
  Sand / Water / Wall / Fire / Plant / Oil / Smoke を 1 元素 1 ファイルで実装する。比重による液体の入れ替わり、引火・消火・植物の成長・煙の寿命など §5 の相互作用表を満たす。
  受け入れ条件: 相互作用表の全項目が目視確認でき、代表シナリオ（水没する砂・浮く油・延焼→鎮火）のスナップショットテストが通る。

- **#6 `Build touch-first toolbar UI and pointer input`** — ラベル: `enhancement`, `ux`
  Pointer Events + `touch-action: none` によるストローク描画（線分補間・ブラシ径 3 段階・消しゴム）、レジストリから自動生成される元素ボタン、一時停止・クリア・About を Preact で実装する。localStorage に選択状態を記憶する。
  受け入れ条件: iOS Safari / Android Chrome 実機でスクロール・ズーム誤爆なく描け、元素を 1 ファイル追加するとツールバーに自動で並ぶ。

- **#7 `Add first-run demo scene and lifecycle polish`** — ラベル: `enhancement`, `ux`
  初期デモシーン（砂山 + 水 + 植物 + 引火する油）、初回ヒント 1 行とフェードアウト、`document.hidden` でのループ停止、タップ 1 点でも塗れる入力の仕上げを実装する。
  受け入れ条件: 初回アクセスから操作ゼロで「動いている盤面」が見え、ヒントが最初のストロークで消える。バックグラウンドタブで CPU 消費が止まる。

- **#8 `Write English README with banner, demo GIF, and contributor guide`** — ラベル: `docs`
  §9 の構成で README を作成し、バナー画像とデモ GIF（スマホ操作 10 秒ループ）を撮影・作成する。「元素を 1 ファイルで足す」手順の CONTRIBUTING.md を添え、プライバシー検証手順と gears の Roadmap 言及を含める。
  受け入れ条件: GIF と Play now リンクがファーストビューに収まり、バッジが 3 個以内で、CONTRIBUTING の手順どおりに新元素を追加できる。

推奨着手順: #1 → #2 → #3 → #4 → (#5, #6 並行) → #7 → #8。

---

## 参考資料（設計時の調査ソース）

- sandspiel の設計と性能（Rust+WASM+WebGL、300×300、描画 <1ms、2015 年 JS 試作は 100×100）: [Making Sandspiel — Max Bittker](https://maxbittker.com/making-sandspiel/) / [GitHub: MaxBittker/sandspiel](https://github.com/MaxBittker/sandspiel) / [JSConf EU 2019 講演ページ](https://2019.jsconf.eu/max-bittker/simulating-sand-building-interactivity-with-webassembly.html)
- 純 JS + Canvas の falling sand 実装と最適化（Uint8Array・400×400・ダブルバッファ・分岐削減）: [Building a Falling Sand Simulation with HTML Canvas — perymimon (Medium)](https://medium.com/@perymimon/building-a-falling-sand-simulation-with-html-canvas-2a73c1339c5e)
- The Powder Toy（C++ / SDL のデスクトップ本格派、GPL）: [GitHub: The-Powder-Toy/The-Powder-Toy](https://github.com/The-Powder-Toy/The-Powder-Toy)
- 入門実装の定番（p5.js での砂 1 種デモ）: [The Coding Train — Challenge #180: Falling Sand](https://thecodingtrain.com/challenges/180-falling-sand/) / 実装スレッド: [Making a falling sand simulator — Hacker News](https://news.ycombinator.com/item?id=31309616)
- JS クローンの実例: [GitHub: JuulH/FallingSand](https://github.com/JuulH/FallingSand) / [GitHub topic: falling-sand](https://github.com/topics/falling-sand)

## Changelog

- 2026-07-05: 初版
