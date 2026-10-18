---
layout: post
title: "webview_flutter Android File Upload - Restore HTML input with file_picker_android"
description: "Fix Android HTML file uploads in webview_flutter 4 by wiring the platform file selector, MIME filters, multi-select mode, and cancellation behavior."
date: 2026-10-19
tags: [webview_flutter, file_picker_android, Android, networking, migration]
comments: true
share: true
---
![Flutter webview_flutter Android file selector flow for HTML uploads](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?auto=format&fit=crop&w=1600&q=80)

If an Android page inside `webview_flutter` contains `<input type="file">`, add the Android platform callback and return real file paths from a picker. The current `webview_flutter` API exposes `AndroidWebViewController.setOnShowFileSelector`; without that callback, the page can render an upload button but the native chooser has no application-level result to send back. This fix is for Android WebView uploads. It is not needed for a normal Flutter form using `file_picker` directly, and it does not automatically turn `capture="camera"` into a camera workflow.

The important boundary is the WebView platform implementation, not `WebViewController` alone.

| Layer | Responsibility | Failure when omitted |
| --- | --- | --- |
| Web page | Sends accepted MIME types and single/multiple mode | Picker cannot apply the page’s constraints |
| `webview_flutter_android` | Receives `FileSelectorParams` | `<input type=file>` stays unanswered |
| `file_picker_android` | Opens Android’s picker and returns paths | WebView receives no upload URI |
| Server | Validates type and size again | A client-side filter becomes a security hole |

## Use the Android platform controller

Add the endorsed Android implementation explicitly because the code imports its API, then add the Android picker package. The versions below were visible in the official package documentation on 2026-10-19; pin versions compatible with the Flutter SDK in your app.

```yaml
dependencies:
  webview_flutter: ^4.14.1
  webview_flutter_android: ^4.14.1
  file_picker_android: ^1.0.1
```

The callback must return paths, not `XFile` objects or `File` instances. `file_picker_android` already provides the adapter expected by `webview_flutter_android`.

```dart
import 'package:file_picker_android/file_picker_android.dart';
import 'package:flutter/foundation.dart';
import 'package:webview_flutter/webview_flutter.dart';
import 'package:webview_flutter_android/webview_flutter_android.dart';

late final WebViewController controller;
final FilePickerAndroid _androidPicker = FilePickerAndroid();

Future<void> configureWebView() async {
  controller = WebViewController()
    ..setJavaScriptMode(JavaScriptMode.unrestricted)
    ..loadRequest(Uri.parse('https://example.com/upload'));

  if (defaultTargetPlatform != TargetPlatform.android) return;
  final android = controller.platform as AndroidWebViewController;

  await android.setAllowFileAccess(true);
  await android.setAllowContentAccess(true);
  await android.setOnShowFileSelector(_selectFilesForWebView);
}

Future<List<String>> _selectFilesForWebView(
  FileSelectorParams params,
) async {
  if (params.mode == FileSelectorMode.save) {
    // file_picker_android selects existing files; it is not a save dialog.
    return <String>[];
  }

  return _androidPicker.pickFile(
    openMultiple: params.mode == FileSelectorMode.openMultiple,
    acceptTypes: params.acceptTypes,
  );
}
```

![The file upload handoff from HTML to the Android platform picker](https://images.unsplash.com/photo-1555066931-4365d14bab8c?auto=format&fit=crop&w=1600&q=80)

The picture to watch is the return direction: the picker produces paths, and the WebView plugin hands those paths back to the pending HTML file input.

## Do not collapse cancellation and failure

An empty list means “no files selected” to the WebView. That is the right result for Back or a canceled picker, but it can also hide an implementation mistake. Log the request mode and accepted MIME types around the picker in debug builds, while keeping user-visible behavior quiet for cancellation.

The `acceptTypes` list comes from HTML such as `accept="image/*,.pdf"`. Preserve it instead of hard-coding `*/*`; the picker is a usability filter, not the final authorization check. The server still needs to validate content, size, and filename independently.

`FileSelectorMode.openMultiple` also deserves an explicit decision. If the web form supports multiple files, pass `true`. If the backend accepts only one upload, return the first path or reject the request at the page boundary. Silently allowing several files can make the browser-side form and API contract disagree.

`isCaptureEnabled` is only a preference from the page. The adapter above deliberately opens a document picker. If the product promises “take a photo,” create a separate camera path with `image_picker`, copy the temporary result to app-owned storage, and return the resulting path only after the capture succeeds. Do not assume that setting `capture` gives the WebView permission to use the camera.

## Release checklist

- Verify the callback is registered after the controller is created and before the page is usable.
- Test `accept="image/*"`, a PDF filter, cancel, and an invalid file type.
- Test one file and multiple files on at least one Android 13+ device.
- Confirm the upload request reaches the server with the expected multipart field name.
- Decide what `save` means; the shown adapter returns an empty result because it does not implement save dialogs.
- Test a page reload while the picker is open. Treat an abandoned selection as cancellation, not a successful upload.
- Enforce MIME sniffing, byte limits, and authentication on the server.

The practical fix is a three-part contract: `webview_flutter_android` receives the HTML request, `file_picker_android` returns Android paths, and the backend validates the uploaded bytes. A basic WebView page does not need a custom native `WebChromeClient`; the platform controller callback is the supported integration point. If the app has no embedded HTML upload form, do not add this adapter just because the package supports it.

Sources: [webview_flutter Android platform guide](https://pub.dev/packages/webview_flutter), [`AndroidWebViewController.setOnShowFileSelector`](https://pub.dev/documentation/webview_flutter_android/latest/webview_flutter_android/AndroidWebViewController-class.html), and the [`file_picker_android` WebView example](https://pub.dev/documentation/file_picker_android/latest/).
