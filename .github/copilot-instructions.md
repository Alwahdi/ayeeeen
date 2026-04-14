# Ayeeeen (عين / Ain) — Copilot Instructions

## Project Overview
Ain is an AI-powered accessibility app for blind/visually impaired users built with Expo + React Native + NativeWind. It uses on-device ML models (no cloud AI) for object detection, OCR, scene description, and navigation.

## Tech Stack
- **Framework**: Expo SDK 54, React Native 0.81
- **Styling**: NativeWind v4 (Tailwind CSS v3 for React Native)
- **Animations**: react-native-reanimated v4
- **Language**: TypeScript (strict mode)
- **Package Manager**: npm

## Design System (Audio-First, Dark Mode Default)
- Primary Background: `#0A0A0A` (deepest black)
- Surface: `#1C1C1E`
- Accent/Listening: `#3B82F6` (blue)
- Success: `#22C55E` (green)
- Warning/Processing: `#F59E0B` (amber)
- Danger/Stop: `#EF4444` (red)
- Typography: Cairo font family (Arabic-optimized), 32-48px ultra-bold status text only
- No small body text — reading is done via TTS

## Architecture Principles
- **Audio-first, not visual-first**: Screen is an input canvas, not output display
- **Zero cognitive overload**: Entire screen is the interaction surface
- **All models run locally**: No cloud AI, no API costs, works offline
- **Arabic-first**: UI and TTS in Arabic

## Coding Conventions
- Use NativeWind className for all styling (no inline StyleSheet.create unless necessary)
- Always set explicit flex direction on containers
- Always provide both light AND dark mode styles when using conditional colors
- Use `Pressable` for interactive elements (supports all pseudo-classes)
- Wrap app with `<SafeAreaProvider>`
- Import `./global.css` at entry point

## Commands
- `npm run start` — Start Expo dev server
- `npm run android` — Run on Android
- `npm run ios` — Run on iOS
- `npm run lint` — ESLint + Prettier check
- `npm run format` — Auto-fix lint/format issues
