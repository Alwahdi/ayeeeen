# Ayeeeen Design System Reference

## Colors (High Contrast, Dark Mode Default)

```ts
export const colors = {
  // Backgrounds
  background: '#0A0A0A',    // Primary — deepest black, reduces glare + battery
  surface: '#1C1C1E',       // Cards, active areas, subtle depth

  // State Colors (screen-spanning, not small indicators)
  accent: '#3B82F6',        // Listening/Active — calm blue
  success: '#22C55E',       // Action complete — green flash
  warning: '#F59E0B',       // Processing/Alert — amber pulse
  danger: '#EF4444',        // Obstacle/Stop — red flash + vibration

  // Text (rarely used — TTS is primary output)
  textPrimary: '#FFFFFF',   // Ultra-bold status text only
  textSecondary: '#A1A1AA', // Subtle labels (if needed)
} as const;
```

### NativeWind Tailwind Config

Add to `tailwind.config.js` → `theme.extend.colors`:

```js
colors: {
  ain: {
    bg: '#0A0A0A',
    surface: '#1C1C1E',
    accent: '#3B82F6',
    success: '#22C55E',
    warning: '#F59E0B',
    danger: '#EF4444',
  },
},
```

Usage: `className="bg-ain-bg"`, `className="bg-ain-accent"`, etc.

## Typography

### Cairo Font Family

- **Only for glanceable status text** — 32px to 48px, ultra-bold
- No small body text exists. All reading is done by TTS.
- Arabic-optimized with excellent RTL support

Setup:
1. Download Cairo font files to `assets/fonts/`
2. Add `expo-font` plugin to `app.json`
3. Configure in `tailwind.config.js`:

```js
fontFamily: {
  cairo: ['Cairo_400Regular'],
  'cairo-medium': ['Cairo_500Medium'],
  'cairo-semibold': ['Cairo_600SemiBold'],
  'cairo-bold': ['Cairo_700Bold'],
},
```

Usage:
```tsx
<Text className="font-cairo-bold text-5xl text-white">جاري التحليل...</Text>
```

### Text Size Scale (for low-vision glanceable UI)

| Use Case | Size | Weight | Example |
|----------|------|--------|---------|
| Main status | `text-5xl` (48px) | `font-cairo-bold` | "مستعد" (Ready) |
| Mode indicator | `text-4xl` (36px) | `font-cairo-semibold` | "كشف الأجسام" |
| Subtle label | `text-3xl` (30px) | `font-cairo-medium` | (rare, avoid) |

## Component Patterns

### The Omni-Surface

The entire phone screen is the interaction surface. No buttons, no nav bar, no tabs.

```tsx
<Animated.View style={animatedBg} className="flex-1">
  <Pressable className="flex-1 items-center justify-center">
    {/* Camera Preview (full screen) */}
    {/* State Overlay: giant icon at 10% opacity */}
    {/* Status Text: bottom center, 48px bold */}
  </Pressable>
</Animated.View>
```

### State Overlay Icons

Screen-spanning icons at 10% opacity that give low-vision users a glanceable state cue:

| State | Icon | Opacity |
|-------|------|---------|
| Idle | — (black screen) | — |
| Listening | 🎙️ microphone | 10% |
| Processing | ⏳ or spinning | 10% |
| Success | ✓ checkmark | 10%, 1s fade |
| Danger | 🛑 stop | 30%, flashing |

### Animated Background Transitions

```tsx
const backgroundColor = useSharedValue(colors.background);

// Transition to listening
backgroundColor.value = withTiming(colors.accent, { duration: 300 });

// Pulse during processing
backgroundColor.value = withRepeat(
  withSequence(
    withTiming(colors.warning, { duration: 800 }),
    withTiming('#0A0A0A', { duration: 800 }),
  ),
  -1, // infinite
  true,
);

// Flash success then return to idle
backgroundColor.value = withSequence(
  withTiming(colors.success, { duration: 200 }),
  withDelay(800, withTiming(colors.background, { duration: 300 })),
);
```

## Layout Rules

1. **Always `flex-1` on root** — screen must fill entire viewport
2. **Always explicit `flex-col` or `flex-row`** — never rely on defaults
3. **No scroll views** — the UI is not scrollable, it's a single surface
4. **No navigation bars/tabs** — mode switching is via swipe gestures
5. **Portrait only** — locked orientation
6. **Safe area padding** — use `pt-safe` / `pb-safe` from NativeWind safe area insets
7. **RTL default** — Arabic text and layout direction

## Dark Mode

The app is ALWAYS dark. There is no light mode toggle.

```tsx
// In app.json: "userInterfaceStyle": "dark"
// All components use the dark palette:
<View className="bg-[#0A0A0A]">  // Not bg-white
<Text className="text-white">     // Not text-black
```

Enforce by setting `userInterfaceStyle: "dark"` in `app.json` (currently set to "light" — needs updating).
