# Flutter GPU Complete API Reference

## Project Setup

### pubspec.yaml

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_gpu:
    sdk: flutter
  vector_math: ^2.1.4

dev_dependencies:
  flutter_gpu_shaders: ^0.3.0
  native_assets_cli: ^0.13.0

flutter:
  assets:
    - build/shaderbundles/
```

### Platform Configs

**iOS** `ios/Runner/Info.plist` — inside `<dict>`:
```xml
<key>FLTEnableFlutterGPU</key>
<true/>
```

**Android** `android/app/src/main/AndroidManifest.xml` — inside `<application>`:
```xml
<meta-data
    android:name="io.flutter.embedding.android.EnableFlutterGPU"
    android:value="true" />
```

### Enable native assets (one-time)
```
flutter config --enable-native-assets
```

### Run
```
flutter run --enable-flutter-gpu
```

---

## Shader Bundle Manifest

Place at project root, e.g., `my_renderer.shaderbundle.json`:

```json
{
    "MyVertex": {
        "type": "vertex",
        "file": "shaders/my_shader.vert"
    },
    "MyFragment": {
        "type": "fragment",
        "file": "shaders/my_shader.frag"
    }
}
```

The compiled output goes to `build/shaderbundles/<name>.shaderbundle`.

---

## Build Hook

Must be at `hook/build.dart`:

```dart
import 'package:native_assets_cli/native_assets_cli.dart';
import 'package:flutter_gpu_shaders/build.dart';

void main(List<String> args) async {
  await build(args, (input, output) async {
    await buildShaderBundleJson(
      buildInput: input,
      buildOutput: output,
      manifestFileName: 'my_renderer.shaderbundle.json',
    );
  });
}
```

**NOTE**: `flutter_gpu_shaders ^0.3.0` uses `buildInput`/`BuildInput`. Older `^0.1.0` used `buildConfig`/`BuildConfig` — these are incompatible with `native_assets_cli >= 0.9.0`.

---

## GLSL Shader Format (Vulkan-Compatible)

**No `#version` or `layout(location=...)` needed** — the shader compiler handles these.

### Vertex Shader

```glsl
uniform VertexUniforms {
  mat4 mvp;
} vertex_uniforms;

in vec3 position;
in vec2 uv;

out vec2 frag_uv;

void main() {
  gl_Position = vertex_uniforms.mvp * vec4(position, 1.0);
  frag_uv = uv;
}
```

### Fragment Shader

```glsl
uniform sampler2D base_color_texture;

in vec2 frag_uv;

out vec4 frag_color;

void main() {
  frag_color = texture(base_color_texture, frag_uv);
}
```

### Key Rules
- All uniforms MUST be in `uniform Block { ... } instance_name;` struct blocks (Vulkan requirement)
- `getUniformSlot()` uses the **block name** (e.g., `'VertexUniforms'`), not instance or member name
- Texture samplers use the sampler variable name for `getUniformSlot()`
- Vertex `in` attributes are matched by **declaration order** to interleaved vertex buffer layout

---

## Core API

### Import

```dart
import 'package:flutter_gpu/gpu.dart' as gpu;
```

### GpuContext (singleton)

```dart
gpu.gpuContext                            // the singleton context
gpu.gpuContext.defaultColorFormat          // PixelFormat
gpu.gpuContext.defaultDepthStencilFormat   // PixelFormat for depth+stencil
gpu.gpuContext.minimumUniformByteAlignment // int
```

### ShaderLibrary

```dart
final lib = gpu.ShaderLibrary.fromAsset('build/shaderbundles/my.shaderbundle')!;
final vertexShader = lib['MyVertex']!;
final fragmentShader = lib['MyFragment']!;
```

### RenderPipeline

```dart
final pipeline = gpu.gpuContext.createRenderPipeline(vertexShader, fragmentShader);
pipeline.vertexShader;    // Shader
pipeline.fragmentShader;  // Shader
```

### UniformSlot

```dart
final slot = shader.getUniformSlot('BlockName');  // uses BLOCK name, not member
```

---

## Buffers

### StorageMode

| Mode | Use |
|------|-----|
| `StorageMode.hostVisible` | CPU-writable data (vertex buffers, textures you upload to) |
| `StorageMode.devicePrivate` | GPU-only, optimal (render targets) |
| `StorageMode.deviceTransient` | Tile memory (depth/stencil, no readback) |

### DeviceBuffer (static data)

```dart
// Returns non-nullable, throws on failure
final buf = gpu.gpuContext.createDeviceBuffer(
  gpu.StorageMode.hostVisible,
  sizeInBytes,
);
buf.overwrite(float32List.buffer.asByteData());

// Or create with initial data:
final buf = gpu.gpuContext.createDeviceBufferWithCopy(byteData);
```

