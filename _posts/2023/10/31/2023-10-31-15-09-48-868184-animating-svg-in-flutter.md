---
layout: post
title: "Animating SVG in Flutter"
description: " "
date: 2023-10-31
tags: [references]
comments: true
share: true
---

SVG (Scalable Vector Graphics) is a widely used format for displaying vector graphics on the web. With Flutter, you can easily incorporate SVG files into your applications and animate them to create fluid and dynamic visual effects. In this blog post, we will explore how to animate SVGs in Flutter.

## Table of Contents
- [What is SVG?](#what-is-svg)
- [Adding SVG to Flutter](#adding-svg-to-flutter)
- [Animating SVG in Flutter](#animating-svg-in-flutter)
  - [Using AnimatedContainer](#using-animatedcontainer)
  - [Using Flutter Animation Framework](#using-flutter-animation-framework)
- [Conclusion](#conclusion)

## What is SVG?
SVG is an XML-based vector image format that describes two-dimensional graphics. It allows for detailed control over the appearance of the graphics and can scale without compromising on quality. SVG files can be created using graphic design software or exported from tools like Adobe Illustrator.

## Adding SVG to Flutter
To use SVG files in Flutter, you need to include the `flutter_svg` package in your project. You can add it to your `pubspec.yaml` file as follows:

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_svg: ^0.22.0
```

After adding the package, you can import it in your Dart code:

```dart
import 'package:flutter_svg/flutter_svg.dart';
```

With the `flutter_svg` package, you can easily load and display SVG files in your Flutter application.

#
### Using AnimatedContainer
The simplest way to animate an SVG in Flutter is by using `AnimatedContainer`. `AnimatedContainer` is a Flutter widget that automatically animates changes in its properties, such as width, height, color, and more. To animate an SVG, you can wrap it inside an `AnimatedContainer` and change any desired property over time.

Here's an example of animating the size of an SVG using `AnimatedContainer`:

```dart
// Import necessary packages
import 'package:flutter_svg/flutter_svg.dart';
import 'package:flutter/material.dart';

class SvgAnimationExample extends StatefulWidget {
  @override
  _SvgAnimationExampleState createState() => _SvgAnimationExampleState();
}

class _SvgAnimationExampleState extends State<SvgAnimationExample> with TickerProviderStateMixin {
  double _size = 100.0;

  void _animateSize() {
    setState(() {
      _size = _size == 100.0 ? 200.0 : 100.0;
    });
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _animateSize,
      child: AnimatedContainer(
        duration: Duration(seconds: 1),
        width: _size,
        height: _size,
        child: SvgPicture.asset(
          'assets/icon.svg', // Replace with your SVG file path
        ),
      ),
    );
  }
}

// Usage:
SvgAnimationExample()
```

In the above example, tapping on the `AnimatedContainer` triggers the animation, changing the size of the SVG from 100.0x100.0 to 200.0x200.0 and vice versa.

### Using Flutter Animation Framework
For more advanced and custom animations, you can utilize the Flutter Animation Framework. This framework provides a set of classes and widgets that allow you to create complex animations with ease.

To animate an SVG using the Flutter Animation Framework, you typically use a combination of `AnimationController`, `Tween`, and `AnimatedBuilder`. With these classes, you have full control over various aspects of the animation, such as the duration, easing curve, and target values.

Here's an example of animating the position of an SVG using the Flutter Animation Framework:

```dart
// Import necessary packages
import 'package:flutter_svg/flutter_svg.dart';
import 'package:flutter/material.dart';

class SvgAnimationExample extends StatefulWidget {
  @override
  _SvgAnimationExampleState createState() => _SvgAnimationExampleState();
}

class _SvgAnimationExampleState extends State<SvgAnimationExample> with SingleTickerProviderStateMixin {
  AnimationController _animationController;
  Animation<Offset> _animation;

  @override
  void initState() {
    _animationController = AnimationController(
      vsync: this,
      duration: Duration(seconds: 1),
    );

    _animation = Tween<Offset>(
      begin: Offset.zero,
      end: Offset(1.0, 1.0),
    ).animate(_animationController);

    _animationController.repeat(reverse: true);

    super.initState();
  }

  @override
  void dispose() {
    _animationController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return SlideTransition(
      position: _animation,
      child: SvgPicture.asset(
        'assets/icon.svg', // Replace with your SVG file path
      ),
    );
  }
}

// Usage:
SvgAnimationExample()
```

In the above example, we animate the position of the SVG using the `SlideTransition` widget and an `Animation<Offset>`. The animation repeats in a reverse manner, giving a continuous sliding effect to the SVG.

## Conclusion
Animating SVGs in Flutter opens up a range of possibilities for creating visually appealing and interactive user interfaces. Whether you choose to use the `AnimatedContainer` widget or the Flutter Animation Framework, you have the flexibility to create engaging animations that bring your SVGs to life. Experiment with different properties and animations to create unique effects in your Flutter applications.

#references #flutter #svg