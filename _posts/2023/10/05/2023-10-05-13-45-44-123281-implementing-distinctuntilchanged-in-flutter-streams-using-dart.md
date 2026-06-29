---
layout: post
title: "Implementing distinctUntilChanged in Flutter streams using Dart"
description: " "
date: 2023-10-05
tags: []
comments: true
share: true
---

## Introduction

When working with streams in Flutter applications, there are times when you want to ignore consecutive duplicate values emitted by a stream. The `distinctUntilChanged` operator allows you to filter out these duplicate values and only emit distinct values.

In this blog post, we will explore how to implement the `distinctUntilChanged` functionality in Flutter streams using Dart.


## Using the distinctUntilChanged operator

To implement `distinctUntilChanged` in Flutter streams, we need to create a custom stream transformer. A stream transformer is a function that takes a stream as input and returns a new transformed stream.

Here's an example of how to create a simple `distinctUntilChanged` stream transformer:

```dart
import 'dart:async';

StreamTransformer<T, T> distinctUntilChanged<T>() {
  T previousValue;
  
  return StreamTransformer<T, T>.fromHandlers(
    handleData: (value, sink) {
      if (value != previousValue) {
        sink.add(value);
        previousValue = value;
      }
    },
  );
}
```

In the above code:
- We define a generic type `T` for the stream values.
- We create a variable `previousValue` to keep track of the previously emitted value.
- We return a `StreamTransformer` that takes `T` as the input type and also returns `T` as the output type.
- Inside the `handleData` method of the transformer, we check if the incoming value is different from the previous value. If it is, we add it to the output stream and update the `previousValue`.


## Using the distinctUntilChanged transformer in a Stream

To use the `distinctUntilChanged` transformer in a stream, we need to chain it with the stream using the `transform` method. Here's an example:

```dart
import 'dart:async';


void main() {
  final stream = Stream<int>.fromIterable([1, 1, 2, 3, 3, 4, 5])
      .transform(distinctUntilChanged());

  stream.listen((value) {
    print(value); // Output: 1, 2, 3, 4, 5
  });
}
```

In the above code, we create a stream from an iterable of integers and then apply the `distinctUntilChanged` transformer using the `transform` method. Finally, we listen to the transformed stream and print the values.

The output of the above code will be:

```
1
2
3
4
5
```

As you can see, the consecutive duplicate values (`1` and `3`) are filtered out by the `distinctUntilChanged` transformer.


## Conclusion

Implementing the `distinctUntilChanged` functionality in Flutter streams using Dart is straightforward. By creating a custom stream transformer, we can easily filter out consecutive duplicate values and only emit distinct values. This can be useful in scenarios where you want to reduce the number of updates triggered by duplicate values in streams.

By using the `distinctUntilChanged` operator, you can optimize your Flutter applications by ensuring that only distinct values propagate through the stream pipeline.

*#Flutter #Dart*