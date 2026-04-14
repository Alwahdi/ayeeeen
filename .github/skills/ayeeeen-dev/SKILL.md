---
name: ayeeeen-dev
description: "Build features for Ayeeeen (عين), an AI-powered accessibility app for blind users. Use when: implementing screens, adding ML features, creating audio-first UX flows, handling gestures/haptics, integrating on-device AI models, building Arabic-first interfaces, adding TTS/STT, creating accessible components, or working on any Ayeeeen feature from design to implementation."
argument-hint: "Describe the feature or screen to build"
---

# Ayeeeen (عين) Feature Development

Build accessible, audio-first features for the Ayeeeen app — an AI-powered assistant for blind and visually impaired users.

## When to Use

- Building a new screen or feature
- Implementing on-device ML (object detection, OCR, scene description, face recognition)
- Creating gesture-based interactions (tap, long press, swipe, shake)
- Adding TTS/STT audio flows
- Integrating haptic feedback patterns
- Working on the camera pipeline
- Building Arabic-first UI components

## Core Principles (Non-Negotiable)

1. **Audio-first, not visual-first** — The screen is an input canvas (camera feed), not an output display. All output is via TTS + haptics.
2. **Zero cognitive overload** — No hunting for buttons. Entire screen = interaction surface.
3. **All models run locally** — No cloud AI, no API calls, works fully offline. Use ML Kit, TFLite, ONNX, Whisper.cpp.
4. **Arabic-first** — All TTS, prompts, and UI text in Arabic. RTL layout default.
5. **Instant feedback** — Every touch/gesture has immediate haptic + audio response. Never leave the user in silence.

## Feature Implementation Procedure

### Step 1: Define the Audio-First Contract

Before writing any code, define the feature's audio contract:

```
Feature: [name]
Trigger: [gesture — single tap / long press / swipe / etc.]
Immediate Feedback: [haptic type + sound cue]
Processing State: [what TTS says while working]
Success Output: [TTS message format in Arabic]
Error Output: [TTS message + haptic for failure]
Danger Interrupt: [does this feature need safety interrupts?]
```

### Step 2: Create the Component

Follow this component architecture:

```tsx
// features/[feature-name]/[FeatureName]Screen.tsx
import { useCallback } from 'react';
import { View, Pressable } from 'react-native';
import Animated, { useAnimatedStyle, useSharedValue, withTiming } from 'react-native-reanimated';

// Audio-first: import TTS and haptics
// import * as Speech from 'expo-speech';
// import * as Haptics from 'expo-haptics';

interface Props {
  // Typed props
}

export function FeatureNameScreen({ }: Props) {
  // State colors map to app state
  const backgroundColor = useSharedValue('#0A0A0A'); // idle

  const animatedBg = useAnimatedStyle(() => ({
    backgroundColor: withTiming(backgroundColor.value, { duration: 300 }),
  }));

  const handleTap = useCallback(() => {
    // 1. Immediate haptic + audio feedback
    // Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
    // playSound('listening-blip');

    // 2. Visual state change
    backgroundColor.value = '#3B82F6'; // listening blue

    // 3. Process (ML inference, OCR, etc.)
    // 4. Speak result via TTS
    // 5. Return to idle
    backgroundColor.value = '#0A0A0A';
  }, []);

  return (
    <Animated.View style={animatedBg} className="flex-1">
      <Pressable
        className="flex-1 items-center justify-center"
        onPress={handleTap}
        accessible
        accessibilityRole="button"
        accessibilityLabel="اضغط لوصف المشهد"
      >
        {/* Camera preview fills entire screen */}
        {/* Overlay: large state icon at 10% opacity */}
      </Pressable>
    </Animated.View>
  );
}
```

### Step 3: Implement the State Machine

Every feature follows the same state flow with corresponding colors and feedback:

| State | Background | Haptic | Audio |
|-------|-----------|--------|-------|
| **Idle** | `#0A0A0A` (black) | — | Silence |
| **Listening** | `#3B82F6` (blue) | Light tap | Short blip sound |
| **Processing** | `#F59E0B` (amber pulse) | — | Optional "جاري التحليل..." |
| **Success** | `#22C55E` (green flash) | Double tap (tick-tick) | TTS result in Arabic |
| **Error** | `#EF4444` (red flash) | Heavy thud | "عذراً، حاول مرة أخرى" |
| **Danger** | `#EF4444` (intense flash) | Continuous vibration | Interrupts everything: "انتبه!" |

### Step 4: Wire Up the Gesture Handler

Map gestures using the interaction model:

