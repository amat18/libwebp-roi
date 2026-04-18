# libwebp-roi — ROI エンコード機能 ドキュメント

> webmproject/libwebp v1.6.0 のフォーク

---

## 目次

1. [概要](#概要)
2. [メカニズム](#メカニズム)
3. [使い方](#使い方)
   - [矩形ROI（`-roi`）](#矩形roi--roi)
   - [非矩形ROI（`-roi_mb`）](#非矩形roi--roi_mb)
   - [マクロブロック番号の求め方](#マクロブロック番号の求め方)
4. [VP8仕様との対応関係](#vp8仕様との対応関係)
5. [変更ファイル](#変更ファイル)
6. [制限事項](#制限事項)
7. [クレジット](#クレジット)

---

## 概要

このフォークは `cwebp` に2種類のROI（関心領域）エンコードオプションを追加します。
ROI内のマクロブロックを最高画質、それ以外を最低画質でエンコードすることで、
ファイルサイズを抑えながら重要領域の画質を維持します。
出力は通常のWebPファイルであり、標準的な画像ビュアーでそのまま閲覧できます。

---

## メカニズム

WebPロッシーエンコード（VP8ベース）は、画像を**16×16ピクセルのマクロブロック**に分割し、
最大4つの**セグメント**にグループ化します。各セグメントは独自の量子化パラメータを持ちます。

このROI機能は2つのセグメントを以下の設定で強制します：

```
segment 0（ROI外）: quant=127, fstrength=63  →  最低画質
segment 1（ROI内）: quant=0,   fstrength=0   →  最高画質
```

各マクロブロックをROI内外に応じてセグメントに割り当てます：

```
入力画像の例（80×48px = 5×3 マクロブロック）

  0   16   32   48   64   80  px
  |    |    |    |    |    |
0 +----+----+----+----+----+
  |  0 |  1 |  2 |  3 |  4 |
16+----+#####+#####+----+----+
  |  5 |# 6#|# 7#|  8 |  9 |    ← -roi 16 16 47 47
32+----+#####+#####+----+----+      または -roi_mb 6,7
  | 10 | 11 | 12 | 13 | 14 |
48+----+----+----+----+----+

  #### = segment 1（ROI内・高画質）
  [  ] = segment 0（ROI外・低画質）
```

#### セグメント割り当てフロー

```
SetupFilterStrength()
  |
  +-- segment 0: quant=127, fstrength=63  （ROI外: 最低画質）
  +-- segment 1: quant=0,   fstrength=0   （ROI内: 最高画質）
  |
  v
SimplifySegments()
  |
  +-- セグメント数が1に削減された場合、強制的に2に戻す
  |
  +-- roi_mb_mask が非NULLの場合（-roi_mb モード）:
  |     mb_info[i].segment = roi_mb_mask[i] ? 1 : 0
  |
  +-- roi == 1 の場合（-roi モード）:
  |     col = i % mb_w,  row = i / mb_w
  |     mb_info[i].segment = (col in [x1..x2] AND row in [y1..y2]) ? 1 : 0
  |
  v
VP8エンコーダが各マクロブロックにセグメント別量子化を適用
```

---

## 使い方

### 矩形ROI（`-roi`）

矩形の関心領域をピクセル座標で指定します。

```
cwebp -roi <x1> <y1> <x2> <y2> input.png -o output.webp
```

| 引数 | 説明 |
|---|---|
| `x1`, `y1` | ROI左上座標（ピクセル、inclusive） |
| `x2`, `y2` | ROI右下座標（ピクセル、inclusive） |

座標は内部で16px単位のマクロブロックグリッドに切り捨てられます。

**例:**
```
cwebp -roi 64 64 192 192 photo.png -o photo_roi.webp
```

---

### 非矩形ROI（`-roi_mb`）

マクロブロックの番号をカンマ区切りで直接指定します。任意形状のROIが指定可能です。

```
cwebp -roi_mb <idx1>,<idx2>,... input.png -o output.webp
```

| 引数 | 説明 |
|---|---|
| `idx1,idx2,...` | ROIにしたいマクロブロックの番号（0始まり、カンマ区切り） |

`-roi_mb` が指定された場合、`-roi` より優先されます。

**例:**
```
# 5×3マクロブロック画像（80×48px）で中央の2ブロックをROIに指定
cwebp -roi_mb 6,7 photo.png -o photo_roi.webp
```

---

### マクロブロック番号の求め方

マクロブロックは左上から右へ、行ごとに0始まりで番号付けされます。

```
画像サイズ W×H px の場合:

  mb_w = ceil(W / 16)   （列数）
  mb_h = ceil(H / 16)   （行数）

  マクロブロック番号 = row * mb_w + col

例: 80×48px（mb_w=5, mb_h=3）

   col:  0    1    2    3    4
row 0: [  0][  1][  2][  3][  4]
row 1: [  5][  6][  7][  8][  9]
row 2: [ 10][ 11][ 12][ 13][ 14]

ピクセル座標 (px, py) からMB番号を求める式:

  idx = (py / 16) * mb_w + (px / 16)
```

---

## VP8仕様との対応関係

本実装はVP8仕様（RFC 6386）のセグメントベース量子化機能を直接利用しています。
標準デコーダの変更は一切不要です。

### WebP → VP8 の階層構造

```
ファイル（RIFF形式）
└── VP8 chunk
    ├── 非圧縮ヘッダ（10 bytes）  frame_type / version / サイズ
    └── VP8ビットストリーム（ブール算術符号）
        ├── フレームヘッダ          ← セグメント定義
        └── マクロブロックデータ    ← MBごとのセグメントID
```

WebPのロッシー形式は「VP8キーフレーム1枚をRIFFでラップしたもの」です。
セグメント機能はすべてVP8仕様に由来します。

### フレームヘッダのセグメント定義（RFC 6386 §9.3 / §9.6）

フレームヘッダの圧縮領域に以下が書き込まれます：

```
segmentation_enabled          L(1)  = 1    ← ROI使用時は必ず1
update_mb_segmentation_map    L(1)  = 1    ← MBごとにIDを書き込む
update_segment_feature_data   L(1)  = 1    ← セグメントごとの量子化値を書く
  segment_feature_mode        L(1)  = 0    ← 絶対値モード（差分でなく）

  segment[0].yac_qi           L(7)  = 127  ← ROI外（最低画質）
  segment[1].yac_qi           L(7)  = 0    ← ROI内（最高画質）
```

`yac_qi` はY面ACの量子化インデックスです（0=高画質 / 127=低画質）。

libwebpでの対応箇所（`SetupFilterStrength()`）：

```c
m->quant = (i == 0) ? 127 : (i == 1) ? 0 : m->quant;
//          ↑ segment[0].yac_qi=127   ↑ segment[1].yac_qi=0
```

### マクロブロックデータのセグメントID（RFC 6386 §19.3）

フレームヘッダの後、MBはラスタースキャン順に符号化されます。
各MBの先頭にセグメントIDがエントロピー符号化されて書かれます：

```
segment_id  ∈ {0, 1, 2, 3}   ← 確率木（mb_segment_tree）で符号化
```

`ApplyROISegments()` はこれを2値に強制します：

```c
enc->mb_info[i].segment = roi_mb_mask[i] ? 1 : 0;
//                         ↑ MBインデックスiのsegment_id
```

### ループフィルタとの関係（RFC 6386 §9.4）

VP8のセグメントは量子化だけでなくループフィルタ強度も制御します：

```
filter_strength[0] = 63   ← ROI外: 強フィルタ（ブロックノイズを目立たなくする）
filter_strength[1] = 0    ← ROI内: フィルタなし（エッジをそのまま保持）
```

高画質側にはフィルタ不要、低画質側には必要、という設定が理にかなっています。

### ApplyROISegments() の設計

`SimplifySegments()` はVP8エンコーダが不要セグメントを統合する最適化処理です。
多パス処理でこれが繰り返し呼ばれるため、ROI割り当ては独立関数として
`SimplifySegments()` の**後**に一度だけ適用するように設計しています：

```
VP8SetSegmentParams()
  ├── SetupFilterStrength()   ← segment[0/1] の quant / fstrength を設定
  ├── SimplifySegments()      ← 不要セグメントの統合（ROIには触れない）
  └── ApplyROISegments()      ← ROI割り当て（MB単位で segment_id を確定）
```

この順序により、SimplifySegments()の縮退結果に関わらずROI割り当てが
常に最終状態として確定します。

---

## 変更ファイル

| ファイル | 変更内容 |
|---|---|
| `src/webp/encode.h` | `WebPConfig` に `roi`, `roi_x1/x2/y1/y2`, `roi_mb_mask` を追加 |
| `src/enc/quant_enc.c` | `SetupFilterStrength()` にROIセグメント量子化設定を追加。ROIセグメント割り当てを `ApplyROISegments()` として独立関数化し、`VP8SetSegmentParams()` から呼び出す |
| `examples/cwebp.c` | `-roi` と `-roi_mb` CLIオプション・ヘルプを追加 |

---

## 制限事項

**1. 矩形モードの座標精度**

`-roi` の座標は16pxグリッドに切り捨てられます。サブマクロブロック精度は指定できません。

```
指定: x1=10  →  実際: x1=0   (10/16 = 0, つまり 0px)
指定: x1=20  →  実際: x1=1   (20/16 = 1, つまり 16px)
```

`-roi_mb` はマクロブロック単位で直接指定するため、この制約はありません。

**2. 画質は2値のみ**

ROI内：最高画質、ROI外：最低画質の2段階のみです。中間値・グラデーションは未対応です。

```
画質

 high |####|
      |####|
      |    |________
 low  |
      +----+---------> ROIからの距離
      ROI  outside
```

**3. `-q` / `-sns` との競合**

ROIモード使用時、`-q`（品質）や `-sns`（空間ノイズシェーピング）は内部で上書きされます。

**4. ロッシーのみ**

ROIエンコードはロッシーWebPのみに適用されます。`-lossless` では無効です。

---

## クレジット

本実装の一部およびコードレビューは [Claude Code](https://claude.ai/code)（Anthropic）を使用して行いました。
