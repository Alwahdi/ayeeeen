---
description: "Use when writing React Native components with NativeWind/Tailwind CSS styling, creating or modifying UI components, working with className props, configuring themes, dark mode, safe area insets, animations, transitions, or styling third-party components. Covers NativeWind v4 API, platform differences, and compatibility constraints."
applyTo: "**/*.{tsx,ts,jsx,js,css}"
---

# NativeWind v4 Styling Guide (Expo + React Native)

## Core Setup

- NativeWind uses Tailwind CSS v3 compiled via `nativewind/preset` in `tailwind.config.js`
- Babel preset: `["babel-preset-expo", { jsxImportSource: "nativewind" }], "nativewind/babel"`
- Metro: `withNativeWind(config, { input: './global.css' })`
- CSS file uses `@tailwind base; @tailwind components; @tailwind utilities;`
- Import `./global.css` at the app entry point

## Styling Components

- Use `className` prop directly on React Native components — no wrappers needed
- NativeWind works via JSX transform, so it works with 3rd-party components too
- Build dynamic classes with conditional logic and array joins:

```tsx
const classNames = [];
if (bold) classNames.push("font-bold");
return <Text className={classNames.join(" ")} />;
```

## Writing Custom Components

- Pass `className` through and merge with defaults:

```tsx
function MyComponent({ className, ...props }) {
  return <Text className={`text-black dark:text-white ${className}`} {...props} />;
}
```

- For components with multiple style props, use separate className props:

```tsx
function MyComponent({ className, textClassName }) {
  return (
    <View className={className}>
      <Text className={textClassName}>Hello</Text>
    </View>
  );
}
```

- Never use `cssInterop` or `remapProps` for your own components — only for third-party components

## Third-Party Components

- Use `remapProps` for components with multiple style props:

```tsx
remapProps(ThirdPartyComponent, {
  className: "style",
  contentContainerClassName: "contentContainerStyle",
});
```

- Use `cssInterop` when a component needs style attributes as props:

```tsx
cssInterop(Svg, {
  className: {
    target: "style",
    nativeStyleToProp: { width: true, height: true },
  },
});
```

### Common Third-Party cssInterop Setups

**react-native-svg** — SVG components need `cssInterop` to map style props:

```tsx
import { cssInterop } from "nativewind";
import Svg, { Circle, Rect, Path, Line, G } from "react-native-svg";

cssInterop(Svg, {
  className: { target: "style", nativeStyleToProp: { width: true, height: true } },
});
cssInterop(Circle, {
  className: { target: "style", nativeStyleToProp: { width: true, height: true, stroke: true, strokeWidth: true, fill: true } },
});
cssInterop(Rect, {
  className: { target: "style", nativeStyleToProp: { width: true, height: true, stroke: true, strokeWidth: true, fill: true } },
});
cssInterop(Path, {
  className: { target: "style", nativeStyleToProp: { stroke: true, strokeWidth: true, fill: true } },
});
cssInterop(Line, {
  className: { target: "style", nativeStyleToProp: { stroke: true, strokeWidth: true } },
});
```

**expo-camera** — Camera view styled via remapProps:

```tsx
import { remapProps } from "nativewind";
import { CameraView } from "expo-camera";

remapProps(CameraView, { className: "style" });
```

**expo-blur / expo-linear-gradient** — same pattern:

```tsx
import { BlurView } from "expo-blur";
import { LinearGradient } from "expo-linear-gradient";

remapProps(BlurView, { className: "style" });
remapProps(LinearGradient, { className: "style" });
```

## Cairo Font Setup

This project uses **Cairo** as the primary Arabic font.

### File Structure

```
assets/fonts/
  Cairo-Regular.ttf
  Cairo-Bold.ttf
  Cairo-SemiBold.ttf
  Cairo-Medium.ttf
  Cairo-Light.ttf
  Cairo-ExtraBold.ttf
  Cairo-Black.ttf
```

