# TouchDesigner GLSL POP 中文語法速查表與指南

## 概覽
本文件協助你撰寫 GLSL POP、GLSL Advanced POP、GLSL Copy POP 的著色器，並說明如何在 GLSL TOP 與 GLSL MAT 中存取 POP buffer。

TouchDesigner 主要支援的 GLSL 版本為 4.60。  
官方 GLSL 文件請參考 [GLSL Documentation](https://www.khronos.org/registry/OpenGL-Refpages/gl4/)。  

---

## 1. 基本提示與指南

- **屬性建立與寫入**  
  - 可於 POP 的 "Create Attributes" 頁面建立新屬性，或於 "Output Attributes" 選擇要修改的輸入屬性。
  - 每個屬性對應一個 Shader Storage Buffer Object (SSBO)。
  - 輸出屬性預設會自動初始化（複製輸入或填預設值），可關閉以提升效能。若未初始化，請務必於 shader 內完整寫入，否則下游 POP 讀取未定值會導致不可預期行為或當機。
  - "Output Access" 可設為 Write-only 或 Read-Write（需原子操作時）。Read-Write 時請先初始化。

- **執行緒數量**  
  - "Auto" 模式：每個元素（點/面/頂點）一個執行緒。
  - "Manual" 模式：自訂執行緒數與 workgroup，需自行管理索引與邊界。
  - `TDNumElements()`：執行緒數（僅 Auto/Attribute 模式有效）。
  - `TDIndex()`：當前執行緒一維索引。

- **Uniforms**  
  - 於參數頁宣告的 uniform 會自動加入 shader。

---

## 2. POP 節點類型與差異

### GLSL POP
- 僅能操作單一屬性類型（點、面、頂點）。
- 可用 "Passes" 參數重複執行 shader。
- 僅能讀取所選類型的輸入屬性。
- 可讀取 index buffer（`TDInputPointIndex`）。

### GLSL Advanced POP
- 可同時操作點、面、頂點屬性。
- 可寫入多個 POP（"Extra Outputs"）。
- 可修改點/面的數量、index buffer。
- 支援 "Per Primitive Batch" 模式。

### GLSL Copy POP
- 針對每個複製執行自訂 GLSL 程式。
- 可透過 TDBuffer 讀取其他 POP buffer。
- 預設僅點會執行自訂 shader，面與頂點僅複製屬性。

---

## 3. 語法速查表

### 執行緒與索引
| 語法 | 說明 |
|------|------|
| `uint TDNumElements();` | 執行緒數（依模式為點/面/頂點數） |
| `uint TDIndex();` | 當前執行緒一維索引 |
| `uint TDInputNumPoints(uint inputIndex);` | 指定輸入的點數 |
| `uint TDInputNumPrims(uint inputIndex);` | 指定輸入的面數 |
| `uint TDInputNumVerts(uint inputIndex);` | 指定輸入的頂點數 |

### 屬性存取
#### 輸入屬性（GLSL POP）
```glsl
TDIn_屬性名() // 讀取輸入0、當前索引的屬性
TDIn_屬性名(uint inputIndex, uint elementId, uint arrayIndex) // 指定輸入/元素/陣列索引
```
#### 輸出屬性
```glsl
屬性名[] // 可寫入的輸出屬性陣列
```
#### 陣列屬性大小
```glsl
const uint cTDArraySize_屬性名; // 陣列屬性大小常數
```
#### 進階 POP 前綴
| 語法 | 說明 |
|------|------|
| `TDInPoint_屬性名()` | 讀取點屬性 |
| `TDInPrim_屬性名()` | 讀取面屬性 |
| `TDInVert_屬性名()` | 讀取頂點屬性 |
| `oTDPoint_屬性名[]` | 輸出點屬性 |
| `oTDPrim_屬性名[]` | 輸出面屬性 |
| `oTDVert_屬性名[]` | 輸出頂點屬性 |

### Index Buffer 操作
```glsl
uint TDInputPointIndex(uint inputIndex, uint vertIndex); // 取得指定輸入、頂點的點索引
```

### Copy POP 專用
| 語法 | 說明 |
|------|------|
| `uint TDNumPoints();` | 輸出點數 |
| `uint TDInputNumPoints();` | 輸入點數 |
| `uint TDCopyIndex();` | 當前複製索引 |
| `TDTemplate_屬性名()` | 讀取模板輸入屬性 |

### 其他 POP buffer 存取（GLSL TOP/MAT）
```glsl
TDBuffer_屬性名(uint elementIndex, uint arrayIndex); // 讀取 POP buffer 屬性
const uint TDBufferLength_屬性名(); // buffer 長度
```

### 內建雜項函數
| 語法 | 說明 |
|------|------|
| `float TDSineLookup(float coord);` | 正弦查表 |
| `mat3 TDRotateToVector(vec3 forward, vec3 up);` | 旋轉矩陣 |
| `float TDPerlinNoise(vec3 v);` | Perlin 噪聲 |
| `float TDSimplexNoise(vec3 v);` | Simplex 噪聲 |
| `vec3 TDHSVToRGB(vec3 c);` | HSV 轉 RGB |
| `vec3 TDRGBToHSV(vec3 c);` | RGB 轉 HSV |
| `float TDRemap(float val, float oldMin, float oldMax, float newMin, float newMax);` | 區間重映射 |
| `float TDLoop(float val, float low, float high);` | 區間循環 |
| `float TDZigZag(float val, float low, float high);` | 之字形循環 |

---

## 4. 範例程式

### GLSL POP 範例
```glsl
void main() {
    const uint id = TDIndex();
    if(id >= TDNumElements())
        return;
    P[id] = TDIn_P(); // 等同於 TDIn_P(0, TDIndex());
}
```
> P 必須在輸入存在，且已選為 Output Attributes。

### GLSL Advanced POP 範例
```glsl
void main() {
    const uint id = TDIndex();
    if(id >= TDNumElements())
        return;
    oTDPoint_P[id] = TDInPoint_P(); // 等同於 TDInPoint_P(0, TDIndex());
}
```
> Basic shader reading the input P attribute and writing it unmodified to the output. Note the extra "Point" prefix to differentiate the class of the attribute, since there is attributes with the same name can exist in different attribute classes.

### GLSL Copy POP 範例
```glsl
void main() {
    const uint id = TDIndex();
    if(id >= TDNumPoints())
        return;
    P[id] = TDIn_P() + TDCopyIndex() * vec3(0.5, 0, 0);
    TDUpdatePointGroups();
}
```
> TDCopyIndex() 取得當前複製索引，TDUpdatePointGroups() 更新點群組。

---

## 5. 常見陷阱與建議

- **記得檢查 id 是否超出範圍**，避免寫出 buffer 範圍導致當機。
- **Output Attributes 必須正確設定**，否則無法寫入。
- **多輸入時，屬性存取需指定 inputIndex**。
- **Manual 模式下 TDNumElements()/TDIndex() 會無效**，需自行管理索引。

---

## 6. 延伸閱讀

- [TouchDesigner 官方文件](https://docs.derivative.ca/Experimental:Write_GLSL_POPs)
- [GLSL 官方文件](https://www.khronos.org/registry/OpenGL-Refpages/gl4/)

---

> 本速查表與指南根據官方文件與原英文 Cheat Sheet 編寫，適合 TouchDesigner 使用者快速查閱與入門。