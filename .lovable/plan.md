# Branding cleanup, new favicon, new loading screen

## What changes

1. **Remove Lovable branding**
   - Hide the "Edit with Lovable" badge on the published site.
   - Remove Lovable wording from the readme and any visible text in the app.
   - Keep the internal build/runtime pieces that carry the Lovable name (build config package, error reporting helper) — removing those breaks the app, so they are only renamed where they are safe to touch. Nothing of them is visible to players.

2. **New favicon (larger)**
   - Use the round blue angler logo you just uploaded as the site icon, exported at 256x256 so it stays sharp on tabs, bookmarks and phone home screens.
   - Delete the old icon file so nothing stale is served.

3. **New loading screen image**
   - Replace the current loading artwork with the second uploaded picture (angler on the pier with the ice fish).
   - Keep the existing layout: full-bleed image, title, progress bar with the swimming fish.

4. **Images stop using the JSON pointer format**
   - The fish coin logo is currently stored as a JSON pointer to a hosted file. It becomes a real image file inside the project, and the wallet panel loads it directly.
   - Non-image assets (3D boat models, sounds) keep their current setup — they are not images and are large.

5. **Longer loading screen**
   - The overlay stays visible longer after assets finish (a slower fade and a minimum on-screen time of roughly 3 seconds instead of ~1 second), so the artwork is readable instead of flashing by.

## Technical notes

- `public/favicon.png` regenerated at 256x256 from the uploaded logo; `__root.tsx` head link updated with explicit sizes.
- `src/assets/loading-banner.jpg` replaced by the uploaded `loadingscreen.webp` (kept as a local import in `LoadingScreen.tsx`).
- `src/assets/logo-fish.png.asset.json` replaced with a real `logo-fish.png`; `WalletButton.tsx` import switched, pointer removed via the assets CLI.
- `LoadingScreen.tsx`: add a mount timestamp and only start the hide timers once a minimum display time has elapsed; fade duration raised.
- Badge hidden through publish settings (requires Pro; if unavailable I'll report it).
