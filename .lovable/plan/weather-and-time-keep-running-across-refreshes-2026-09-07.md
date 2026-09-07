# Weather and time keep running across refreshes

## Problem

Right now the in-game clock always starts at 07:00 and the weather always starts at "Clear" whenever the page loads. The weather then re-rolls randomly on a timer while you play. So if it is raining at 09:00 and you refresh, you come back to a sunny 07:00 morning.

## Goal

After a refresh — at any moment — you land back into exactly the weather and time that the world is in at that moment. No reset, no jump.

## Approach

Make the clock and the weather calculated from the real elapsed time since a fixed starting point, instead of from "when this browser tab opened".

- Time of day: derived from the current real time and the configured day length, so it always continues where the world is.
- Weather: the timeline is split into fixed slots (the existing change interval). Each slot's weather is picked with a repeatable draw based on the slot number and the configured weather chances. The same slot always produces the same weather, so a refresh in the middle of a rainy slot returns rain.
- Side effect (positive): every player sees the same weather and the same time, which also makes the world feel shared.

## Technical notes

- `src/hooks/useDayNight.ts`: replace the frame-accumulated `clock.hour` seed with `hourFromEpoch(Date.now())` using `dayLengthSeconds()`; keep the per-frame advance for smoothness but re-anchor to real time on init and periodically to avoid drift.
- `src/hooks/useWeather.ts`: initialise `kind` lazily from a new `weatherForSlot(slotIndex)` helper — a small deterministic PRNG (hash of slot index) sampling `getFishData().weatherCycle.weights`.
- `src/components/game/WeatherCycleController.tsx`: instead of an elapsed-time counter with `Math.random()`, compute the current slot index from `Date.now()` each frame and call `setKind(weatherForSlot(slot))` only when the slot changes.
- Weather data loads asynchronously (`fishData.functions.ts`); once weights arrive, recompute the current slot's weather so the first paint corrects itself instead of staying on the default.
- No database changes required.

## Verification

- Load the game, note weather and clock, refresh: both continue instead of resetting.
- Open two tabs: same weather and same time in both.