**API:**
- `int sizeInBytes` (final)
- `bool overwrite(ByteData sourceBytes, {int destinationOffsetInBytes = 0})`

### HostBuffer (per-frame transient data — bump allocator)

```dart
final transients = gpu.gpuContext.createHostBuffer();
final bufferView = transients.emplace(someByteData);  // returns BufferView
```

### BufferView

```dart
gpu.BufferView(deviceBuffer, offsetInBytes: 0, lengthInBytes: buf.sizeInBytes)
```

---

## Textures

### Creating Textures

```dart
// Returns non-nullable, throws on failure
final texture = gpu.gpuContext.createTexture(
  gpu.StorageMode.devicePrivate,  // or hostVisible, deviceTransient
  width, height,
  format: gpu.PixelFormat.r8g8b8a8UNormInt,  // default
  enableRenderTargetUsage: true,   // default true
  enableShaderReadUsage: true,     // default true
  enableShaderWriteUsage: false,   // default false
  coordinateSystem: gpu.TextureCoordinateSystem.renderToTexture,
);
```

### Uploading Image Data

```dart
// For PNG/image loading:
final imageData = await rootBundle.load('assets/texture.png');
final codec = await ui.instantiateImageCodec(imageData.buffer.asUint8List());
final frame = await codec.getNextFrame();
final image = frame.image;
final byteData = await image.toByteData(format: ui.ImageByteFormat.rawRgba);

final tex = gpu.gpuContext.createTexture(
  gpu.StorageMode.hostVisible, image.width, image.height,
  enableShaderReadUsage: true,
  enableRenderTargetUsage: false,
);
tex.overwrite(byteData!);
image.dispose();
```

### Getting ui.Image from Texture

```dart
final uiImage = texture.asImage();  // dart:ui Image for Flutter Canvas
```

---

## Rendering

### CommandBuffer + RenderPass

```dart
final commandBuffer = gpu.gpuContext.createCommandBuffer();

final renderTarget = gpu.RenderTarget.singleColor(
  gpu.ColorAttachment(
    texture: colorTexture,
    clearValue: Vector4(r, g, b, a),  // from package:vector_math/vector_math.dart
  ),
  depthStencilAttachment: gpu.DepthStencilAttachment(
    texture: depthTexture,
    depthClearValue: 1.0,
  ),
);

final pass = commandBuffer.createRenderPass(renderTarget);
```

**IMPORTANT:** `ColorAttachment.clearValue` is `Vector4?` from `package:vector_math/vector_math.dart` (NOT `vector_math_64`). flutter_gpu imports `vector_math` as `vm` internally.

### ColorAttachment

```dart
gpu.ColorAttachment({
  loadAction = LoadAction.clear,
  storeAction = StoreAction.store,
  Vector4? clearValue,        // from vector_math (32-bit), NOT vector_math_64
  required Texture texture,
  Texture? resolveTexture,
})
```

### DepthStencilAttachment

```dart
gpu.DepthStencilAttachment({
  depthLoadAction = LoadAction.clear,
  depthStoreAction = StoreAction.dontCare,
  double depthClearValue = 0.0,   // use 1.0 for standard depth testing
  stencilLoadAction = LoadAction.clear,
  stencilStoreAction = StoreAction.dontCare,
  int stencilClearValue = 0,
  required Texture texture,
})
```

### RenderPass Operations

```dart
// Pipeline
pass.bindPipeline(pipeline);

// Vertex/Index data
pass.bindVertexBuffer(bufferView, vertexCount);
pass.bindIndexBuffer(bufferView, gpu.IndexType.int16, indexCount);

// Uniforms and textures
pass.bindUniform(uniformSlot, bufferView);
pass.bindTexture(uniformSlot, texture, sampler: gpu.SamplerOptions(...));

// Depth/stencil
pass.setDepthWriteEnable(true);
pass.setDepthCompareOperation(gpu.CompareFunction.less);

// Blend
pass.setColorBlendEnable(true);
pass.setColorBlendEquation(gpu.ColorBlendEquation(
  colorBlendOperation: gpu.BlendOperation.add,
  sourceColorBlendFactor: gpu.BlendFactor.one,
  destinationColorBlendFactor: gpu.BlendFactor.oneMinusSourceAlpha,
  alphaBlendOperation: gpu.BlendOperation.add,
  sourceAlphaBlendFactor: gpu.BlendFactor.one,
  destinationAlphaBlendFactor: gpu.BlendFactor.oneMinusSourceAlpha,
));

// Other state
pass.setCullMode(gpu.CullMode.backFace);
pass.setWindingOrder(gpu.WindingOrder.clockwise);

// Draw (handles both indexed and non-indexed automatically)
pass.draw();

// For multiple objects: clear and rebind
pass.clearBindings();
```

