# TouchDesigner GLSL POPs ガイド

## 概要

このドキュメントは、GLSL POP、GLSL Advanced POP、GLSL Copy POP のシェーダーを作成するためのガイドであり、GLSL TOP および GLSL MAT で POP バッファにアクセスする方法を説明します。

TouchDesigner が主にサポートする GLSL バージョンは 4.60 です。  
公式 GLSL ドキュメントは [GLSL Documentation](https://www.khronos.org/registry/OpenGL-Refpages/gl4/) を参照してください。

---

## 1. 属性の読み書き

- POP の "Create Attributes" ページで新しい属性を作成するか、"Output Attributes" で変更する入力属性を選択できます。
- 各属性は Shader Storage Buffer Object (SSBO) に対応しています。
- 出力属性はデフォルトで自動的に初期化されます（入力をコピーするか、デフォルト値を設定します）。パフォーマンスを向上させるために無効にすることができます。初期化されていない場合は、シェーダー内で完全に書き込む必要があります。そうしないと、下流の POP が未定義の値を読み取ることになり、予期しない動作やクラッシュを引き起こす可能性があります。
- "Output Access" は Write-only または Read-Write（原子操作が必要な場合）に設定できます。Read-Write の場合は、事前に初期化してください。

---

## 2. スレッド数

- "Auto" モード：各要素（ポイント/プリミティブ/頂点）に対して1つのスレッド。
- "Manual" モード：スレッド数とワークグループをカスタマイズし、インデックスと境界を自分で管理する必要があります。
- `TDNumElements()`：スレッド数（Auto/Attribute モードでのみ有効）。
- `TDIndex()`：現在のスレッドの1次元インデックス。

---

## 3. Uniforms

- パラメータページで宣言された uniform は自動的にシェーダーに追加されます。

---

## 4. POP ノードタイプ

### GLSL POP

- 単一の属性タイプ（ポイント、プリミティブ、頂点）のみを操作できます。
- "Passes" パラメータを使用してシェーダーを繰り返し実行できます。
- 選択したタイプの入力属性のみを読み取ることができます。
- インデックスバッファ（`TDInputPointIndex`）を読み取ることができます。
- "Auto" モード：各要素に対して1つのスレッド。

#### 例

```glsl
void main() {
    const uint id = TDIndex();
    if(id >= TDNumElements())
        return;
    P[id] = TDIn_P(); // TDIn_P(0, TDIndex()) と同等
}
```
> P は入力が存在し、出力属性として選択されている必要があります。

---

### GLSL Advanced POP

- ポイント、プリミティブ、頂点属性を同時に操作できます。
- 複数の POP に書き込むことができます（"Extra Outputs"）。
- ポイント/プリミティブの数やインデックスバッファを変更できます。
- "Per Primitive Batch" モードをサポートしています。

#### 例

```glsl
void main() {
    const uint id = TDIndex();
    if(id >= TDNumElements())
        return;
    oTDPoint_P[id] = TDInPoint_P(); // TDInPoint_P(0, TDIndex()) と同等
}
```
> 属性タイプを区別するために "Point" プレフィックスを使用します。

---

### GLSL Copy POP

- 各コピーに対してカスタム GLSL プログラムを実行します。
- TDBuffer を介して他の POP バッファを読み取ることができます。
- デフォルトでは、ポイントのみがカスタムシェーダーを実行し、プリミティブと頂点は属性をコピーするだけです。

#### 例

```glsl
void main() {
    const uint id = TDIndex();
    if(id >= TDNumPoints())
        return;
    P[id] = TDIn_P() + TDCopyIndex() * vec3(0.5, 0, 0);
    TDUpdatePointGroups();
}
```
> TDCopyIndex() は現在のコピーインデックスを取得し、TDUpdatePointGroups() はポイントグループを更新します。

---

## 5. ビルトイン関数と定数

### スレッドとインデックス

| 構文 | 説明 |
|------|------|
| `uint TDNumElements();` | スレッド数（モードに応じてポイント/プリミティブ/頂点数） |
| `uint TDIndex();` | 現在のスレッドの1次元インデックス |
| `uint TDInputNumPoints(uint inputIndex);` | 指定された入力のポイント数 |
| `uint TDInputNumPrims(uint inputIndex);` | 指定された入力のプリミティブ数 |
| `uint TDInputNumVerts(uint inputIndex);` | 指定された入力の頂点数 |

### 属性アクセス

#### 入力属性（GLSL POP）

```glsl
TDIn_属性名() // 入力0、現在のインデックスの属性を読み取る
TDIn_属性名(uint inputIndex, uint elementId, uint arrayIndex) // 指定された入力/要素/配列インデックス
```

#### 出力属性

```glsl
属性名[] // 書き込み可能な出力属性配列
```

#### 配列属性サイズ

```glsl
const uint cTDArraySize_属性名; // 配列属性サイズの定数
```

#### 高度な POP プレフィックス

| 構文 | 説明 |
|------|------|
| `TDInPoint_属性名()` | ポイント属性を読み取る |
| `TDInPrim_属性名()` | プリミティブ属性を読み取る |
| `TDInVert_属性名()` | 頂点属性を読み取る |
| `oTDPoint_属性名[]` | 出力ポイント属性 |
| `oTDPrim_属性名[]` | 出力プリミティブ属性 |
| `oTDVert_属性名[]` | 出力頂点属性 |

### インデックスバッファ操作

```glsl
uint TDInputPointIndex(uint inputIndex, uint vertIndex); // 指定された入力、頂点のポイントインデックスを取得
```

### Copy POP 専用

| 構文 | 説明 |
|------|------|
| `uint TDNumPoints();` | 出力ポイント数 |
| `uint TDInputNumPoints();` | 入力ポイント数 |
| `uint TDCopyIndex();` | 現在のコピーインデックス |
| `TDTemplate_属性名()` | テンプレート入力属性を読み取る |

### その他の POP バッファアクセス（GLSL TOP/MAT）

```glsl
TDBuffer_属性名(uint elementIndex, uint arrayIndex); // POP バッファ属性を読み取る
const uint TDBufferLength_属性名(); // バッファの長さ
```

### ビルトイン雑多関数

| 構文 | 説明 |
|------|------|
| `float TDSineLookup(float coord);` | サインルックアップ |
| `mat3 TDRotateToVector(vec3 forward, vec3 up);` | 回転行列 |
| `float TDPerlinNoise(vec3 v);` | パーリンノイズ |
| `float TDSimplexNoise(vec3 v);` | シンプレックスノイズ |
| `vec3 TDHSVToRGB(vec3 c);` | HSV から RGB への変換 |
| `vec3 TDRGBToHSV(vec3 c);` | RGB から HSV への変換 |
| `float TDRemap(float val, float oldMin, float oldMax, float newMin, float newMax);` | 範囲の再マッピング |
| `float TDLoop(float val, float low, float high);` | 範囲のループ |
| `float TDZigZag(float val, float low, float high);` | ジグザグループ |

---

## 6. よくある用語の説明

- **Position (P)**：標準属性、内蔵の選択可能な属性には法線（N）、UV、色（Cd）などがあります。
- **Shader**：GPU 上で実行されるプログラムで、頂点シェーダー、ピクセルシェーダー、コンピュートシェーダーに分かれます。
- **Storage**：各オペレーターの Python 辞書で、追加データにアクセスできます。
- **GPU**：グラフィックプロセッサで、高速なグラフィックおよびデータ計算を担当します。
- **SOP**：ポイント、プリミティブ、頂点などの幾何情報を含む演算ユニットです。
- **Primitive**：SOP の面タイプで、多角形、曲線、NURBS、Bezier、Patch などがあります。
- **TOP**：画像演算ユニットで、GPU 上で動作します。
- **MAT**：マテリアル演算ユニットで、シェーダーを SOP または 3D オブジェクトに適用します。
- **Polygon**：頂点で構成される多角形で、各頂点はポイントに対応し、ポイントは位置と属性を含みます。

---

> 出典：[TouchDesigner 公式ドキュメント](https://docs.derivative.ca/index.php?title=Experimental:Write_GLSL_POPs&oldid=33782)