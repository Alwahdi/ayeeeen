# Camera Pipeline Reference

## Architecture Overview

```
┌─────────────────────────────────────────┐
│          react-native-vision-camera      │
│             (Camera Preview)             │
│                                          │
│  ┌─────────────────────────────────┐     │
│  │     Frame Processor (Worklet)   │     │
│  │                                 │     │
│  │  frame → resize → model input   │     │
│  │           ↓                     │     │
│  │  ┌─────────────────────┐       │     │
│  │  │  react-native-fast  │       │     │
│  │  │  -tflite inference  │       │     │
│  │  └─────────┬───────────┘       │     │
│  │            ↓                    │     │
│  │   detections[] / text / faces   │     │
│  └─────────────┬───────────────────┘     │
│                ↓                          │
│  ┌─────────────────────────────────┐     │
│  │   Results Processing (JS)       │     │
│  │   - Spatial mapping             │     │
│  │   - Danger detection            │     │
│  │   - TTS queue management        │     │
│  └─────────────────────────────────┘     │
└─────────────────────────────────────────┘
```

## Setup

### Install Dependencies

```bash
npm install react-native-vision-camera react-native-worklets-core
npm install react-native-fast-tflite
```

### Permissions in app.json

```json
{
  "expo": {
    "plugins": [
      [
        "react-native-vision-camera",
        {
          "cameraPermissionText": "عين يحتاج الكاميرا لوصف محيطك",
          "enableMicrophonePermission": true,
          "microphonePermissionText": "عين يحتاج الميكروفون للأوامر الصوتية"
        }
      ]
    ]
  }
}
```

### Camera Setup Hook

```ts
// shared/hooks/useCamera.ts
import { useCallback, useRef } from 'react';
import {
  Camera,
  useCameraDevice,
  useCameraPermission,
  useFrameProcessor,
} from 'react-native-vision-camera';
import { useSharedValue } from 'react-native-reanimated';

export function useCamera() {
  const device = useCameraDevice('back');
  const { hasPermission, requestPermission } = useCameraPermission();
  const cameraRef = useRef<Camera>(null);

  // Use shared value to pass detection results from worklet → JS
  const latestDetections = useSharedValue<Detection[]>([]);

  return {
    device,
    hasPermission,
    requestPermission,
    cameraRef,
    latestDetections,
  };
}
```

## Frame Processor Pattern

Frame processors run on a separate worklet thread — they don't block the UI.

```ts
// features/object-detection/useObjectDetectionProcessor.ts
import { useFrameProcessor } from 'react-native-vision-camera';
import { useTensorflowModel } from 'react-native-fast-tflite';
import { useSharedValue } from 'react-native-reanimated';
import { Worklets } from 'react-native-worklets-core';

export function useObjectDetectionProcessor() {
  const model = useTensorflowModel(require('../../assets/models/yolov8n.tflite'));
  const detections = useSharedValue<Detection[]>([]);

  // This callback runs on JS thread when results are ready
  const onDetectionResults = Worklets.createRunOnJS((results: Detection[]) => {
    // Process results: danger check, TTS queue, haptics
    handleDetections(results);
  });

  const frameProcessor = useFrameProcessor((frame) => {
    'worklet';

    if (model.state !== 'loaded') return;

    // Run inference on the frame
    const output = model.model.runSync([frame]);

    // Parse YOLO output into detections
    const results = parseYOLOOutputWorklet(output, frame.width, frame.height);

    // Send results to JS thread
    onDetectionResults(results);
  }, [model, onDetectionResults]);

  return { frameProcessor, detections };
}
```

### Camera Component Integration

```tsx
// features/object-detection/ObjectDetectionScreen.tsx
import { Camera } from 'react-native-vision-camera';
import { View, Pressable } from 'react-native';
import Animated from 'react-native-reanimated';

export function ObjectDetectionScreen() {
  const { device, hasPermission, cameraRef } = useCamera();
  const { frameProcessor } = useObjectDetectionProcessor();
  const { speak } = useTTS();
  const backgroundColor = useSharedValue('#0A0A0A');

  if (!device || !hasPermission) {
    return <PermissionRequest />;
  }

  return (
    <Animated.View style={useAnimatedStyle(() => ({
      flex: 1,
      backgroundColor: backgroundColor.value,
    }))}>
      {/* Camera fills entire screen */}
      <Camera
        ref={cameraRef}
        style={{ position: 'absolute', top: 0, left: 0, right: 0, bottom: 0 }}
        device={device}
        isActive={true}
        frameProcessor={frameProcessor}
        pixelFormat="rgb"
      />

      {/* Invisible touch surface on top of camera */}
      <Pressable
        className="flex-1"
        onPress={() => {
          // Trigger single-shot description
          backgroundColor.value = '#3B82F6';
        }}
        onLongPress={() => {
          // Toggle continuous auto-mode
        }}
        accessible
        accessibilityLabel="اضغط لوصف المشهد"
      />

      {/* State overlay: large icon + status text */}
      <StateOverlay state={currentState} />
    </Animated.View>
  );
}
```

## Frame Processing Strategies

### Continuous Mode (Auto-Scan)

Process every Nth frame to balance performance and responsiveness:

```ts
const frameCount = useSharedValue(0);
const PROCESS_EVERY_N = 3; // Process every 3rd frame (~10fps at 30fps camera)

const frameProcessor = useFrameProcessor((frame) => {
  'worklet';
  frameCount.value += 1;
  if (frameCount.value % PROCESS_EVERY_N !== 0) return;

  // Run detection...
}, []);
```

### Single-Shot Mode (Tap to Describe)

Capture a single frame on demand:

