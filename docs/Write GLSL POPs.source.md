# Overview

This helps you write shaders for the GLSL POP, GLSL Advanced POP, GLSL Copy POP, and access POPs buffers in GLSL_TOP and GLSL_MAT

The official GLSL documentation can be found at this address. TouchDesigner's main supported version of GLSL is 4.60.

---

## Reading and writing attributes

GLSL POPs can either create and write new attributes with the "Create Attributes" page of each POP, or select input attributes to be modified, with the "Output Attributes" parameters, that are then allocated for writing. Unmodified input attributes for the POP from the first input are passed to downstream POP with references and don't use extra memory. See Memory Reduction via References in Experimental:POP#Some_POP_Features_and_Concepts.

Each attribute is a Shader Storage Buffer Object (SSBO).

Output attributes (including new attributes) are unitialized, and it is up to the shader code to write values, except if "Initialize Output Attributes" is On (default), in that case, for existing attributes, a copy from the input values is performed, and new attributes are filled with the default values from the "Create Attributes" page. This is an extra shader dispatch happening before the custom shader is run, and should be off for performance if the users know all the output values are written to. Reading from unitialized values in downstream POPs can cause unpredictable behavior and crashes.

The "Output Access" menu allows to set up the output attributes to be write-only (writeonly SSBO qualifier) or Read-Write. Read-Write is necessary to perform atomic operations. In that case the output values should be initialized first, either using the toggle parameter, or during the first pass in a multi-pass POP (GLSL POP only).

The input attributes from all inputs are automatically made available for reading.

---

## Number of Threads

See Compute_Shader and Compute Shader for an overview on how threads are arranged in number of dispatches and number of threads per workgroup. The GLSL POPs offer simplified modes as well as full control over the number of threads. In most case one will want one thread to read and write the value for one attribute element (one point, vertex or primitive), which is accomplished with different menu entries depending on the GLSL POP.

---

## Uniforms

Uniforms declared with the parameter fields on their respective pages are declared automatically in the shader.

---

## POPs overview

### GLSL POP

This is the simplest of the GLSL POPs. It allows the user to work on one class of attributes at a time: point, vertex, or primitive attributes.

The only feature present in the GLSL POP and not the GLSL Advanced POP is the ability to run the shader multiple times, with the "Passes" parameter.

The input attributes of the selected class from all inputs are available for reading, but not from the other classes.

The index buffer for all inputs is also available for reading, see `uint TDInputPointIndex(uint inputIndex, uint vertIndex)` below.

The "Number of Threads" mode "Auto" creates one thread per element of the selected class, point, vertex, or primitive.

"Number of Elements in Other Input" is similar to "Auto" for other inputs.