### SamplerOptions

```dart
gpu.SamplerOptions(
  minFilter: gpu.MinMagFilter.linear,   // or .nearest
  magFilter: gpu.MinMagFilter.linear,
  mipFilter: gpu.MipFilter.nearest,
  widthAddressMode: gpu.SamplerAddressMode.clampToEdge,
  heightAddressMode: gpu.SamplerAddressMode.clampToEdge,
)
```

### Submit and Get Image

```dart
commandBuffer.submit();
final image = colorTexture.asImage();  // dart:ui Image
canvas.drawImage(image, Offset.zero, Paint());
```

---

## Complete Rendering Pattern

```dart
import 'dart:ui' as ui;
import 'dart:typed_data';
import 'package:flutter_gpu/gpu.dart' as gpu;
import 'package:vector_math/vector_math.dart' show Vector4;
import 'package:vector_math/vector_math_64.dart' show Matrix4, Vector3, makePerspectiveMatrix;

ui.Image renderFrame(int width, int height, double rotation) {
  // 1. Color + depth textures
  final color = gpu.gpuContext.createTexture(
    gpu.StorageMode.devicePrivate, width, height,
    enableRenderTargetUsage: true, enableShaderReadUsage: true,
    coordinateSystem: gpu.TextureCoordinateSystem.renderToTexture,
  );
  final depth = gpu.gpuContext.createTexture(
    gpu.StorageMode.deviceTransient, width, height,
    format: gpu.gpuContext.defaultDepthStencilFormat,
    enableRenderTargetUsage: true,
    coordinateSystem: gpu.TextureCoordinateSystem.renderToTexture,
  );

  // 2. Command buffer + render pass
  final cmd = gpu.gpuContext.createCommandBuffer();
  final pass = cmd.createRenderPass(gpu.RenderTarget.singleColor(
    gpu.ColorAttachment(texture: color, clearValue: Vector4(0, 0, 0, 1)),
    depthStencilAttachment: gpu.DepthStencilAttachment(texture: depth, depthClearValue: 1.0),
  ));

  // 3. Pipeline + state
  pass.bindPipeline(pipeline);
  pass.setDepthWriteEnable(true);
  pass.setDepthCompareOperation(gpu.CompareFunction.less);

  // 4. Bind data
  pass.bindVertexBuffer(vertexBufView, vertexCount);
  pass.bindIndexBuffer(indexBufView, gpu.IndexType.int16, indexCount);

  final transients = gpu.gpuContext.createHostBuffer();
  final mvpView = transients.emplace(
    Float32List.fromList(mvpMatrix.storage).buffer.asByteData(),
  );
  pass.bindUniform(vertexShader.getUniformSlot('VertexUniforms'), mvpView);
  pass.bindTexture(fragmentShader.getUniformSlot('base_color_texture'), texture);

  // 5. Draw + submit
  pass.draw();
  cmd.submit();
  return color.asImage();
}
```

---

## OBJ Mesh Loading Pattern

For loading OBJ files into flutter_gpu:

1. Parse `v` (positions), `vt` (UVs), `f` (faces) from OBJ text
2. OBJ faces use format `v/vt/vn` — indices are 1-based
3. Deduplicate vertices by `(posIdx, uvIdx)` pair → interleaved `[px, py, pz, u, v, ...]`
4. Triangulate quads: ABCD → ABC + ACD (fan from first vertex)
5. Upload to `DeviceBuffer` with `StorageMode.hostVisible` (static, once)
6. Bind via `BufferView` each frame

---

## Widget Integration Pattern

Use `CustomPaint` + `SingleTickerProviderStateMixin`:

```dart
class GpuWidget extends StatefulWidget {
  final bool isActive;
  // ...
}

class _GpuWidgetState extends State<GpuWidget>
    with SingleTickerProviderStateMixin {
  late Ticker _ticker;
  double _rotation = 0.0;
  ui.Image? _image;

  void _onTick(Duration elapsed) {
    // Accumulate rotation, call renderer, setState with new image
    final image = renderer.render(w, h, _rotation, mesh);
    setState(() {
      _image?.dispose();  // dispose previous frame
      _image = image;
    });
  }

  // Use CustomPaint with drawImageRect to display the GPU-rendered image
}
```

Key: Always `dispose()` previous `ui.Image` before replacing it.
