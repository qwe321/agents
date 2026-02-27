---
name: flutter-gpu-expert
description: Expert knowledge on using flutter_gpu library for GPU rendering in Flutter and Dart. Use when working with flutter_gpu, GPU shaders, 3D rendering, textures, mesh creation, or shader bundles in Flutter.
user-invocable: true
argument-hint: [question or task]
---

# Flutter GPU Expert Reference

You are an expert on the `flutter_gpu` package for low-level GPU rendering in Flutter. Use this reference for all flutter_gpu work.

When answering questions or implementing features, refer to the detailed API reference in `reference.md` in this skill directory.

## Quick Setup Checklist

1. **pubspec.yaml**: Add `flutter_gpu: {sdk: flutter}` and `vector_math` to deps; `flutter_gpu_shaders: ^0.3.0` and `native_assets_cli: ^0.13.0` to dev_dependencies; register `build/shaderbundles/` as asset
2. **Platform configs**: iOS `Info.plist` needs `FLTEnableFlutterGPU` key; Android needs `EnableFlutterGPU` meta-data
3. **Shader bundle manifest**: JSON file at project root mapping shader names to `.vert`/`.frag` files
4. **Build hook**: `hook/build.dart` calling `buildShaderBundleJson()` — see version-specific API below
5. **Shaders**: Vulkan-compatible GLSL — uniforms MUST be in struct blocks, vertex attributes matched by declaration order
6. **Run**: `flutter config --enable-native-assets` then `flutter run --enable-flutter-gpu`

## Build Hook Version Compatibility (CRITICAL)

`flutter_gpu_shaders` and `native_assets_cli` must use matching versions. The API changed between versions:

**flutter_gpu_shaders ^0.3.0 + native_assets_cli ^0.13.0** (RECOMMENDED):
```dart
import 'package:native_assets_cli/native_assets_cli.dart';
import 'package:flutter_gpu_shaders/build.dart';

void main(List<String> args) async {
  await build(args, (input, output) async {  // NOTE: "input" not "config"
    await buildShaderBundleJson(
      buildInput: input,       // NOTE: "buildInput" not "buildConfig"
      buildOutput: output,
      manifestFileName: 'my.shaderbundle.json',
    );
  });
}
```

**flutter_gpu_shaders ^0.1.0 + native_assets_cli ^0.7.0** (OLD, avoid):
```dart
// Uses buildConfig/BuildConfig instead of buildInput/BuildInput
// Uses BuildOutput instead of BuildOutputBuilder
// Will fail with native_assets_cli >= 0.9.0
```

**Common error**: `Type 'BuildOutput' not found` — means `flutter_gpu_shaders` version is too old for your `native_assets_cli`. Upgrade both together.

## Critical Gotchas

- `createDeviceBuffer()` and `createTexture()` return **non-nullable** types and throw on failure (no `!` needed)
- `ColorAttachment.clearValue` is `Vector4?` from `package:vector_math/vector_math.dart` (NOT `vector_math_64`)
- flutter_gpu internally imports `vector_math` (32-bit) via `as vm` — use `vector_math.dart` Vector4 for clearValue, but `vector_math_64.dart` for Matrix4 math
- `draw()` handles BOTH indexed and non-indexed — there is no separate `drawIndexed()`
- Uniform `getUniformSlot()` takes the **block name** (e.g., `'VertexUniforms'`), not member or instance name
- Vertex attributes are matched by **declaration order** in GLSL, not by name
- `Matrix4.translate()` is deprecated — use `translateByDouble()` or `translateByVector3()`
- Shader GLSL does NOT need `#version` or `layout(location=...)` — the shader compiler handles it
- `HostBuffer` is a bump allocator for per-frame transient data; `DeviceBuffer` for static data
