# Eitri iOS Health

## Installation via Swift Package Manager (Xcode)

1. Open the main project or workspace in Xcode.
2. Go to `File > Add Packages...`.
3. Enter the URL for this repository (https://github.com/Calindra/eitri-ios-health) when prompted.
4. Ensure the `EitriHealth` product is added to the main target.

## Configuration

### HealthKit Capability

Your host app must enable the **HealthKit** capability in Xcode:

1. Open the host app target in Xcode.
2. Go to `Signing & Capabilities`.
3. Click `+ Capability` and add `HealthKit`.

This adds the `com.apple.developer.healthkit` entitlement required to access the HealthKit store.

### Info.plist Requirements

Your host app must include the following key in its `Info.plist` file:

#### For Reading Health Data

```xml
<key>NSHealthShareUsageDescription</key>
<string>Your app needs access to your health data to provide health-based features.</string>
```

**Important Notes:**
- The module does not include this key by default - it must be added to your host app's Info.plist.
- Customize the description string to explain why your app specifically needs health data access.
- HealthKit is **not available on iPad** (and other devices without HealthKit support). Use `Eitri.health.isAvailable()` from Bifrost to detect this at runtime.


## Registering health module

```swift
import EitriHealth

// [...]

let eitriMachineContext = EitriMachineInstanceManager.start()

// get main eitri-machine instance
let mainEitriMachine = eitriMachineContext.mainMachine

// configure
mainEitriMachine.configure(
  // configure params
)

//register modules
mainEitriMachine.modules.register(module: HealthModule())
```

## How to use the module in your eitri-apps

Bifrost-side API and platform notes (units, characteristics availability, etc.) are documented at the [Bifrost Health API reference](https://cdn.83io.com.br/library/eitri-bifrost/doc/latest/classes/_internal_.Health.html).

### Methods

Exposed under the `health` namespace.

| Method | Description |
|---|---|
| `isAvailable` | Returns `{ available, reason? }` based on `HKHealthStore.isHealthDataAvailable()`. |
| `getSupportedDataTypes` | Returns every `HealthDataType` the module knows about when HealthKit is available; empty otherwise. |
| `requestPermissions` | Shows HealthKit's single permissions sheet aggregating all requested identifiers. Accepts samples, characteristics and `"exercise"` in the same `read` array. Idempotent — when every requested identifier has already been decided, no prompt is shown. Resolves with no payload; Apple does not expose read authorization status, so apps detect access by calling the relevant read method and checking the result. |
| `readSamples` | Reads one page of raw samples for a single data type. `limit` is the per-page sample cap, clamped to `1..500`, default `100`. Paginate with `cursor`: pass the previous page's opaque `nextCursor` to fetch the next page; drive the loop on `nextCursor` (absent ⇒ exhausted), not on page size. Resolves with `samples: []` when the user has not granted access. |
| `readExercises` | Reads one page of exercises — every workout (`HKWorkout`) whose start instant falls in `[startDate, endDate)`. No type filter — caller narrows by inspecting `exercise.exerciseType`. Same pagination contract as `readSamples` (per-page `limit` clamped to `1..500`, default `100`, paginate via `cursor`/`nextCursor`). Requires `"exercise"` to have been requested via `requestPermissions`. |
| `getCharacteristics` | Reads static user attributes (`biologicalSex`, `dateOfBirth`, `bloodType`, `fitzpatrickSkinType`, `wheelchairUse`). Only the names passed in the input appear in the output. iOS-only — Android always returns `null` for every requested name. |

### `HealthExercise.exerciseType` values

Reference of values iOS may emit: [HealthKit `HKWorkoutActivityType`](https://developer.apple.com/documentation/healthkit/hkworkoutactivitytype).

Read sample values yourself by calling `readSamples` for `"distance"` / `"activeEnergyBurned"` / `"heartRate"` scoped to the exercise's `startDate`/`endDate`.
