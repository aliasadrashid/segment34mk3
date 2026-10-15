# Rain/dry forecast field ("when will it rain / stop raining")

## Context

A user requested the Datascope-style rain field: text like `100% in 50 minutes`,
`Dry in 30 minutes`, `60% in 2 days`, `Dry 3 days`. This is already on the
wishlist in `README.md` ("Things people have asked for" — "When is next
rain/non-rain"). The user wants it available on the long fields (Line 1 /
Line 2 above the clock), and wants credit given to "Ondrej" in the release
notes.

Key finding from research: **none of the 3 weather backends currently fetch
multi-day precipitation data** — everything tops out at ~24h. Per the user's
steer, we should fetch/process only as much as needed and avoid persisting a
full forecast we don't otherwise need — we only care about "when does the
rain/dry state next change." So instead of storing multi-day arrays, each
backend computes one small derived result at fetch time and stores only that.

## Design

### Fits the existing token-format system, not a new field-type code

`weatherLine1Shows` / `weatherLine2Shows` (the "Line 1/2" long fields) already
support arbitrary composed text via `weatherFormat1`/`weatherFormat2`
(free-text properties) and `WeatherDisplayHelper.getWeatherByFormat()`
(`source/WeatherDisplayHelper.mc:235`), which expands single-letter tokens
(`t`=temp, `w`=wind, `p`=precip%, `u`=UV, `m`=max UV next 8h, etc.). This is
exactly the "available for the longer fields" mechanism the user asked for.

Plan: add one new token, **`x`**, meaning "rain/dry forecast". No new
`*Shows` type code, no `ValueResolver` switch-case, no `settings.xml`
listEntry changes, no `properties.xml` changes — users opt in by adding `x`
to their existing `weatherFormat1`/`weatherFormat2` string. Only the token
legend string (`settings_weather_format_1`/`_2` in
`resources/strings/strings.xml:377-378`) needs updating to document it.
No translation files need touching — computed/short text in this codebase is
already hardcoded English (see `getWeatherConditionShort()`,
`WeatherDisplayHelper.mc:93`), not localized via `Rez.Strings`; the new
literal words ("Dry", "in") follow that same precedent.

### New compact storage key: `rain_forecast`

Each backend computes this once per fetch and stores it — no raw multi-day
array is persisted (keeps the existing memory-constrained storage schema
untouched: `current_conditions` and `hourly_forecast` are unchanged).

```
{
  "kind"   => 0 | 1 | 2   // 0 = dry now, rain incoming; 1 = raining now, will dry; 2 = dry for whole lookahead
  "at"     => Number      // epoch seconds of the transition (kind 0 or 1 only)
  "chance" => Number      // percent, the incoming rain's probability (kind 0 only)
}
```

`WeatherDisplayHelper` reads this dict + `Time.now()` and formats:
- kind 0: `"<chance>% in " + formatDuration(at - now)`
- kind 1: `"Dry in " + formatDuration(at - now)`
- kind 2: `"Dry " + LOOKAHEAD_DAYS + "+ days"` (per the user's suggestion —
  we only looked `LOOKAHEAD_DAYS` ahead, so phrase it as a lower bound, not
  a precise claim)

`formatDuration(seconds)`: `<60min` → `"Xm"`, `<24h` → `"Xh"` (rounded),
else → `"Xd"` (safety fallback, shouldn't normally trigger since day-level
crossings are handled by the `kind` logic itself).

Threshold: **50%** precipitation probability, used both to detect rain onset
and to detect it turning dry (confirmed with user). Constant
`RAIN_THRESHOLD_PERCENT = 50` in `WeatherDisplayHelper.mc`.

Lookahead cap: **3 days** (`LOOKAHEAD_DAYS = 3`), chosen because it lets
OpenWeatherMap's request stay modest (3 days = 24 3-hourly slots vs. the 40
needed for 5 days) while still giving a useful answer; the "dry N+ days"
phrasing means we don't need to claim more precision than we fetched.

### Per-backend computation (mirrors `maxUvIndexNext8h` style: a static, pure,
unit-testable function per backend/shape — though per project convention
(established preference: skip TDD/tests in this repo — messy to mock Garmin
weather data), no test file will be added)