### Loading via expo-font (app.json)

```json
{
  "expo": {
    "plugins": [
      ["expo-font", {
        "fonts": [
          "./assets/fonts/Cairo-Regular.ttf",
          "./assets/fonts/Cairo-Bold.ttf",
          "./assets/fonts/Cairo-SemiBold.ttf",
          "./assets/fonts/Cairo-Medium.ttf",
          "./assets/fonts/Cairo-Light.ttf",
          "./assets/fonts/Cairo-ExtraBold.ttf",
          "./assets/fonts/Cairo-Black.ttf"
        ]
      }]
    ]
  }
}
```

### Tailwind Config

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      fontFamily: {
        cairo: ["Cairo-Regular"],
        "cairo-light": ["Cairo-Light"],
        "cairo-medium": ["Cairo-Medium"],
        "cairo-semibold": ["Cairo-SemiBold"],
        "cairo-bold": ["Cairo-Bold"],
        "cairo-extrabold": ["Cairo-ExtraBold"],
        "cairo-black": ["Cairo-Black"],
      },
    },
  },
};
```

### Usage

```tsx
<Text className="font-cairo">نص عادي</Text>
<Text className="font-cairo-bold">نص عريض</Text>
<Text className="font-cairo-semibold text-4xl">عنوان</Text>
```

### Important Notes

- React Native does NOT support fallback fonts — only the first font in an array is used
- File name MUST match the font's PostScript name (check in Font Book on macOS)
- Do NOT use `font-bold` to make Cairo bold — use `font-cairo-bold` (separate font family)
- Variable fonts do NOT work on React Native — use static weight files only
- Verify fonts work with inline `style={{ fontFamily: "Cairo-Regular" }}` before using className

## Platform Differences (Critical)

- **Flex direction**: React Native defaults to `column`, web to `row` — always set explicitly
- **Flex**: Use `flex-1` for consistency across platforms
- **dp vs px**: Nativewind treats them as equivalent; use `px` in theme values
- **rem**: Web uses 16px, native uses 14px — Nativewind handles this automatically
- **Colors on Views**: `<View>` does not accept `color` style — move text colors to `<Text>`
- **Always declare explicit styles**: Provide both light AND dark mode styles:

```tsx
// ❌ Bad
<Text className="dark:text-white" />
// ✅ Good
<Text className="text-black dark:text-white" />
```

- Platform modifiers: `ios:`, `android:`, `web:`, `native:`

## Color Scheme / Dark Mode

- Use `useColorScheme()` from `nativewind` to read color scheme
- Use `colorScheme.set("dark" | "light" | "system")` for manual toggle
- `setColorScheme` and `toggleColorScheme` require `darkMode: "class"` in tailwind config
- System preference mode is default and recommended

## CSS Variables & Dynamic Themes

- Use `vars()` from `nativewind` to set CSS variables inline:

```tsx
import { vars } from "nativewind";
<View style={vars({ "--brand-color": brandColor })}>
  <Text className="text-[--brand-color]">Themed</Text>
