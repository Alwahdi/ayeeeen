Your workflow
Daily dev: work on develop branch → push → auto dev build
Ready to test: open PR to main → auto preview build + OTA
Ship it: merge PR to main → auto production build + OTA
Hotfix JS-only: npm run update:prod → instant OTA to all production users

npm run build:dev       # Dev build (Android)
npm run build:preview   # Preview build (Android)
npm run build:prod      # Production build (Android)
npm run update:dev      # OTA to development channel
npm run update:preview  # OTA to preview channel
npm run update:prod     # OTA to production channel