```ts
const captureAndDescribe = useCallback(async () => {
  if (!cameraRef.current) return;

  // Take photo
  const photo = await cameraRef.current.takePhoto({
    qualityPrioritization: 'speed',
  });

  // Run OCR on the captured image
  const text = await TextRecognition.recognize(photo.path);

  // Run object detection
  const detections = await runDetectionOnImage(photo.path);

  // Compose and speak result
  speak(composeDescription(detections, text));
}, []);
```

## Danger Detection Pipeline

Runs on EVERY frame in parallel with regular detection. Has highest priority.

```ts
// shared/utils/danger-detection.ts

interface DangerResult {
  type: 'stairs' | 'moving_object' | 'edge' | 'obstacle';
  severity: 'warning' | 'critical';
  direction: string;
  distance?: number;
}

export function checkForDangers(detections: Detection[]): DangerResult | null {
  // Priority-ordered danger checks:

  // 1. Moving objects (cars, bikes, people running toward user)
  const movingThreats = detections.filter(d =>
    ['car', 'bicycle', 'motorcycle', 'bus', 'truck'].includes(d.label)
    && d.confidence > 0.6
    && d.bbox.area > 0.15 // Takes up >15% of frame = close
  );
  if (movingThreats.length > 0) {
    return {
      type: 'moving_object',
      severity: 'critical',
      direction: getDirection(movingThreats[0].bbox),
    };
  }

  // 2. Stairs / steps
  const stairs = detections.find(d => d.label === 'stairs' && d.confidence > 0.5);
  if (stairs) {
    return {
      type: 'stairs',
      severity: 'critical',
      direction: getDirection(stairs.bbox),
    };
  }

  // 3. Close obstacles (anything large and directly ahead)
  const closeObstacles = detections.filter(d =>
    d.bbox.area > 0.3 // Takes up >30% of frame
    && isDirectlyAhead(d.bbox)
  );
  if (closeObstacles.length > 0) {
    return {
      type: 'obstacle',
      severity: 'warning',
      direction: 'ahead',
    };
  }

  return null;
}
```

### Danger Response (Interrupts Everything)

```ts
async function handleDanger(danger: DangerResult) {
  // 1. IMMEDIATELY stop any current speech
  Speech.stop();

  // 2. Heavy haptic warning
  await hapticPatterns.danger();

  // 3. Play alert sound (sharp, attention-grabbing)
  // await playAlertSound();

  // 4. Speak danger warning
  const message = dangerTemplates[danger.type](danger.direction);
  Speech.speak(message, {
    language: 'ar-SA',
    rate: 1.1, // Slightly faster for urgency
  });

  // 5. Flash screen red
  backgroundColor.value = withRepeat(
    withSequence(
      withTiming('#EF4444', { duration: 200 }),
      withTiming('#0A0A0A', { duration: 200 }),
    ),
    3,
  );
}
```

## TTS Queue Manager

Manages what gets spoken and in what order, with priority interrupts:

```ts
// shared/utils/tts-queue.ts
import * as Speech from 'expo-speech';

type Priority = 0 | 1 | 2 | 3;
// 0 = Danger (interrupts everything)
// 1 = Navigation
// 2 = Scene description
// 3 = OCR / General

interface TTSItem {
  text: string;
  priority: Priority;
}

class TTSQueue {
  private queue: TTSItem[] = [];
  private isSpeaking = false;

  enqueue(item: TTSItem) {
    if (item.priority === 0) {
      // Danger: interrupt immediately
      Speech.stop();
      this.queue = [item]; // Clear queue, only danger
      this.processNext();
      return;
    }

    // Insert by priority (lower number = higher priority)
    const insertIndex = this.queue.findIndex(q => q.priority > item.priority);
    if (insertIndex === -1) {
      this.queue.push(item);
    } else {
      this.queue.splice(insertIndex, 0, item);
    }

    if (!this.isSpeaking) this.processNext();
  }

  private processNext() {
    if (this.queue.length === 0) {
      this.isSpeaking = false;
      return;
    }

    this.isSpeaking = true;
    const item = this.queue.shift()!;

    Speech.speak(item.text, {
      language: 'ar-SA',
      rate: 0.9,
      onDone: () => this.processNext(),
      onError: () => this.processNext(),
    });
  }

  clear() {
    Speech.stop();
    this.queue = [];
    this.isSpeaking = false;
  }
}

export const ttsQueue = new TTSQueue();
```

## Performance Optimization

### Model Preloading

Load all models at app startup, not when the feature is first used:

```ts
// App.tsx or a ModelProvider
const objectModel = useTensorflowModel(require('./assets/models/yolov8n.tflite'));
const classificationModel = useTensorflowModel(require('./assets/models/mobilenet_v3.tflite'));

// Warm-up: run dummy inference to prime GPU/NPU
useEffect(() => {
  if (objectModel.state === 'loaded') {
    objectModel.model.runSync([dummyInput]);
  }
}, [objectModel.state]);
```

### Frame Resize for Inference

YOLOv8-nano expects 640x640 input. Resize frames before inference:

```ts
// In frame processor (worklet)
const resized = frame.resizeSync(640, 640);
const output = model.model.runSync([resized]);
```

### Memory Management

```ts
// Clean up when backgrounded
import { AppState } from 'react-native';

useEffect(() => {
  const sub = AppState.addEventListener('change', (state) => {
    if (state === 'background') {
      // Stop camera, pause inference, clear TTS queue
      setIsActive(false);
      ttsQueue.clear();
    } else if (state === 'active') {
      setIsActive(true);
    }
  });
  return () => sub.remove();
}, []);
```
