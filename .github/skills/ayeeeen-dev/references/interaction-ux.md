# Interaction & UX Reference

## Gesture Map

| Gesture | Action | Context | Implementation |
|---------|--------|---------|---------------|
| **Single Tap** | Describe scene / Read text | Anywhere on screen | `<Pressable onPress>` |
| **Long Press** | Toggle Auto-Mode (continuous scanning) | Anywhere on screen | `<Pressable onLongPress>` |
| **Two-Finger Tap** | Repeat last spoken phrase | Anywhere on screen | Custom via `react-native-gesture-handler` |
| **Swipe Left** | Next mode | Full screen | Pan gesture, velocity-based |
| **Swipe Right** | Previous mode | Full screen | Pan gesture, velocity-based |
| **Shake** | Emergency Stop (panic button) | Device-wide | `expo-sensors` Accelerometer |

### Mode Carousel (Swipe Navigation)

```
← Swipe Left                    Swipe Right →
[Scene Description] ↔ [OCR/Reading] ↔ [Face Recognition]
```

Each mode change announces itself via TTS: "وضع قراءة النصوص" (Text reading mode).

## Haptic Dictionary

```ts
// shared/constants/haptics.ts
import * as Haptics from 'expo-haptics';

export const hapticPatterns = {
  // Interaction feedback
  tap: () => Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light),
  select: () => Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium),

  // Result feedback
  success: async () => {
    await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
    await delay(100);
    await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
    // tick-tick
  },

  error: () => Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy),
    // thud

  // Danger — continuous vibration
  danger: async () => {
    for (let i = 0; i < 5; i++) {
      await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy);
      await delay(150);
    }
    // bzzzz-bzzzz-bzzzz
  },

  // Proximity ping — speeds up as user approaches object
  proximityPing: (distance: number) => {
    // distance: 0-1 normalized (0 = touching, 1 = far)
    const interval = Math.max(100, distance * 1000); // 100ms-1000ms
    Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
    return interval; // caller sets timeout for next ping
  },
} as const;
```

## Voice UX Rules

### Tone & Style
- **Calm, concise, authoritative but friendly**
- Maximum 2 sentences per response
- User can tap for more detail

### Arabic TTS Templates

```ts
// shared/constants/arabic.ts

// Object Detection Results
const objectTemplates = {
  single: (obj: string, direction: string, distance: string) =>
    `${obj} ${direction} على بعد ${distance}`,
  // "كرسي أمامك على بعد مترين"

  multiple: (objects: string[]) =>
    `أمامك ${objects.join('، و')}`,
  // "أمامك طاولة، وكرسيين، وباب"

  clear: () => 'المسار واضح أمامك',
  // "The path is clear ahead"
};

// Danger Alerts (ALWAYS interrupt current speech)
const dangerTemplates = {
  stairs: (direction: 'up' | 'down') =>
    `انتبه! درج ${direction === 'down' ? 'للأسفل' : 'للأعلى'} أمامك مباشرة!`,

  movingObject: (obj: string) =>
    `توقف، ${obj} يعبر أمامك`,

  obstacle: (obj: string, distance: string) =>
    `حذر! ${obj} على بعد ${distance}`,

  edge: () => 'انتبه! حافة أمامك',
};

// Navigation
const navTemplates = {
  turnLeft: () => 'انعطف يساراً',
  turnRight: () => 'انعطفي يميناً',
  straight: () => 'استمر للأمام',
  arrived: () => 'وصلت إلى وجهتك',
};

// System
const systemTemplates = {
  ready: () => 'مستعد',
  processing: () => 'جاري التحليل...',
  error: () => 'عذراً، حاول مرة أخرى',
  modeSwitch: (mode: string) => `وضع ${mode}`,
  autoModeOn: () => 'وضع المسح التلقائي مفعّل',
  autoModeOff: () => 'تم إيقاف المسح التلقائي',
  emergencyStop: () => 'تم الإيقاف',
};

// Directions
const directions = {
  ahead: 'أمامك',
  left: 'على يسارك',
  right: 'على يمينك',
  above: 'فوقك',
  below: 'تحتك',
  behind: 'خلفك',
} as const;
```

### Voice Rules Checklist

1. **Never say "أرى" (I see)** — State what exists with spatial context
2. **Always include direction** — ahead, left, right, above, below
3. **Include distance when available** — "على بعد مترين" (2 meters away)
4. **Danger overrides everything** — `Speech.stop()` then speak danger alert
5. **No filler words** — No "يبدو أن" (it seems), no "أعتقد" (I think)
6. **Consistent spatial terms** — Use the `directions` map, don't vary phrasing

## Spatial Audio

Pan TTS and alert sounds based on where objects are in the camera frame:

```ts
// shared/utils/spatial-audio.ts

// Camera frame is divided into zones:
// [Left 30%] [Center 40%] [Right 30%]

export function getAudioPan(bboxCenterX: number, frameWidth: number): number {
  const normalized = bboxCenterX / frameWidth; // 0-1
  // Map to -1 (full left) to +1 (full right)
  return (normalized - 0.5) * 2;
}

// Use with expo-av for panned audio playback
// Use with TTS by prefixing direction in speech: "على يسارك، كرسي"
```

## Onboarding Flow

First-time app open (blind-first design):

```
1. App Opens
   → Heavy haptic
   → TTS: "مرحباً بك في عين. أنا هنا لأكون عينيك. اضغط في أي مكان على الشاشة للبدء."
   → Screen: solid #0A0A0A

2. User Taps Anywhere
   → Light haptic
   → TTS: "ممتاز. سأحتاج إلى إذن الكاميرا والميكروفون لمساعدتك."
   → Screen: #3B82F6

3. Camera Permission
   → TTS: "اسحب لليمين للموافقة على استخدام الكاميرا"
   → Native permission dialog (VoiceOver/TalkBack reads it)

4. Microphone Permission
   → TTS: "اسحب لليمين للموافقة على استخدام الميكروفون"
   → Native permission dialog

5. Ready
   → Success haptic (tick-tick)
   → TTS: "أنت جاهز! اضغط في أي مكان لبدء وصف ما حولك. اضغط مطولاً للمسح التلقائي."
   → Screen: #22C55E flash → #0A0A0A
```

## Emergency Stop (Shake)

The shake gesture is the universal panic button:

```ts
// hooks/useEmergencyStop.ts
import { Accelerometer } from 'expo-sensors';

export function useEmergencyStop(onStop: () => void) {
  useEffect(() => {
    const subscription = Accelerometer.addListener(({ x, y, z }) => {
      const magnitude = Math.sqrt(x * x + y * y + z * z);
      if (magnitude > 2.5) { // Shake threshold
        onStop();
      }
    });
    Accelerometer.setUpdateInterval(100);
    return () => subscription.remove();
  }, [onStop]);
}

// onStop should:
// 1. Speech.stop() — kill all TTS
// 2. Cancel any pending ML inference
// 3. Stop continuous scanning
// 4. Heavy haptic feedback
// 5. Brief TTS: "تم الإيقاف"
// 6. Return to idle state (#0A0A0A)
```