</View>
```

- Use `useUnstableNativeVariable("--var-name")` to read variable values in JS

## Safe Area Insets

- Wrap app with `<SafeAreaProvider>` (Expo Router does this automatically)
- Use utility classes: `p-safe`, `pt-safe`, `pb-safe`, `m-safe`, `h-screen-safe`
- Offset variants: `mt-safe-or-4`, `pt-safe-offset-[2px]`

## Animations & Transitions (Experimental)

- Powered by `react-native-reanimated`
- Supported: `animate-spin`, `animate-ping`, `animate-pulse`, `animate-bounce`, custom keyframes
- Transitions: `transition`, `transition-all`, `transition-colors`, `transition-opacity`, `transition-transform`
- Timing: `duration-{n}`, `delay-{n}`, `ease-{n}`
- `transition-shadow` is NOT supported

## States & Pseudo-classes

- `hover:` requires `onHoverIn/Out` (works on `Pressable`, `TextInput`)
- `active:` requires `onPressIn/Out`
- `focus:` requires `onFocus/Blur`
- `disabled:` checks `disabled={true}` prop
- Group states: parent gets `group/{name}`, children use `group-active/{name}:`

## Native Compatibility Summary

### ✅ Fully Supported
- Layout: `flex`, `hidden`, `absolute`, `relative`, `container`
- Flexbox: `flex-1`, `flex-row`, `flex-col`, `items-*`, `justify-*`, `self-*`, `gap-*`
- Sizing: `w-{n}`, `h-{n}`, `w-full`, `h-full`, `w-screen`, `h-screen`, `min-*`, `max-*`
- Spacing: `m-*`, `p-*` (all directions), `m-auto`
- Borders: `border-*`, `rounded-*`, `border-solid`, `border-dashed`, `border-dotted`
- Colors: `bg-{n}`, `text-{n}`, `border-{n}`
- Typography: `text-{size}`, `font-{weight}`, `font-{family}`, `italic`, `uppercase`, `lowercase`, `capitalize`, `tracking-*`, `leading-{n}`, `line-clamp-*`
- Effects: `opacity-*`, `shadow-*` (needs bg color on native)
- Transforms: `rotate-*`, `scale-*`, `skew-*`, `translate-*`
- SVG (with cssInterop): `fill-*`, `stroke-*`
- Pointer events: `pointer-events-none`, `pointer-events-auto`
- Aspect ratio: `aspect-auto`, `aspect-video`, `aspect-square`
- Container queries (with plugin)

### 🧪 Experimental (Reanimated)
- `animate-*`, `transition-*`, `duration-*`, `delay-*`, `ease-*`

### ❌ NOT Supported on Native
- Grid: `grid`, `grid-cols-*`, `grid-rows-*`
- Display: `block`, `inline`, `inline-flex`, `table`
- Position: `fixed`, `sticky`
- Filters: `blur`, `brightness`, `contrast`, `grayscale`, `sepia`, `backdrop-*`
- Background: `bg-gradient-*`, `bg-clip-*`, `bg-repeat-*`, `bg-position-*`
- Overflow axis: `overflow-x-*`, `overflow-y-*`
- Ring: `ring-*`
- Outline: `outline-*`
- Text: `truncate`, `text-ellipsis`, `indent-*`, `whitespace-*`, `word-break-*`
- Line height relative: `leading-none`, `leading-tight`, etc.
- `border-none` — use `border-0` instead
- `shadow-inner`, `shadow-[n]` arbitrary

## Theme Configuration

- Use `platformSelect()` for per-platform values
- Use `platformColor()` for native system colors
- Use `hairlineWidth()` for `StyleSheet.hairlineWidth`
- Use `pixelRatio()`, `fontScale()` for device-specific scaling

```js
const { platformSelect, platformColor } = require("nativewind/theme");
module.exports = {
  theme: {
    extend: {
      colors: {
        error: platformSelect({
          ios: platformColor("systemRed"),
          android: platformColor("?android:colorError"),
          default: "red",
        }),
      },
    },
  },
};
```

## Style Specificity

- Order (highest to lowest): `!important` > inline/remapped styles > className styles
- Use `!` prefix to force specificity: `!text-red-500`

## TypeScript

- Add `/// <reference types="nativewind/types" />` in `nativewind-env.d.ts`
- Do NOT name the file `nativewind.d.ts` or match any folder name

## Troubleshooting

- Always clear cache: `npx expo start --clear`
- Verify Tailwind compiles: `npx tailwindcss --input ./global.css --output output.css`
- Use `verifyInstallation()` from `nativewind` inside a component (not global scope)
- Debug mode: `DEBUG=nativewind npx expo start --clear`
- Shadows need a background color on native
- `rotate` arbitrary values need `deg` unit: `rotate-[90deg]`