| Gesture | Action | Implementation |
|---------|--------|---------------|
| **Single Tap** | Describe scene / Read text | `onPress` on full-screen Pressable |
| **Long Press** | Toggle Auto-Mode (continuous) | `onLongPress` with state toggle |
| **Two-Finger Tap** | Repeat last phrase | Custom gesture detector |
| **Swipe Left/Right** | Switch modes | `react-native-gesture-handler` pan |
| **Shake** | Emergency stop (panic button) | Accelerometer listener via `expo-sensors` |

### Step 5: Integrate On-Device ML

Choose the right local model — see [ML Model Reference](./references/ml-models.md) for full setup.
For camera + frame processor integration, see [Camera Pipeline Reference](./references/camera-pipeline.md).

| Task | Library / Model | Notes |
|------|----------------|-------|
| Object Detection | `react-native-fast-tflite` + YOLOv8-nano | Real-time, ~30fps on modern devices |
| OCR (Arabic + English) | ML Kit Text Recognition v2 | Arabic support built-in |
| Face Detection | ML Kit Face Detection | Landmarks + classification |
| Barcode/QR | ML Kit Barcode Scanning | Instant, all formats |
| Scene Description | TFLite Florence-2 / MobileVLM | Phase 2 — requires optimization |
| Speech-to-Text | `whisper.cpp` via native module | Offline Arabic recognition |
| Text-to-Speech | `expo-speech` (native TTS) | Arabic voice, adjust rate/pitch |

### Step 6: Style with NativeWind

Follow the project's NativeWind conventions:

```tsx
// DO: Use className for all styling
<View className="flex-1 flex-col bg-[#0A0A0A] dark:bg-[#0A0A0A]">

// DO: Explicit flex direction on every container
<View className="flex-row items-center">

// DO: Cairo font family for any visible text
<Text className="font-cairo-bold text-4xl text-white">

// DON'T: No small text. If text is visible, it's 32-48px ultra-bold (for low-vision glance)
// DON'T: No StyleSheet.create unless absolutely needed for animated styles
```

### Step 7: Arabic TTS Voice Rules

See [Interaction & UX Reference](./references/interaction-ux.md) for the full voice UX guide.

Key TTS rules:
- **Never say "أرى" (I see)**. State what IS there with spatial context.
  - Bad: "أرى أمامي سيارة زرقاء"
  - Good: "سيارة زرقاء متوقفة أمامك على بعد مترين"
- **Priority stack**: Danger > Navigation > Scene > OCR
- **Concise**: Max 2 sentences per response. User can tap for more detail.
- **Spatial audio**: Pan left/right based on object position in frame.

### Step 8: Add Accessibility Props

Every interactive element MUST have:

```tsx
<Pressable
  accessible={true}
  accessibilityRole="button"
  accessibilityLabel="وصف بالعربي"     // Arabic description
  accessibilityHint="اضغط مرتين للتفعيل"  // How to activate
  accessibilityState={{ disabled: false, busy: isProcessing }}
>
```

## File Organization

```
features/
  object-detection/
    ObjectDetectionScreen.tsx    # Main screen
    useObjectDetection.ts        # ML inference hook
    ObjectDetectionOverlay.tsx   # Camera overlay (if any)
  ocr/
    OCRScreen.tsx
    useTextRecognition.ts
  navigation/
    NavigationScreen.tsx
    useObstacleDetection.ts
  scene-description/
    SceneScreen.tsx
    useSceneDescription.ts
shared/
  hooks/
    useTTS.ts                    # Text-to-speech wrapper
    useHaptics.ts                # Haptic feedback patterns
    useGestures.ts               # Gesture detection
    useCamera.ts                 # Camera pipeline
  components/
    OmniSurface.tsx              # Full-screen interactive surface
    StateOverlay.tsx             # Background color + icon overlay
    DangerAlert.tsx              # Interrupting danger warning
  constants/
    colors.ts                    # Design system colors
    haptics.ts                   # Haptic pattern definitions
    arabic.ts                    # Arabic string constants
  utils/
    spatial-audio.ts             # L/R audio panning
    model-loader.ts              # TFLite model loading utilities
```

## Checklist Before Shipping a Feature

- [ ] Audio contract defined (trigger → feedback → result)
- [ ] All output is via TTS, not visual text
- [ ] Haptic feedback on every interaction
- [ ] State machine: idle → listening → processing → success/error
- [ ] Background color changes match state
- [ ] Arabic TTS messages follow voice rules (no "أرى", spatial context)
- [ ] Danger interrupts override everything
- [ ] Works fully offline (no network calls)
- [ ] Accessibility props on all interactive elements
- [ ] NativeWind className styling (no StyleSheet.create)
- [ ] Cairo font for any visible text (32-48px only)
- [ ] Tested with TalkBack (Android) / VoiceOver (iOS)
