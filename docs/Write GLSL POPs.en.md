# TouchDesigner GLSL POPs Guide

## Overview

This document assists you in writing GLSL POP, GLSL Advanced POP, and GLSL Copy POP shaders, and explains how to access POP buffers in GLSL TOP and GLSL MAT.

TouchDesigner primarily supports GLSL version 4.60.  
For official GLSL documentation, please refer to [GLSL Documentation](https://www.khronos.org/registry/OpenGL-Refpages/gl4/).

---

## 1. Attribute Read/Write

- New attributes can be created on the "Create Attributes" page of the POP, or you can select which input attributes to modify on the "Output Attributes" page.
- Each attribute corresponds to a Shader Storage Buffer Object (SSBO).
- Output attributes are initialized by default (copying input or filling with default values), which can be turned off for performance improvement. If not initialized, ensure to write to them completely in the shader; otherwise, downstream POPs reading uninitialized values may lead to unpredictable behavior or crashes.
- "Output Access" can be set to Write-only or Read-Write (when atomic operations are needed). If Read-Write, please initialize first.

---

## 2. Thread Count

- "Auto" mode: one thread per element (point/primitive/vertex).
- "Manual" mode: custom thread count and workgroup, requiring manual management of indices and boundaries.
- `TDNumElements()`: number of threads (valid only in Auto/Attribute mode).
- `TDIndex()`: one-dimensional index of the current thread.

---

## 3. Uniforms

- Uniforms declared on the parameter page are automatically added to the shader.

---

## 4. POP Node Types

### GLSL POP

- Can only operate on a single attribute type (point, primitive, vertex).
- The shader can be executed multiple times using the "Passes" parameter.
- Can only read the selected type of input attributes.
- Can read the index buffer (`TDInputPointIndex`).
- "Auto" mode: one thread per element.

#### Example

```glsl
void main() {
    const uint id = TDIndex();
    if(id >= TDNumElements())
        return;
    P[id] = TDIn_P(); // Equivalent to TDIn_P(0, TDIndex());
}
```
> P must exist in the input and be selected as Output Attributes.

---

### GLSL Advanced POP

- Can operate on point, primitive, and vertex attributes simultaneously.
- Can write to multiple POPs ("Extra Outputs").
- Can modify the number of points/primitives and the index buffer.
- Supports "Per Primitive Batch" mode.

#### Example

```glsl
void main() {
    const uint id = TDIndex();
    if(id >= TDNumElements())
        return;
    oTDPoint_P[id] = TDInPoint_P(); // Equivalent to TDInPoint_P(0, TDIndex());
}
```
> Use the "Point" prefix to distinguish attribute types.

---

### GLSL Copy POP

- Executes a custom GLSL program for each copy.
- Can read other POP buffers through TDBuffer.
- By default, only points execute the custom shader; primitives and vertices only copy attributes.

#### Example

```glsl
void main() {
    const uint id = TDIndex();
    if(id >= TDNumPoints())
        return;
    P[id] = TDIn_P() + TDCopyIndex() * vec3(0.5, 0, 0);
    TDUpdatePointGroups();
}
```
> TDCopyIndex() retrieves the current copy index, TDUpdatePointGroups() updates the point groups.

---

## 5. Built-in Functions and Constants

### Threads and Indices

| Syntax | Description |
|--------|-------------|
| `uint TDNumElements();` | Number of threads (based on mode for points/primitives/vertices) |
| `uint TDIndex();` | One-dimensional index of the current thread |
| `uint TDInputNumPoints(uint inputIndex);` | Number of points for the specified input |
| `uint TDInputNumPrims(uint inputIndex);` | Number of primitives for the specified input |
| `uint TDInputNumVerts(uint inputIndex);` | Number of vertices for the specified input |

### Attribute Access

#### Input Attributes (GLSL POP)

```glsl
TDIn_attributeName() // Read attribute from input 0 at the current index
TDIn_attributeName(uint inputIndex, uint elementId, uint arrayIndex) // Specify input/element/array index
```

#### Output Attributes

```glsl
attributeName[] // Writable output attribute array
```

#### Array Attribute Size

```glsl
const uint cTDArraySize_attributeName; // Constant for array attribute size
```

#### Advanced POP Prefixes

| Syntax | Description |
|--------|-------------|
| `TDInPoint_attributeName()` | Read point attribute |
| `TDInPrim_attributeName()` | Read primitive attribute |
| `TDInVert_attributeName()` | Read vertex attribute |
| `oTDPoint_attributeName[]` | Output point attribute |
| `oTDPrim_attributeName[]` | Output primitive attribute |
| `oTDVert_attributeName[]` | Output vertex attribute |

### Index Buffer Operations

```glsl
uint TDInputPointIndex(uint inputIndex, uint vertIndex); // Get point index for specified input and vertex
```

### Copy POP Specific

| Syntax | Description |
|--------|-------------|
| `uint TDNumPoints();` | Number of output points |
| `uint TDInputNumPoints();` | Number of input points |
| `uint TDCopyIndex();` | Current copy index |
| `TDTemplate_attributeName()` | Read template input attribute |

### Accessing Other POP Buffers (GLSL TOP/MAT)

```glsl
TDBuffer_attributeName(uint elementIndex, uint arrayIndex); // Read POP buffer attribute
const uint TDBufferLength_attributeName(); // Buffer length
```

### Built-in Miscellaneous Functions

| Syntax | Description |
|--------|-------------|
| `float TDSineLookup(float coord);` | Sine lookup table |
| `mat3 TDRotateToVector(vec3 forward, vec3 up);` | Rotation matrix |
| `float TDPerlinNoise(vec3 v);` | Perlin noise |
| `float TDSimplexNoise(vec3 v);` | Simplex noise |
| `vec3 TDHSVToRGB(vec3 c);` | Convert HSV to RGB |
| `vec3 TDRGBToHSV(vec3 c);` | Convert RGB to HSV |
| `float TDRemap(float val, float oldMin, float oldMax, float newMin, float newMax);` | Remap value |
| `float TDLoop(float val, float low, float high);` | Loop value within range |
| `float TDZigZag(float val, float low, float high);` | Zigzag loop |

---

## 6. Common Terminology

- **Position (P)**: Standard attribute; built-in optional attributes include normals (N), UVs, colors (Cd), etc.
- **Shader**: A program running on the GPU, divided into Vertex, Pixel, and Compute Shaders.
- **Storage**: A Python dictionary for each Operator, allowing access to additional data.
- **GPU**: Graphics Processing Unit, responsible for efficient graphics and data computation.
- **SOP**: A computational unit containing geometric information such as points, primitives, and vertices.
- **Primitive**: The face types of SOPs, including polygons, curves, NURBS, Bezier, and Patch.
- **TOP**: Image processing unit operating on the GPU.
- **MAT**: Material processing unit that applies shaders to SOPs or 3D objects.
- **Polygon**: A polygon composed of vertices, where each vertex corresponds to a point containing position and attributes.

---

> Source: [TouchDesigner Official Documentation](https://docs.derivative.ca/index.php?title=Experimental:Write_GLSL_POPs&oldid=33782)