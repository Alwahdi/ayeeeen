# On-Device ML Model Reference

All AI runs locally on the device. No cloud APIs, no internet required.

## Model Selection Matrix

| Feature | Library | Model | Platform | Performance |
|---------|---------|-------|----------|-------------|
| Object Detection | `react-native-fast-tflite` | YOLOv8-nano TFLite | Android + iOS | ~30fps, 3.2MB |
| OCR (Arabic + EN) | `@react-native-ml-kit/text-recognition` v2 | ML Kit Text Recognition | Android + iOS | <500ms per frame |
| Face Detection | `@react-native-ml-kit/face-detection` | ML Kit Face Detection | Android + iOS | Real-time, landmarks |
| Barcode/QR | `@react-native-ml-kit/barcode-scanning` | ML Kit Barcode | Android + iOS | Instant, all formats |
| Scene Description | `react-native-fast-tflite` | Florence-2 / MobileVLM | Android + iOS | Phase 2, 1-3s |
| Speech-to-Text | `whisper.cpp` native module | Whisper tiny/base | Android + iOS | ~1s for 5s audio |
| Text-to-Speech | `expo-speech` | Native OS TTS engine | Android + iOS | Instant, Arabic voice |
| Color Detection | Custom pixel sampling | Camera frame analysis | Android + iOS | Instant |
| Image Classification | `react-native-fast-tflite` | MobileNetV3 | Android + iOS | <100ms, 5.4MB |

## Phase 1 Models (Ship First)

These are proven, well-supported, and production-ready:

### Object Detection — YOLOv8-nano

```ts
// hooks/useObjectDetection.ts
import { useTensorflowModel } from 'react-native-fast-tflite';

export function useObjectDetection() {
  const model = useTensorflowModel(require('../assets/models/yolov8n.tflite'));

  const detect = useCallback(async (frame: Frame) => {
    if (model.state !== 'loaded') return [];
    const result = model.model.runSync([frame]);
    return parseYOLOOutput(result); // → [{label, confidence, bbox}]
  }, [model]);

  return { detect, isReady: model.state === 'loaded' };
}
```

**Setup:**
1. `npm install react-native-fast-tflite`
2. Download `yolov8n.tflite` + `labels.txt` to `assets/models/`
3. For camera frames, use `react-native-vision-camera` with frame processors

### OCR — ML Kit Text Recognition

```ts
// hooks/useTextRecognition.ts
import TextRecognition from '@react-native-ml-kit/text-recognition';

export function useTextRecognition() {
  const recognizeFromImage = useCallback(async (imageUri: string) => {
    const result = await TextRecognition.recognize(imageUri, {
      // v2 supports Arabic script
      script: TextRecognition.Script.ARABIC,
    });
    return result.text; // Full extracted text
  }, []);

  return { recognizeFromImage };
}
```

**Setup:**
1. `npm install @react-native-ml-kit/text-recognition`
2. Requires `expo-camera` for live capture or `expo-image-picker` for photos
3. Arabic text recognition needs ML Kit v2 — verify version supports Arabic script

### Face Detection — ML Kit

```ts
import FaceDetection from '@react-native-ml-kit/face-detection';

const faces = await FaceDetection.detect(imageUri, {
  performanceMode: 'fast',
  landmarkMode: 'all',
  classificationMode: 'all', // smiling, eyes open
});
// → [{bounds, landmarks, smilingProbability, ...}]
```

### Barcode Scanning — ML Kit

```ts
import BarcodeScanning from '@react-native-ml-kit/barcode-scanning';

const barcodes = await BarcodeScanning.scan(imageUri, {
  formats: [
    BarcodeScanning.BarcodeFormat.QR_CODE,
    BarcodeScanning.BarcodeFormat.EAN_13,
    BarcodeScanning.BarcodeFormat.EAN_8,
  ],
});
// → [{value, format, boundingBox}]
```

### Text-to-Speech — expo-speech

```ts
// hooks/useTTS.ts
import * as Speech from 'expo-speech';

export function useTTS() {
  const speak = useCallback((text: string, options?: { interrupt?: boolean }) => {
    if (options?.interrupt) {
      Speech.stop(); // Cut off current speech for danger alerts
    }
    Speech.speak(text, {
      language: 'ar-SA',   // Arabic (Saudi)
      rate: 0.9,           // Slightly slower for clarity
      pitch: 1.0,
      onDone: () => { /* return to idle state */ },
      onError: () => { /* haptic error feedback */ },
    });
  }, []);

  const stop = useCallback(() => Speech.stop(), []);

  return { speak, stop };
}
```

## Phase 2 Models (After Core Is Stable)

These require more optimization work:

### Scene Description — Florence-2 / MobileVLM

- Generates natural language descriptions of camera frames
- Requires quantized model (~500MB → optimize to ~100MB)
- Inference time: 1-3 seconds on recent devices
- Arabic output requires either: (a) Arabic-trained VLM, or (b) English VLM + on-device translation
- Consider ONNX Runtime for cross-platform inference

### Speech-to-Text — Whisper.cpp

- Offline Arabic speech recognition
- Whisper tiny model: ~75MB, good for short commands
- Whisper base model: ~142MB, better accuracy
- Integration via native module (C++ bridge)
- Libraries: `whisper.rn` (React Native bindings for whisper.cpp)

## Camera Pipeline Architecture

```
Camera Frame (react-native-vision-camera)
    │
    ├─→ Frame Processor (runs on UI thread via worklets)
    │       │
    │       ├─→ Object Detection (YOLOv8, every frame)
    │       ├─→ Danger Detection (priority, every frame)
    │       └─→ OCR (on-demand, single frame capture)
    │
    ├─→ Results → Spatial Mapping (where in frame → which ear)
    │
    └─→ TTS Queue (priority-ordered)
            ├─→ [P0] Danger: immediate interrupt
            ├─→ [P1] Navigation: next in queue
            ├─→ [P2] Scene: after nav completes
            └─→ [P3] OCR/General: lowest priority
```

## Model File Organization

```
assets/
  models/
    yolov8n.tflite           # Object detection (~3.2MB)
    yolov8n-labels.txt        # Class labels
    mobilenet_v3.tflite       # Image classification (~5.4MB)
    whisper-tiny.bin          # STT model (~75MB) — Phase 2
    florence2-quant.onnx      # Scene description (~100MB) — Phase 2
```

## Performance Guidelines

1. **Frame rate target**: 30fps for detection, process every 3rd frame if needed
2. **Model loading**: Load on app start, keep in memory. Don't reload per-frame.
3. **Thread management**: ML inference on a background thread, never block UI
4. **Memory budget**: Keep total model memory under 200MB
5. **Battery**: Disable continuous scanning when app is backgrounded
6. **Warm-up**: Run a dummy inference on startup to prime the model
