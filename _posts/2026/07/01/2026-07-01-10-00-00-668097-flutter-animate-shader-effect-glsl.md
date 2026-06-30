---
layout: post
title: "flutter_animate ShaderEffect — GLSL Fragment Shader Animations"
description: "Use flutter_animate ShaderEffect with GLSL fragment shaders to animate shimmer, reveal, and distortion effects while keeping widget code readable."
date: 2026-07-01
tags: [flutter_animate, animation, performance, FlutterWeb]
comments: true
share: true
---

![Flutter shader animation shown on a mobile screen](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

Short answer: `ShaderEffect` lets flutter_animate drive a GLSL fragment shader with the same timeline syntax you already use for fade, scale, slide, and blur. The trick is loading the shader once, passing the `FragmentShader` into `.shader()`, and keeping uniforms boring.

I used to reach for `CustomPainter` whenever a design asked for a wipe, glow, or liquid reveal. That works, but the code grows fast. With `ShaderEffect`, the widget tree stays normal and the pixel math moves into a `.frag` file where it belongs.

## When ShaderEffect Makes Sense

Use `ShaderEffect` when the animation changes pixels rather than layout.

Good fits:

- shimmer that bends around text or icons
- circular reveal masks
- heat haze, water ripple, or glitch distortion
- animated gradient lighting on a card
- image-aware effects that need the rendered widget as a texture

Bad fits:

- moving a widget from A to B
- expanding a panel height
- sequencing route transitions
- simple opacity or scale feedback

For those, normal flutter_animate effects are easier to debug. I would not put a shader on a button press just because it looks clever. The moment a designer asks for "the pixels to ripple", then shader work starts paying rent.

## Add The Shader Asset

Flutter compiles `.frag` files when they are listed under `flutter.shaders` in `pubspec.yaml`. I tested this with Flutter 3.44 and `flutter_animate` 4.5.2.

```yaml
flutter:
  shaders:
    - shaders/reveal_wipe.frag
```

The annoying part: a missing shader path does not fail where your widget is built. It fails when `FragmentProgram.fromAsset()` runs. I lost time staring at the animation chain before noticing the asset path typo.

## A Minimal Reveal Shader

Here is a small shader that reads the rendered widget as `image`, then reveals it from left to right. `flutter_animate` can update the first uniforms for you: width, height, animation value, and the widget snapshot.

```glsl
#include <flutter/runtime_effect.glsl>

uniform vec2 size;
uniform float value;
uniform sampler2D image;

out vec4 fragColor;

void main() {
  vec2 uv = FlutterFragCoord().xy / size;
  vec4 color = texture(image, uv);

  float edge = smoothstep(value - 0.08, value, uv.x);
  fragColor = color * edge;
}
```

`value` moves from `0.0` to `1.0`. At the start almost nothing is visible. At the end the original widget is fully visible. `smoothstep` gives the wipe a soft edge; without it, the reveal looks like a hard crop.

## Load The FragmentShader Once

Do not load the shader inside `build()`. Load the `FragmentProgram` in `initState()`, create a `FragmentShader`, then rebuild when it is ready.

```dart
import 'dart:ui';

import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class ShaderRevealCard extends StatefulWidget {
  const ShaderRevealCard({super.key});

  @override
  State<ShaderRevealCard> createState() => _ShaderRevealCardState();
}

class _ShaderRevealCardState extends State<ShaderRevealCard> {
  FragmentShader? _shader;

  @override
  void initState() {
    super.initState();
    _loadShader();
  }

  Future<void> _loadShader() async {
    final program = await FragmentProgram.fromAsset('shaders/reveal_wipe.frag');
    if (!mounted) return;
    setState(() => _shader = program.fragmentShader());
  }

  @override
  Widget build(BuildContext context) {
    final card = Container(
      width: 280,
      padding: const EdgeInsets.all(20),
      decoration: BoxDecoration(
        color: const Color(0xFF1E293B),
        borderRadius: BorderRadius.circular(8),
      ),
      child: const Text(
        'Shader reveal',
        style: TextStyle(color: Colors.white, fontSize: 24),
      ),
    );

    final shader = _shader;
    if (shader == null) return Opacity(opacity: 0, child: card);

    return card.animate().shader(
      shader: shader,
      duration: 900.ms,
      curve: Curves.easeOutCubic,
    );
  }
}
```

That is the smallest useful version. No manual controller, no `CustomPainter`, no `AnimatedBuilder`. The widget is rendered to an image, the shader samples that image, and flutter_animate updates the animation value.

![Abstract code and graphics rendering workflow](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

## What updateUniforms Actually Sends

`ShaderEffect` has an `update` callback, but the default behavior already calls the same standardized uniform helper. The order matters:

```text
float 0: width
float 1: height
float 2: value
image sampler 0: rendered widget image
```

That maps cleanly to this GLSL header:

```glsl
uniform vec2 size;
uniform float value;
uniform sampler2D image;
```

Here is the catch: `sampler2D` does not consume a float slot. If you add a custom float, it starts after `value`, not after `image`.

This shader adds a `strength` uniform at float slot 3:

```glsl
#include <flutter/runtime_effect.glsl>

uniform vec2 size;
uniform float value;
uniform sampler2D image;
uniform float strength;

out vec4 fragColor;

void main() {
  vec2 uv = FlutterFragCoord().xy / size;
  float wave = sin((uv.y * 24.0) + (value * 6.28318)) * strength;
  vec4 color = texture(image, vec2(uv.x + wave, uv.y));
  fragColor = color;
}
```

Pass that extra value from Dart with `update`:

```dart
card.animate().shader(
  shader: shader,
  duration: 1200.ms,
  curve: Curves.linear,
  update: (details) {
    details.updateUniforms(floats: const [0.018]);
    return null;
  },
);
```

I prefer `details.updateUniforms()` over raw `shader.setFloat()` until the shader stabilizes. It avoids the "why is my width becoming time?" class of mistakes. Once the effect is locked, direct slots are fine if you need every tiny bit of control.

## Foreground, Background, Or Replace?

`ShaderLayer.replace` is the default, and it is the one I use most. The shader output becomes the visual result.

```dart
card.animate().shader(
  shader: shader,
  layer: ShaderLayer.replace,
  duration: 900.ms,
);
```

`ShaderLayer.foreground` draws the shader over the widget. That is useful for sparkles, scan lines, or a light sweep.

```dart
card.animate().shader(
  shader: shader,
  layer: ShaderLayer.foreground,
  duration: 700.ms,
);
```

`ShaderLayer.background` draws behind the widget. I rarely use it because a decorated `Container` is usually clearer, but it can work for animated halos.

## The overflow Trap

Shaders are clipped to the target bounds unless you give them overflow. A blur or glow that samples outside the widget can look chopped off at the edges.

```dart
card.animate().shader(
  shader: shader,
  duration: 900.ms,
  overflow: const EdgeInsets.all(24),
);
```

I hit this with a glow shader and blamed the shader math for twenty minutes. The math was fine. The pixels were simply clipped.

The performance angle is real, though. More overflow means a larger offscreen area. On a cheap Android device, a 24px glow around one card was fine; the same glow on a scrolling list of 40 cards was not. Tie this back to the profiling pattern from {% post_url 2026-06-26-20-00-00-495267-flutter-animate-performance-profiling-low-end-devices %}: test it on the slowest device you support, not the laptop simulator.

## Why FlutterFragCoord Beats gl_FragCoord

Use `FlutterFragCoord()` for local coordinates. It gives you the position inside the widget being shaded, which is what you usually want for UI effects.

```glsl
vec2 uv = FlutterFragCoord().xy / size;
```

`gl_FragCoord` can behave differently across rendering backends because it is screen-space oriented. That difference is exactly the kind of bug that appears only after you send a TestFlight build.

## A Practical Shimmer Sweep

This is the shader I would actually ship for a card highlight. It keeps the original widget visible, then adds a diagonal white sweep.

```glsl
#include <flutter/runtime_effect.glsl>

uniform vec2 size;
uniform float value;
uniform sampler2D image;

out vec4 fragColor;

void main() {
  vec2 uv = FlutterFragCoord().xy / size;
  vec4 base = texture(image, uv);

  float sweep = uv.x + uv.y * 0.35;
  float band = smoothstep(value - 0.12, value, sweep) -
               smoothstep(value, value + 0.12, sweep);

  vec3 lit = mix(base.rgb, vec3(1.0), band * 0.35);
  fragColor = vec4(lit, base.a);
}
```

The Dart side can loop it:

```dart
premiumCard.animate(
  onPlay: (controller) => controller.repeat(),
).shader(
  shader: shader,
  duration: 1800.ms,
  curve: Curves.linear,
  layer: ShaderLayer.replace,
);
```

For a one-shot loading-to-ready transition, I would not loop it. Pair the shimmer with a `CallbackEffect` or a target-driven state change instead. The previous `CustomEffect` article has the same mental model for keeping visual math local: {% post_url 2026-06-23-14-00-00-359472-flutter-animate-custom-effect-custom-painter %}.

## Debugging Checklist

- Black output usually means one of the uniforms is unset or the shader returned alpha 0.
- A shader that works in Canvas but not `ShaderEffect` often has uniform order wrong.
- A chopped blur or glow probably needs `overflow`.
- A shader that reloads every rebuild will stutter; load it once.
- A looped distortion should usually use `Curves.linear`.
- Web needs real device/browser testing. Shader support has improved, but browser GPU paths still vary.

ShaderEffect is worth using when the effect is genuinely pixel-based. Keep the shader tiny, keep the Dart side boring, and profile before putting it in a long list. If the animation can be expressed with `.fade()`, `.scale()`, `.move()`, or `.custom()`, use those first.

---

Next up: `ToggleEffect` in flutter_animate — alternating widget states on each animation cycle without resetting back to the beginning.

**References**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [ShaderEffect API reference](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/ShaderEffect-class.html)
- [ShaderUpdateDetails updateUniforms](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/ShaderUpdateDetails/updateUniforms.html)
- [Flutter fragment shaders guide](https://docs.flutter.dev/ui/design/graphics/fragment-shaders)
