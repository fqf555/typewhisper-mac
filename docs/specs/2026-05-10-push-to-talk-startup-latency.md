# Push-to-Talk Startup Latency

## Summary

Push-to-talk feedback now appears immediately when a recording request is accepted, instead of waiting for microphone hardware startup to finish.

The implementation splits recording startup into two phases:
- immediate user feedback, represented by `DictationViewModel.State.starting`
- audio readiness, represented by the existing transition to `DictationViewModel.State.recording`

This keeps the slow and failure-prone parts of `AVAudioEngine` startup off the hotkey feedback path while preserving the existing semantics that `.recording` means audio capture has actually started.

## Problem

Before this change, pressing the push-to-talk hotkey could feel delayed by roughly a second. The start sound and recording indicator were both gated on synchronous audio startup, so any delay in route activation, Bluetooth stabilization, input format settling, or `AVAudioEngine.start()` delayed user-visible confirmation.

That was especially noticeable because push-to-talk needs immediate feedback: the user must know whether the hotkey was accepted before they start speaking.

## Root Cause

`DictationViewModel.startRecording(...)` previously performed `AudioRecordingService.startRecording(...)` before it played the recording start sound or set the visible recording state.

`AudioRecordingService.startRecording(...)` can legitimately block while it:
- validates microphone availability
- activates the selected audio route
- waits for Bluetooth route stabilization
- creates and configures `AVAudioEngine`
- waits for a settled input format
- installs the input tap and converter
- starts the engine
- waits for initial input readiness on selected routes
- runs retry and recovery logic for transient startup failures

Those operations are necessary for reliable capture, but they should not be on the immediate UI feedback path.

## Design

Recording startup is now explicitly modeled as:

1. Request accepted: cheap preconditions pass, UI changes to `.starting`, the start sound plays unless intentionally skipped, and media pause or audio ducking begins.
2. Audio startup: `AudioRecordingService.startRecording(...)` runs on a dedicated serial startup queue.
3. Startup completes: the main actor verifies that the startup token is still current, then transitions to `.recording` and starts the existing recording workflow.
4. Startup is cancelled or stale: if the user releases push-to-talk or cancels before audio is ready, the token is marked cancelled. If audio startup later succeeds, the just-started audio is stopped immediately with `.immediate` and the session remains failed or discarded.

This keeps the indicator responsive without pretending audio samples are available before the engine is ready.

## Implemented Changes

`TypeWhisper/ViewModels/DictationViewModel.swift`:
- Added `State.starting`.
- Added a dedicated `audioStartupQueue` for serialized startup work.
- Added startup tokens and cancellation tracking so late audio startup completions cannot resurrect cancelled recordings.
- Treats `.starting` as active recording activity for public state queries and UI bindings.
- Moves sound feedback, indicator state, media pause, and audio ducking before synchronous audio startup.
- Logs `requestToFeedbackMs` separately from existing audio startup timing.
- Moves context capture, workflow matching, timer start, live streaming, `recordingStarted` events, and deferred metadata capture to the successful audio-ready phase.
- Handles push-to-talk release, Escape cancel, and interruption while startup is still pending.

`TypeWhisper/Services/AudioRecordingService.swift`:
- Keeps `AudioRecordingService.startRecording(...)` synchronous internally, but makes `@Published` updates safe when startup is called from the background startup queue.
- Publishes `isRecording` and `recoveryError` updates on the main thread when needed.

Indicator and menu UI:
- `NotchIndicatorPanel`, `OverlayIndicatorPanel`, and `MinimalIndicatorPanel` show during `.starting`.
- `NotchIndicatorView`, `OverlayIndicatorView`, `MinimalIndicatorView`, `SharedIndicatorComponents`, `MenuBarView`, and `SetupWizardView` treat `.starting` as active recording activity.
- Accessibility labels distinguish startup with “Starting recording” where the UI has a state-specific label.

Tests:
- Updated API recording tests to account for immediate feedback before audio startup completes.
- Added coverage for release during startup, verifying the session fails cleanly and any late-started audio is stopped immediately.
- Added a wait helper for tests that need to observe the eventual `.recording` state after asynchronous startup.

## Behavior

Expected push-to-talk behavior after this change:
- Pressing the hotkey shows the configured indicator immediately.
- The start sound plays immediately for non-Bluetooth input devices.
- Bluetooth input still skips the start sound as before.
- Audio startup latency can still exist, but it no longer delays visible feedback.
- The recording timer starts when audio capture is actually ready.
- Releasing the hotkey before audio is ready fails the session as too short and prevents transcription.
- If audio startup completes after cancellation, the audio service is stopped immediately and the UI does not move to `.recording`.
- Startup failures restore media side effects and surface the same mapped errors as before.

## Metrics

The implementation keeps the existing startup metrics and adds a separate feedback metric:
- `requestToFeedbackMs`: request timestamp to visible or audible feedback
- `requestToAudioStartMs`: request timestamp to audio startup completion
- `audioStartMs`: duration of audio startup work
- `totalStartMs`: total work before the app is fully in `.recording`

The important product result is that `requestToFeedbackMs` should stay low even when `requestToAudioStartMs` is high because of route or hardware startup.

## Verification

Automated verification used a targeted macOS test run:

```bash
xcodebuild -project TypeWhisper.xcodeproj -scheme TypeWhisper -configuration Debug -derivedDataPath .build/DerivedDataTests test -only-testing:TypeWhisperTests/APIRouterAndHandlersTests/testApiStartRecordingCapturesSelectedTextAfterActiveAppContext -only-testing:TypeWhisperTests/APIRouterAndHandlersTests/testPushToTalkInterruptionDiscardStopsImmediatelyAndMarksSessionFailedByDefault -only-testing:TypeWhisperTests/APIRouterAndHandlersTests/testPushToTalkReleaseDuringStartupFailsSessionAndStopsStartedAudio -only-testing:TypeWhisperTests/APIRouterAndHandlersTests/testApiStartRecording_pausesMediaBeforeAudioStart -only-testing:TypeWhisperTests/APIRouterAndHandlersTests/testApiStartRecordingFailureKeepsImmediateFeedbackAndSkipsContextCapture -only-testing:TypeWhisperTests/APIRouterAndHandlersTests/testApiStartRecording_playsStartSoundBeforeAudioStart
```

The selected tests passed with 0 failures.

A Release build was also installed locally and manually smoke-tested. Push-to-talk feedback appeared immediately, and normal recording continued to work after audio startup completed.

## Notes

The change intentionally preserves the existing distinction between “recording requested” and “audio capture started”. `.starting` is the request-accepted UI state; `.recording` remains the audio-ready state.

Future improvements can use the new `requestToFeedbackMs` and `requestToAudioStartMs` split to identify slow device routes without conflating hardware startup time with perceived UI responsiveness.