- **`OpenMeteoService.mc`** (`source/OpenMeteoService.mc`): add
  `precipitation_probability_max` to the `daily` param, bump
  `forecast_days` from `2` → `4` (today + 3 more, matches the 3-day
  lookahead beyond today). In `onResponse()`, after building the existing
  local `hf_data` (hourly, unchanged) and the local daily arrays, compute
  `rain_forecast` from: hourly slots for the near-term crossing, then daily
  `precipitation_probability_max[1..3]` for anything beyond the hourly
  horizon. Store only the compact dict — the daily precip array itself is
  never persisted.

- **`OpenWeatherService.mc`** (`source/OpenWeatherService.mc`): bump `cnt`
  from `8` → `24` (3 days at 3h resolution) in the forecast request. In
  `onForecastResponse()`, keep persisting `hourly_forecast` from only the
  first 8 entries (unchanged — existing graph/max-UV features rely on that
  window), but scan the *full* local `list` (up to 24 entries) to compute
  `rain_forecast`, using each 3-hour slot's `pop` bucketed into which day it
  falls on. Persist only the compact dict; the extra 16 fetched entries are
  discarded after computing it.

- **`WeatherStorage.mc`** (native Garmin path): call
  `Weather.getDailyForecast()` (guarded by `Weather has :getDailyForecast`,
  same pattern as the existing `Weather has :getHourlyForecast` check at
  line 64), alongside the already-fetched `Weather.getHourlyForecast()`.
  Use up to the first 3 daily entries for the beyond-today crossing, and the
  hourly array for the near-term crossing. Persist only the compact dict —
  neither the daily nor the extra hourly data is stored beyond what's
  already kept today.

  Need to confirm `Weather.DailyForecast` exposes `forecastDate` and
  `precipitationChance` (expected, mirroring `HourlyForecast`'s
  `forecastTime`/`precipitationChance` already used at
  `WeatherStorage.mc:77-79`) — verify at compile time (Monkey C would fail
  to resolve the field if wrong) and via `x has :precipitationChance` guard
  consistent with the existing `uvIndex` guard pattern (line 84).

- Note: Only fetch extra weather data if this feature is IN USE, which means we also have to re-fetch it if the user starts using it after a settings update.

- Shared crossing-detection logic (the "scan forward for first slot
  crossing the threshold" part) should be a small shared static helper in
  `WeatherDisplayHelper.mc` that each backend's storage code can call with
  its own hourly+daily data, to avoid duplicating the same loop three times.

### `WeatherDisplayHelper.mc`

- Add `getRainForecast() as String`: reads `Application.Storage["rain_forecast"]`
  and `Time.now()`, formats per the rules above. Returns `""` if the key is
  absent (matches existing getters' behavior on missing data).
- Wire into `getWeatherByFormat()` (`WeatherDisplayHelper.mc:235`): add
  `else if (ch.equals("x")) { result = result + getRainForecast(); }`.

### Docs

- `resources/strings/strings.xml:377-378`: append `x=rain/dry forecast` to
  both `settings_weather_format_1` and `settings_weather_format_2` legend
  strings.
- `README.md`: add a bullet to the current top changelog section (`3.0`,
  line 88) crediting the requester, matching the existing crediting style
  seen in git history (e.g. `"...thanks Alvin for the report!"`):
  `- Weather field to show when it will start/stop raining (thanks Ondrej!)`.
  Remove the now-implemented wishlist bullet at `README.md:83-84`.

## Verification

- `monkeybrains` build/type-check for at least one MIP device (e.g.
  `fenix847mm` or similar per `AGENTS.md`'s build command) to confirm the
  new `Weather.getDailyForecast()` fields resolve and everything compiles
  across the annotation families.
- Manually trace through the `kind`/threshold logic with a few concrete
  example timestamps (mirroring the user's own examples: currently raining
  at 80% → dry in ~30m; dry now, 60% at hour 20 → "60% in ~2d" if beyond the
  hourly horizon) to confirm `formatDuration` and day-bucketing produce the
  expected strings.
- Live end-to-end testing of real forecast data isn't practical in the
  simulator (no mocked weather source), so this will need real-device
  verification with `x` added to `weatherFormat1` once merged.
