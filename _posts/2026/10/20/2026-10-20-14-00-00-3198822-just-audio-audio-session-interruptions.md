---
layout: post
title: "just_audio Audio Session Interruptions - Prevent Flutter Playback from Going Silent After Calls"
description: "Fix just_audio playback that pauses, ducks, or goes silent after calls and navigation prompts by choosing one interruption owner and configuring audio_session after every audio plugin."
date: 2026-10-20
tags: [just_audio, audio_session, Android, iOS, performance]
comments: true
share: true
---

![just_audio audio session interruption flow for calls and navigation prompts](https://images.unsplash.com/photo-1505740420928-5e560c06d30e?w=1200&q=80)

*The key state transition is the interruption boundary, not the progress bar moving forward.*

If a Flutter podcast or audiobook app becomes silent after a phone call, navigation prompt, or another recorder starts, the safer fix is to make one layer own interruption recovery. Use `just_audio`'s automatic handling for a simple player, or set `handleInterruptions: false` and handle `audio_session` events yourself when recording, speech, or another audio plugin shares the app. Do not configure the session in several plugins and also call `play()` from multiple listeners.

## Why the progress bar can lie

`just_audio` and `audio_session` are related but have different jobs. The player controls media state; the session represents the platform's audio focus and interruption events. On iOS the session configuration is app-wide. On Android, the configuration is shared through `audio_session`, while each audio plugin still has to apply the relevant attributes.

That creates two production traps:

| Symptom | Likely boundary | Safer decision |
| --- | --- | --- |
| Playback pauses during a call and never resumes | No end-event handling, or two handlers race | Pick automatic or manual recovery |
| UI says playing but no sound is heard | Audio focus was lost while Dart state stayed true | Re-activate the session before resuming |
| Podcast becomes ducked after opening a recorder | A later plugin overwrote the shared session | Configure once after all audio plugins load |
| Audio works in a music screen but not a recording screen | Music and speech attributes are being mixed | Use a session policy per product mode |

The open `just_audio` issue [#1363](https://github.com/ryanheise/just_audio/issues/1363) is a useful warning: some Android reports describe playback stopping while the progress state continues. The issue is not proof that every silent player has the same cause, but it shows why a UI boolean is not enough to diagnose audio focus.

## Choose the interruption owner first

For a music-only app, the default player behavior is usually enough. `just_audio` can pause or duck for interruptions and may resume when the interruption ends. This is the smallest configuration:

```dart
final player = AudioPlayer(
  handleInterruptions: true,
);
```

This choice is not a good fit when the app also records, reads text aloud, or has a custom rule such as “never resume after a phone call.” In that case, disable the player's automatic policy and make the session listener the only recovery path.

## A manual policy for podcasts and mixed audio

The following service pauses during a real interruption, remembers whether playback was active, and only resumes after the platform reports that the interruption ended. It does not resume after a duck event, because ducking is not a full playback stop.

```dart
import 'dart:async';

import 'package:audio_session/audio_session.dart';
import 'package:just_audio/just_audio.dart';

class PodcastAudio {
  PodcastAudio()
      : player = AudioPlayer(handleInterruptions: false);

  final AudioPlayer player;
  StreamSubscription<AudioInterruptionEvent>? _interruptions;
  bool _wasPlayingBeforePause = false;

  Future<void> start() async {
    final session = await AudioSession.instance;
    await session.configure(AudioSessionConfiguration.speech());

    _interruptions = session.interruptionEventStream.listen((event) async {
      if (event.begin) {
        if (event.type == AudioInterruptionType.pause) {
          _wasPlayingBeforePause = player.playing;
          await player.pause();
        }
        return;
      }

      if (event.type == AudioInterruptionType.pause &&
          _wasPlayingBeforePause) {
        _wasPlayingBeforePause = false;
        if (await session.setActive(true)) {
          await player.play();
        }
      }
    });
  }

  Future<void> dispose() async {
    await _interruptions?.cancel();
    await player.dispose();
  }
}
```

The important detail is `setActive(true)` before `play()`. A completed interruption does not guarantee that the app still owns the audio focus. If activation is denied because another call or audio app is still active, leave the player paused and let the user retry.

## Configure after every plugin is loaded

The `audio_session` documentation warns that another audio plugin can overwrite the app's session settings. A recorder or text-to-speech plugin may be initialized after the player and silently change the category or audio attributes. Treat the final configuration as application startup policy, not as a side effect hidden in one widget.

```dart
Future<void> configureAudio() async {
  // Construct or initialize just_audio, recorder, and speech plugins first.
  final session = await AudioSession.instance;
  await session.configure(AudioSessionConfiguration.speech());
}
```

Use `music()` for music, `speech()` for spoken content, and a custom configuration only when the platform behavior is understood. A podcast app that uses `music()` may duck when it should pause; a recorder that uses a playback policy may compete with the microphone.

## Reproducible release checklist

- [ ] Decide whether `just_audio` or your service owns interruption handling.
- [ ] Do not register a second listener that calls `play()` after automatic handling.
- [ ] Configure `AudioSession` after recorder, speech, and background-audio plugins load.
- [ ] Test an incoming phone call, a short navigation prompt, Bluetooth disconnect, and app backgrounding.
- [ ] Compare `player.playing`, `processingState`, and interruption events in logs; do not log only the progress bar.
- [ ] On Android, repeat the test with the screen locked and with another audio app active.
- [ ] On iOS, test both a phone call and a short Siri or navigation interruption.

`just_audio` gives a good default for a single playback path. The failure-prone case is a mixed audio app where several packages believe they own the same platform session. Pick one policy, configure it at the application boundary, and treat a successful `play()` call as different from confirmed audible output.

Sources checked 2026-10-20: [`just_audio` documentation](https://pub.dev/packages/just_audio), [`audio_session` documentation](https://pub.dev/packages/audio_session), [`just_audio` source notes on interruption handling](https://github.com/ryanheise/just_audio/blob/minor/just_audio/lib/just_audio.dart), and [issue #1363](https://github.com/ryanheise/just_audio/issues/1363).