If the number of elements for the selected class and input is known on the GPU and not on the CPU (see Experimental:POP#Indirect_POPs_and_Memory_Management), an indirect command buffer is filled, and the compute shader is launched with an indirect dispatch.

"Manual Number of Elements" creates threads according to the "Number of Elements" parameter.

"Number of Elements" from Attribute create threads according the value of the first element of the POP and attribute specified. In that case the compute shader is launched with an indirect dispatch.

In all those cases, the actual number of threads is the requested number of threads rounded up to the nearest workgroup size, `uint TDNumElements()` contains the number of requested threads, and `uint TDIndex()` contains a 1d index.

With the "Manual" mode, the work group size and number of dispatches can be specified in 3 dimensions, in that case TDNumElements() and TDIndex() are undefined and will causes a compile error.

**Example shader:**

```glsl
void main() {
    const uint id = TDIndex();
    if(id >= TDNumElements())
        return;
        
    P[id] = TDIn_P(); //same as TDIn_P(0, TDIndex());
}
```

For `P[id] = TDIn_P();` to compile without error, P needs to be present in the input, and selected for writing in "Output Attributes".

`TDIn_P()` reads the P attribute in input 0 (first input), at element TDIndex(), it is a short hand for `TDIn_P(0, id)`, since id is mapped to TDIndex().

This is written to element id in the output with P[id].

As described above, TDNumElements() is the number of requested threads (one per point in "Auto" mode and with Attribute Class "Point" selected). The actual number of threads is most likely higher, since it is rounded up to the work group size, which is set to 32 for NVIDIA hardware and 64 for AMD hardware. The if condition prevents the shader to write outside of the bounds of the allocated SSBO, which can lead to unpredictible results and crashes.

---

### GLSL Advanced POP

The GLSL Advanced POP allows to do what the GLSL POP can with the following differences and additional features:

- ability to read, write and create all classes of attributes (point, primitive, vertex) in the same shader
- ability to write to attributes from multiple POPs ("Extra Outputs" page). The modified outputs can be selected with the GLSL Select POP.
- ability to modify the number of points and primitives ("Output" page)
- ability to write to index buffer
- additional "Per Primitive Batch" Shader Dispatch Mode, which runs the shader once per primitive batch present in the first input.

All the primitives of the same type (Triangles, Quads, Line Strips, Lines, Point Primitives) are grouped together in the index buffer for processing and rendering efficiency. When working with multiple primitive types it can be advantageous to dispatch the shader once per primitive type, since in that case some variables are constants and the users knows all the primitives are the same type without extra if() conditions.

When working with primitives the GLSL Advanced POP allows to set the number of primitives from each type, using the parameters or attributes, but it can also be convenient to use a Topology POP to combine the result of multiple GLSL Advanced POPs (it is often advantageous to have one shader per attribute class) and set the topology info.

**Example shader:**

```glsl
void main() {
    const uint id = TDIndex();
    if(id >= TDNumElements())
        return;
        
    oTDPoint_P[id] = TDInPoint_P(); //same as TDInPoint_P(0, TDIndex());
}
```

Basic shader reading the input P attribute and writing it unmodified to the output. Note the extra "Point" prefix to differentiate the class of the attribute, since there is attributes with the same name can exist in different attribute classes.

---

### GLSL Copy POP

The GLSL Copy POP allows to run custom GLSL code on each copy. All the copies are limited to have the same topology though, similar to the Copy POP. Also similar to the copy POP a template input is available. There are no other inputs so other POP attributes need to be accessed using the "POP Buffers" page and the TDBuffer() function, similar to GLSL TOP and MAT, see below.

Similar to how the Copy POP works internally, one compute shader is dispatched to update the points, one for the vertices, and one for the primitives.

By default, only the points run a custom shader, while vertices and primitives only copy input attributes to all copies.

However it is also possible to have a custom shader for the vertices and for the primitives.

**Example Point shader:**

```glsl
void main() {
    const uint id = TDIndex();
    if(id >= TDNumPoints())
        return;
        
    //TDInputNumPoints(): number of points on input geo
    //TDInputPointIndex(), TDInputIndex(): point id matching current point id on input
    //TDCopyIndex(): copy number
    
    //TDTemplate_Attrib() functions to sample template input attributes

    //P[id] = TDIn_P() + TDTemplate_P();
    //same as P[id] = TDIn_P(TDInputPointIndex()) + TDTemplate_P(TDCopyIndex());

    P[id] =  TDIn_P() + TDCopyIndex() * vec3(0.5, 0, 0);
    
    //updates existing point groups if any
    TDUpdatePointGroups();
}
```

One thread is executed per output point, meaning TDIndex() is the output point index. The TDInputPointIndex() refers to the index of the input point matching the current output point. TDIn_P() with no attribute reads TDInputPointIndex() since we assume we want to read from the matching original point in the input. TDCopyIndex() is the current copy index. TDTemplate_P() reads P from the template input at the copy index location when no argument are provided. TDUpdatePointGroups() is a helper function to update any point group that was on the input: in each copy points that were part of a group in the input will be part of the same group.

---

## Built-in functions and constants

### Number of Elements and Indexing

GLSL POP, and GLSL Advanced POP single dispatch mode and per batch mode

```glsl
uint TDNumElements();
//number of requested threads based on dispatch mode and number of threads mode (so depending on mode sometimes = TDInputNumPoints() etc)
//(actual number of threads is ceil of workgroup size 32 on nvidia)
//undefined when number of threads is manual and user chooses workgroup size and number of workgroup dispatches

uint TDIndex();
//1d index for current thread
//undefined when number of threads is manual

uint TDInputNumPoints(uint inputIndex); //number of points in inputIndex input
uint TDInputNumPrims(uint inputIndex); //number of prims in inputIndex input
uint TDInputNumVerts(uint inputIndex); //number of verts in inputIndex input

//returns 0 if input doesn't exist
TDInputNumPoints() = TDInputNumPoints(0);
TDInputNumPrims() = TDInputNumPrims(0);
TDInputNumVerts() = TDInputNumVerts(0);
//if no input index argument is specified it returns values for input 0
```

GLSL POP only:

```glsl
uint TDInputNumElements(); //wrapper for TDInputNumPoints() or TDInputNumPrims() or TDInputNumVerts() depending on attribute class
```

---

### Index Buffer Access

**Read Access**

```glsl
uint TDInputPointIndex(uint inputIndex, uint vertIndex); //index buffer access with bounds checking
uint TDInputPointIndex(uint vertIndex) = TDInputPointIndex(0, uint vertIndex);
```

**Write Access**

Only enabled in the GLSL Advanced POP when one of the Max Primitives parameter is set to custom.

```glsl
uint I[]; //array variable
```

---

### Attribute Access

All the attributes from the inputs are bound and available for reading.

Attributes added to the “Output Attributes” param or attributes created in the POP are allocated and can be written to.

A reference to the input attributes from the first input is created for the other attributes, similar to unmodified attributes in the other POPs ((r) in info panel)

#### GLSL POP:

**Input Attributes**

```glsl
attribType TDIn_AttribName(uint inputIndex, uint elementId, uint arrayIndex) //to access input inputIndex AttribName

TDIn_AttribName() = TDIn_AttribName(0, TDIndex(), 0); //if no argument is specified it returns the value at elementId for input 0, at arrayIndex 0 if attribute is an array attribute
TDIn_AttribName(uint inputIndex, uint elementId) = TDIn_AttribName(inputIndex, elementId, 0); arrayIndex is optional and defaults to 0

//this performs bounds checking, return last elements if outside of bounds, last arrayIndex if arrayIndex is out of bounds

attribType TDInCache_AttribName(uint inputIndex, uint cacheIndex, uint elemId, uint arrayIndex); //to access frames cached with cache POPs, arrayIndex is optional
```

**Output Attributes**

```glsl
attribType AttribName[]; //output variable, no function to allow use in atomic functions (AtomicAdd, etc)
```

**Attribute Array Size**

```glsl
const uint cTDArraySize_Attrib;
```

---

#### GLSL Advanced POP:

Similar to GLSL POP with extra attribute class prefix in function names.

**Input Attributes**

```glsl
attribType TDInPoint_AttribName(uint inputIndex, uint pointId, uint arrayIndex);
attribType TDInPrim_AttribName(uint inputIndex, uint primId, uint arrayIndex);
attribType TDInVert_AttribName(uint inputIndex, uint vertId, uint arrayIndex);

//similar to GLSL POP function for point, prim, vert attribute
attribType TDInCachePoint_AttribName(uint inputIndex, uint cacheIndex, uint elemId, uint arrayIndex);
attribType TDInCacheVert_AttribName(uint inputIndex, uint cacheIndex, uint elemId, uint arrayIndex);
attribType TDInCachePrim_AttribName(uint inputIndex, uint cacheIndex, uint elemId, uint arrayIndex);
```

**Output Attributes**

```glsl
attribType oTDPoint_AttribName[];
attribType oTDPrim_AttribName[];
attribType oTDVert_AttribName[];
```

**Attribute Array Size**

```glsl
const uint cTDArraySizePoint_Attrib;
const uint cTDArraySizePrim_Attrib;
const uint cTDArraySizeVert_Attrib;
```

---

#### GLSL Advanced POP Extra outputs:

Similar to GLSL Advanced POP, except the extra output names are part of the functions, instead of them being access by an index.

**Number of Elements in Input**

```glsl
uint TDInputNumPoints_OutputName();
uint TDInputNumPrims_OutputName();
uint TDInputNumVerts_OutputName();
```

**Input Attributes**

```glsl
attribType TDInPoint_OutputName_AttribName(uint pointId, uint arrayIndex);
attribType TDInPrim_OutputName_AttribName(uint primId, uint arrayIndex);
attribType TDInVert_OutputName_AttribName(uint vertId, uint arrayIndex);
```

**Output Attributes**

```glsl
attribType oTDPoint_OutputName_AttribName[];
attribType oTDPrim_OutputName_AttribName[];
attribType oTDVert_OutputName_AttribName[];
```

**Attribute Array Size**

```glsl
const uint cTDArraySizePoint_OutputName_Attrib;
const uint cTDArraySizePrim_OutputName_Attrib;
const uint cTDArraySizeVert_OutputName_Attrib;
```

**Index Buffer**

```glsl
uint TDInputPointIndex_OuputName(uint vertIndex);
```

---

#### GLSL Copy POP:

**Number of Elements and Indexing**

**Point shader**

```glsl
uint TDNumPoints(); //number of output points
uint TDInputNumPoints(); //number of input points
```

**Vertex shader**

```glsl
uint TDNumVertsBatch(); //number of output verts in current primitive batch
uint TDInputNumVertsBatch(); //number of input verts in current primitive batch
uint TDInputNumVerts(); //total number of input verts
uint TDVertIndex(); //current global vert index
```

**Primitive shader**

```glsl
uint TDNumPrimsBatch(); //number of output prims in current primitive batch
uint TDInputNumPrimsBatch(); //number of input prims in current primitive batch
uint TDInputNumPrims(); //total number of input prims
uint TDPrimIndex();  //current global prim index
```

**All GLSL Copy POP shaders**

```glsl
TDInputIndex(); //index matching the input (source) point/prim/vert
TDCopyIndex(); //current copy index
TDTemplateNumPoints(); //number of poinst in template input
```

**Attribute Access:**

**Source input:**

```glsl
attribType TDIn_AttribName(uint elementId, uint arrayIndex);
TDIn_AttribName(uint elementId) = TDIn_AttribName(elementId, 0); //arrayIndex is optional and defaults to 0
TDIn_AttribName() = TDIn_AttribName(TDInputIndex(), 0); //if no argument is specified it returns the value for element at TDInputIndex()

const uint cTDArraySize_Attrib; //constant with size of array for array attributes
```

**Template input:**

```glsl
attribType TDTemplate_AttribName(uint elementId, uint arrayIndex);
TDTemplate_AttribName(uint elementId) = TDTemplate_AttribName(elementId, 0);
TDTemplate_AttribName() = TDTemplate_AttribName(TDCopyIndex(), 0); //if no argument is specified it returns the value for the point at TDCopyIndex()

const uint cTDTemplateArraySize_Attrib; //constant with size of array for array attributes
```

**Other POP attribute access:**

Since the GLSL Copy POP doesn't have other inputs, other POP attributes can be accessed with the TDBuffer() function and related constants, see Experimental:Write_GLSL_POPs#GLSL_TOP_and_MAT:_POP_attribute_access

**Output Attributes**

```glsl
attribType AttribName[]; //output variable, no function to allow use in atomic functions (AtomicAdd, etc)
```

---

## Misc Functions

**Point Shader:**

```glsl
void TDUpdatePointGroups(); //updates existing point groups if any
```

**Vertex Shader:**

```glsl
void TDUpdateTopology(); //updates topology (index buffer and line strip index buffer for line strips)
```

**Primitive Shader**

```glsl
void TDUpdateLineStripsInfo(); //updates topology (line strips info buffer)
void TDUpdatePrimGroups(); //updates existing prim groups if any
```

---

### Primitive restart index for line strips

The maximum unsigned value, 0xFFFFFFFF in 32bits

```glsl
const uint cTDPrimIndexRestart;
```

---

### Dimension info

GLSL POP and GLSL Advanced POP only

```glsl
uint[cTDDimSize] TDDimension(); //returns an array with the dimensions
uint[cTDDimSize] TDDimCoords(uint pointIndex); //returns an array with the coordinates of pointIndex relative to dimensions: matches _DimI[] built-in attribute
const uint cTDDimSize; //constant that holds the number of dimensions
```

---

### Primitive Batches info and built-in attributes equivalent

GLSL POP, and GLSL Advanced POP
(for per primitive batch dispatch mode, only useful for secondary inputs)

```glsl
uint TDInputPrimIndex(uint vertIndex);
//The index of the primitive vertIndex vertex is part of.
//This matches builtin attribute _PrimI run on vertices.
//This also matches Line Strip Index on Line Metrics for Line Strips

uint TDInputVertPrimIndex(uint vertIndex);
//the index of vertIndex vertex in the current primitive (goes from 0 to numVertsPerPrim - 1)
//this matches builtin attribute _VertPrimI run on vertices.

uint TDInputPrimType(uint primIndex); //primitive type for primIndex primitive
uint TDInputPrimType_Vert(uint vertIndex); //primitive type for the primitive vertIndex is part of

//Primitive types:
const uint cTDTrianglesType = 0;
const uint cTDQuadsType = 1;
const uint cTDLineStripsType = 2;
const uint cTDLinesType  = 3;
const uint cTDPointPrimsType  = 4;

uint TDInputNumVertsPerPrim(uint inputIndex, uint primIndex);
//the number of vertices in primIndex primitive in input inputIndex
//this matches builtin attribute _NumVertsPerPrim run on primitives

uint TDInputNumVertsPerPrimFromVert(uint inputIndex, uint vertIndex);
//the number of vertices of the primitive vertIndex vertex is part of in input inputIndex
//this matches builtin attribute _NumVertsPerPrim run on vertices
//this also matches Number of Verts in Line Metrics

uint TDInputPrimVertsStartIndex(uint inputIndex, uint primIndex); //the index of the first vertex for primIndex primitive in input inputIndex
uint TDInputPrimsStartIndex(uint inputIndex, uint primType); //index of first input primitive in primType primitive batch
uint TDInputVertsStartIndex(uint inputIndex, uint primType); //index of first input vertex in primType primitive batch
```

For the above, if only one argument is specified, inputIndex = 0.

For GLSL POP, extra functions with no argument are wrapping some of the above as follow:

- run on vertices:
  - `TDInputPrimIndex() = TDInputPrimIndex(TDIndex());`
  - `TDInputPrimType() = TDInputPrimType_Vert(TDIndex());`
  - `TDInputNumVertsPerPrim() = TDInputNumVertsPerPrim_Vert(TDIndex());`
  - `TDInputVertPrimIndex() = TDInputVertPrimIndex(TDIndex());`
- run on primitives
  - `TDInputPrimIndex() = TDIndex();`
  - `TDInputPrimType() = TDInputPrimType(TDIndex());`
  - `TDInputNumVertsPerPrim() = TDInputNumVertsPerPrim(TDIndex());`
  - `TDInputPrimVertsStartIndex() = TDInputPrimVertsStartIndex(TDIndex());`

These have "Input" in their names because GLSL Advanced POP can have a different output, and to have the same functions for GLSL POP and GLSL Advanced POP. The functions for the output are currently not defined for GLSL Advanced POP single dispatch mode.

---

### GLSL Advanced POP per primitive batch dispatch mode:

```glsl
uint TDInputPrimIndex();
//for primitives : index of current input primitive (TDInputPrimsStartIndex() + TDIndex())
//for vertices : index of primitive the current vertex is part of

uint TDInputPrimVertsStartIndex();
//for primitives : number of verts of current input primitive
//for vertices : number of verts of primtive the current vertex is part of

uint TDInputPrimType();
//for primitives : primitive type of current input primitive (constant over batch)
//for vertices : primitive type of primitive the current vertex is part of (constant over batch)

uint TDInputPrimVertsStartIndex(); //primitives only: the index of the first vertex for the current input primitive
uint TDInputVertPrimIndex(); //vertices only: the index of the current vertex in the input primitive (goes from 0 to numVertsPerPrim - 1)
    
uint TDInputPrimsStartIndex(); //index of first input primitive in current primitive batch
uint TDPrimsStartIndex(); //index of first output primitive in current primitive batch

uint TDInputVertsStartIndex(); //index of first input vertex in current primitive batch
uint TDVertsStartIndex(); //index of first output vertex in current primitive batch

uint TDInputNumPrimsBatch(); //number of input primitives in current primitives batch
uint TDNumPrimsBatch(); //number of output primitives in current primitives batch

uint TDInputNumVertsBatch(); //number of input verts in current primitives batch
uint TDNumVertsBatch(); //number of output verts in current primitives batch
```

---

## GLSL POPs Helper functions

Common Functions on the Write a GLSL material page [Write_a_GLSL_Material#Common_Functions](https://docs.derivative.ca/Write_a_GLSL_Material#Common_Functions)

```glsl
float TDSineLookup(float coord);

mat3 TDRotateToVector(vec3 forward, vec3 up);
mat3 TDRotateOnAxis(float radians, vec3 axis)
mat3 TDCreateRotMatrix(vec3 from, vec3 to);
mat3 TDCreateTBNMatrix(vec3 normal, vec3 tangent, float handedness);

mat3 TDRotateX(float radians)
mat3 TDRotateY(float radians)
mat3 TDRotateZ(float radians)

float TDPerlinNoise(vec2 v);
float TDPerlinNoise(vec3 v);
float TDPerlinNoise(vec4 v);
float TDSimplexNoise(vec2 v);
float TDSimplexNoise(vec3 v);
float TDSimplexNoise(vec4 v);

vec3 TDHSVToRGB(vec3 c);
vec3 TDRGBToHSV(vec3 c);

vec3 TDEquirectangularToCubeMap(vec2 equiCoord);
vec2 TDCubeMapToEquirectangular(vec3 cubemapCoord);
vec2 TDCubeMapToEquirectangular(vec3 cubemapCoord, out float mipMapBias);
```

### New Functions

```glsl
float TDRemap(float val, float oldMin, float oldMax, float newMin, float newMax);
vec2 TDRemap(vec2 val, vec2 oldMin, vec2 oldMax, vec2 newMin, vec2 newMax);
vec3 TDRemap(vec3 val, vec3 oldMin, vec3 oldMax, vec3 newMin, vec3 newMax);

float TDLoop(float val, float low, float high)
float TDZigZag(float val, float low, float high)
float TDAverage(float valA, float valB)
```

---

## Accessing POPs buffers in GLSL TOP and GLSL MAT

### GLSL TOP and MAT: POP attribute access

```glsl
attribType TDBuffer_AttribName(uint elementIndex, uint arrayIndex); //access to POP attribute buffer declared on Buffer Page
TDBuffer_AttribName(uint elementIndex) = TDBuffer_AttribName(elementIndex, 0); //arrayIndex is optional, defaults to 0

const uint TDBufferLength_AttribName(); //length of buffer
const uint cTDBufferArraySize_AttribName; //constant with size of array for array attributes
```

### GLSL MAT: Input attribute access

This only allow input attribute access by vertex Index, lookups are automatically performed for point and primitive attributes. To access any element of a POP attribute, use the TDBuffer() function described above.

```glsl
attribType TDAttrib_AttribName(uint vertIndex, uint arrayIndex) //access to input geo attribute (has to be declared on Attributes page)
TDAttrib_AttribName(uint arrayIndex) = TDAttrib_AttribName(gl_VertexID, arrayIndex); //vertIndex is optional, defaults to gl_VertexID
TDAttrib_AttribName() = TDAttrib_AttribName(gl_VertexID, 0);
//for points and primitive attributes lookups are performed based on the vertex;

const uint cTDAttribArraySize_AttribName //constant with size of array for array attributes
```

---

## Glossary

- **Information associated with SOP geometry.** Points and primitives (polygons, NURBS, etc.) can have any number of attributes - position (P) is standard, and built-in optional attributes are normals (N), texture coordinates (uv), color (Cd), etc.
- **The OpenGL (pre-2022) or Vulkan (2022-) code that runs on the GPU and creates rendered images from polygons and textures.** A shader is programmed in Text DATs and referenced by a GLSL Material or a GLSL TOP. Shaders are composed of up to three parts: Vertex Shader, Pixel Shader and Compute Shader.
- **Storage** is a python dictionary in each operator, where users can store and fetch extra data.
- **The Graphics Processing Unit.** This is the high-speed, many-core processor of the graphics card/chip that takes geometry, images and data from the CPU and creates images and processed data.
- **Each SOP has a list of Points.** Each point has an XYZ 3D position value plus other optional attributes. Each polygon Primitive is defined by a vertex list, which is list of point numbers.
- **A surface type in SOPs** that includes polygon, curve (NURBS and Bezier), patch (NURBS and Bezier) and other basic shapes like sphere, tube and metaball. Points and Primitives are part of the Geometry Detail, which is a part of a SOP.
- **An Operator Family** that creates, composites and modifies images, and reads/writes images and movies to/from files and the network. TOPs run on the graphics card's GPU.
- **MATs or Materials** are an Operator Family that applies a Shader to a SOP or 3D Geometry Object for rendering textured surfaces with lighting.
- **A sequence of vertices form a Polygon in a SOP.** Each vertex is an integer index into the Point List, and each Point holds an XYZ position and attributes like Normals and Texture Coordinates.

---

Retrieved from "https://docs.derivative.ca/index.php?title=Experimental:Write_GLSL_POPs&oldid=33782"