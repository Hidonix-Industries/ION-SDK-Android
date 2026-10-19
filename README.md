# ION Android SDK Documentation

Welcome to the ION Android SDK. This document provides an overview of how to integrate and use the SDK in your Android applications.

## Table of Contents
- [Setup and Configuration](#setup-and-configuration)
- [Permissions](#permissions)
- [Core Integration](#core-integration)
    - [IONSdk Initialization](#ionsdk-initialization)
    - [License Features](#license-features)
    - [IONSdk Data API](#ionsdk-data-api)
    - [IONLocationManager](#ionlocationmanager)
    - [IonLocationListener](#ionlocationlistener)
- [Callback Overview](#callback-overview)
- [Positioning Availability](#positioning-availability)
- [Position Reliability and Calibration](#position-reliability-and-calibration)
- [Routing and Wayfinding](#routing-and-wayfinding)
- [AR-Assisted Navigation](#ar-assisted-navigation)
- [Diagnostics](#diagnostics)
- [Models Reference](#models-reference)
- [Migration Notes](#migration-notes)

Third-party software used by the SDK and its licenses are listed in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

---

## Setup and Configuration

Before you start using the SDK, you must configure your Android project and the ION CMS.

### 1. CMS Configuration

When creating your license in the ION CMS:

- Set the correct **Bundle Identifier** of your application
- The API Key is generated after successful license creation

The SDK validates the license against the provided bundle identifier.

---

### 2. Android Project Configuration

Requirements:

- `minSdk` **26**
- `compileSdk` **34** or higher
- Java 8 / Kotlin JVM target 1.8 or higher
- Kotlin **1.9** or higher (the SDK is built with Kotlin 2.0.21)
- ABIs `armeabi-v7a` and `arm64-v8a` (the positioning core ships native libraries)

Store your API key in `gradle.properties` to avoid exposing sensitive data:

```
ION_API_KEY=your_api_key
```

Expose it in your module `build.gradle`:

```
buildConfigField "String", "ION_API_KEY", "\"${ION_API_KEY}\""
```

Access it in your code:

```
val apiKey = BuildConfig.ION_API_KEY
```

Add the following dependencies inside your app module `build.gradle` file. The published artifact only brings in `kotlinx-coroutines-android` and `kotlinx-serialization-json` transitively; everything else must be declared by the app:

```
dependencies {

    // ION SDK
    implementation 'com.github.Hidonix-Industries:ION-SDK-Android:2.2.0'

    // Required dependencies
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.7.2")

    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.11.0")

    implementation("com.google.code.gson:gson:2.10.1")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
    implementation("com.google.android.gms:play-services-location:20.0.0")
    implementation("com.google.flatbuffers:flatbuffers-java:25.2.10")

    // BLE beacon scanning
    implementation("org.altbeacon:android-beacon-library:2.19.5")

    // Positioning models (neural dead reckoning, floor classifier)
    implementation("org.tensorflow:tensorflow-lite:2.17.0")

    // Only required if you use AR-assisted navigation (IONArPreview)
    implementation("com.google.ar:core:1.51.0")

}
```

In your `settings.gradle`, make sure `mavenCentral()` and `jitpack.io` are included:

```
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```


---

## Permissions

The SDK requires the following Android permissions to function correctly.

### Required Permissions

The SDK's own manifest already declares the permissions below, and manifest merging propagates them to your application. They are listed here so you know what your app ends up requesting:


    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
    <uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
    <uses-permission android:name="android.permission.NEARBY_WIFI_DEVICES" />
    <uses-permission android:name="android.permission.BLUETOOTH" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
    <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
    <uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
    <uses-permission android:name="android.permission.HIGH_SAMPLING_RATE_SENSORS" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.CAMERA" />

    <uses-feature android:name="android.hardware.camera.ar" android:required="false" />

If your app also needs the GPS hardware declared, add it yourself:

    <uses-feature android:name="android.hardware.location.gps" />

### ARCore is optional

The SDK declares `android.hardware.camera.ar` with `required="false"` and
`<meta-data android:name="com.google.ar.core" android:value="optional" />`.

This is deliberate: the Play Store will **not** filter out devices without AR support, and ARCore is **not** force-installed. The ARCore installation prompt is raised only by `IONArPreview.openSession()`, that is only when you have actually turned AR on. A device without ARCore keeps working on dead reckoning alone.

### Runtime Permissions

In addition to the manifest declarations, several permissions must be requested at runtime, before initializing or starting the SDK:

    ACCESS_FINE_LOCATION
    ACCESS_COARSE_LOCATION
    ACCESS_BACKGROUND_LOCATION
    BLUETOOTH / BLUETOOTH_ADMIN
    BLUETOOTH_SCAN / BLUETOOTH_CONNECT
    NEARBY_WIFI_DEVICES        (Android 13+)
    POST_NOTIFICATIONS         (Android 13+, only if you show SDK-driven notifications)
    CAMERA                     (only for AR-assisted navigation)

Failure to request and grant these permissions at runtime may result in limited or non-functional SDK behavior.

It is recommended to handle permission requests before SDK initialization and ensure that all required permissions are granted to guarantee full functionality.

### Checking permissions

The SDK cannot request permissions itself (a request needs an Activity), but it can tell you which ones are missing:

```
IONSdk.checkPermissions(context) { allGranted: Boolean, missing: List<String> ->
    if (!allGranted) {
        // request `missing` from your Activity
    }
}
```

It checks the permissions without which GPS and BLE scanning do not start: `ACCESS_FINE_LOCATION`, plus `BLUETOOTH_SCAN` and `BLUETOOTH_CONNECT` on Android 12+. `CAMERA` and `ACCESS_BACKGROUND_LOCATION` are optional and are not reported. It does not require `initialize()`, and the callback runs on the caller's thread.

---

## Core Integration

### IONSdk Initialization

Initialize the SDK once the runtime permissions have been requested (see [Runtime Permissions](#runtime-permissions)):

```
IONSdk.initialize(
    applicationContext,
    BuildConfig.ION_API_KEY,
    "YOUR_BUNDLE_IDENTIFIER"
) { isValid: Boolean, error: String? ->

    if (isValid) {
        // SDK ready
    } else {
        // Handle error
    }
}
```

#### Callback Parameters

- `isValid`: Indicates whether the license is valid
- `error`: Contains an error message if validation fails

#### Notes

- Initialization is **asynchronous**. Internally the SDK downloads its remote configuration, resolves the dynamic base URL and requests an access token
- Call `initialize()` only once per app lifecycle
- Always verify license validity before using SDK features: create the `IONLocationManager` only after the callback has returned `isValid == true`
- The cached access token is bound to the API key + bundle pair: if you change either, the SDK discards the old token and requests a new one

#### License re-check

```
IONSdk.checkLicense { isValid, error -> }
```

Forces a fresh token request and reports whether the license is still valid.

---

### License Features

A license enables a set of features. Read them with:

```
// Kotlin (suspend)
val features: List<IONLicenseFeature> = IONSdk.getLicenseFeatures()

// Callback variant, safe to call from Java
IONSdk.getLicenseFeatures { features, error -> }
```

`IONLicenseFeature`:

- `POI`
- `SAFETY`
- `WAYFINDING`
- `AR_NAVIGATION`
- `TURN_BY_TURN_AR` → route arrows on the floor in the camera view, in addition to the map
- `AR_ONLY` → the position comes from AR only (see [AR-only licenses](#ar-only-licenses))

Notes:

- An **empty list means no feature is enabled**, not "all of them": treat it as a denial
- Feature ids unknown to this SDK version are dropped instead of failing the call, so the backend can add features without breaking published apps
- Features are read from the same license validation that issues the token. If no validation has happened yet in this process (e.g. `initialize()` found a cached token), `getLicenseFeatures()` performs one
- The SDK itself enforces only `AR_ONLY`. The other features are for your app to gate its own UI

---

### IONSdk Data API

`IONSdk` also exposes the read-only content API. Unless stated otherwise these are Kotlin `suspend` functions and must be called from a coroutine.

#### Venues, buildings and floors

- `getMyLocations(): List<LocationRoot>` → all locations enabled for the license
- `getLocation(lat, lon): LocationRoot` → the location covering a geographical point
- `getBuilding(locationId, buildingId): Building`
- `getBuildingByBeacon(locationId, beaconUuid): Building` → resolves a building from a detected beacon UUID
- `getFloor(floorId): LevelInfo` → lightweight floor descriptor (name, image, rotation, georeference)

#### POIs

- `getNavigationPois(locationId): IONPoiContainer`
- `getStandardPoisByFloor(floorId, callback)` → callback-based, safe to call from Java
- `getStandardPoiDetails(poiId, callback)` → callback-based, full detail of a single POI

#### Geofences

- `getGeofencesByFloor(floorId): List<IONGeofence>`
- `getSystemGeofencesByFloor(floorId): List<IONGeofence>`
- `sendGeofenceEvent(GeofenceEventRequest): GeofenceEventResponse`

#### AR markers

- `getArMarkersByBuilding(buildingId): List<ARMarker>` → the printed markers surveyed for a building

#### License

- `getLicenseFeatures(): List<IONLicenseFeature>` → see [License Features](#license-features)

#### Telemetry

- `sendPosition(PositionRequest): Boolean` → returns true when the backend accepted the sample (HTTP 202)

---

### IONLocationManager

The `IONLocationManager` is the main entry point for location-based features.

It provides:

- Indoor positioning
- Floor detection
- Geofencing
- POI retrieval
- Routing and wayfinding
- AR-assisted navigation

#### Initialization

Create the manager only after `IONSdk.initialize()` has returned a valid license, and register the listener immediately after creating it:

```
IONSdk.initialize(applicationContext, BuildConfig.ION_API_KEY, "YOUR_BUNDLE_IDENTIFIER") { isValid, error ->
    if (isValid) {
        val ionLocationManager = IONLocationManager.create(applicationContext)
        ionLocationManager.setIonLocationListener(listener)
        ionLocationManager.startLocationUpdates()
    }
}
```

`IONLocationManager.create()` starts positioning as soon as the manager is created; a manager created before the license has been validated does not start.

#### Lifecycle

- `startLocationUpdates()` → kept for compatibility: positioning already starts in `create()`, so this call does not start anything by itself
- `stopLocationUpdates()` → stops tracking and detaches the listener. Stopping is final for that instance: to start tracking again, create a new `IONLocationManager`

#### Floors, geofences, routing

- `ChangeFloor(level: IONLevel)` → forces the active floor
- `addGeofences(notifications: List<IONGeofence>)` → registers geofences to be monitored
- `getWayfindingTo(poiId: Int, poiType: Int): Wayfinding` → distance and polylines to a POI
- `getRoute(startPoint, destination, destinationFloorId, accessibilityMode, waypoints, callback)` → full turn-by-turn route

#### Map

- `isWalkable(lat, lng): Boolean?` → whether the point falls inside the walkable area of the floor the SDK is currently positioning on. `null` means unknown (no floor or no walkable mask yet), which is not the same as "outside". Cheap enough to be called once per rendered frame, e.g. to draw the dot grey while it is outside the walkable area

#### AR

- `setArNavigationEnabled(enabled: Boolean)`
- `setArRole(role: IONArRole)`
- `attachArPreview(preview: IONArPreview?)`
- `updateArPosition(x, y, accuracyMeters, isTracking)`
- `setArRoute(route: Route?)` → draws the route as arrows on the floor in the camera view
- `requireArMarkerRescan(): Boolean` → pauses AR navigation until a new marker is scanned
- `setArRouteHint(position, floorId, bearingDeg)` / `clearArRouteHint()` → lets AR recover without a marker using the route being followed

See [AR-Assisted Navigation](#ar-assisted-navigation).

#### Calibration

- `getLastCalibrationGestureMs(): Long` → boot-clock instant (`SystemClock.elapsedRealtime`) of the last figure-of-eight calibration gesture the SDK recognised, or `0` if it has not seen one this session.

  It is a timestamp and not a boolean on purpose: the gesture lasts a couple of seconds, so the useful question is never "is it happening now" but "has one happened since the moment I care about" — a comparison only the caller can make.

- `revalidateCalibration()` → requires fresh calibration evidence. Call it when the app returns to the foreground: the magnetic environment may have changed while the app was in the background

---

### IonLocationListener

Implement `IonLocationListener` to receive real-time updates.

> **Breaking change.** The listener contract changed in this release: `onEnterVenue(IONVenue)` was replaced by `onEnterBuilding(Building, IONLevel)`, `onCalibrationNeeded()` now carries a percentage and a level, and two callbacks were added (`onLocationDetected`, `onIndoorPositionReliabilityChanged`). See [Migration Notes](#migration-notes).

#### Example

```
ionLocationManager.setIonLocationListener(object : IonLocationListener {

    override fun onLocationChanged(location: IONLocation) {}

    override fun onLevelChanged(level: Int, georeference: MutableList<GeoCoordinate>?) {}

    override fun onHeadingChanged(heading: Double) {}

    override fun onEnterBuilding(building: Building?, outdoorLevel: IONLevel?) {}

    override fun onExitVenue() {}

    override fun onExitBuilding() {}

    override fun onLocationDetected(location: LocationRoot?) {}

    override fun onGeofenceEvent(
        response: GeofenceEventResponse?,
        event: IONGeofence.GeofenceEvent?
    ) {}

    override fun onGeofencesLoaded(geofences: MutableList<IONGeofence>?) {}

    override fun onPoisForFloorChanged(level: IONLevel?) {}

    override fun onPoisForLocationLoaded(
        standard: MutableList<IONStandardPOI>?,
        service: MutableList<IONServicePOI>?
    ) {}

    override fun onIndoorPositionReliabilityChanged(
        reliability: IndoorPositionReliability
    ) {}

    override fun onCalibrationNeeded(pct: Int, level: CalibrationLevel) {}

    override fun resetCalibration() {}

    // Optional: these callbacks have a default implementation

    override fun onPositioningAvailabilityChanged(availability: PositioningAvailability) {}

    override fun onCalibrationStatus(status: CalibrationStatus) {}

    override fun onNavigationMotionBlockChanged(blocked: Boolean) {}

    override fun onNavigationMotionRecoveryChanged(waitingForWalking: Boolean) {}

})

ionLocationManager.startLocationUpdates()
```

`IndoorPositionReliability` is imported from `com.hidonix.ion.sdk.internal`; every other type comes from `com.hidonix.ion.sdk.api`.

---

## Callback Overview

### Position & Heading

- `onLocationChanged(location)` → Called when a new position is available
- `onHeadingChanged(heading)` → Called when device orientation changes

### Floor & Building

- `onLevelChanged(level, georeference)` → Floor change detected
- `onLocationDetected(location)` → The venue (`LocationRoot`) the user is in has been resolved
- `onEnterBuilding(building, outdoorLevel)` → Entered a building; `outdoorLevel` is the synthetic level used outdoors
- `onExitVenue()` → Exited the venue
- `onExitBuilding()` → Exited the building

### Availability

- `onPositioningAvailabilityChanged(availability)` → Whether the SDK can position the user at all (optional). See [Positioning Availability](#positioning-availability)

### Geofencing

- `onGeofenceEvent(response, event)` → Geofence triggered
- `onGeofencesLoaded(geofences)` → Geofence data loaded

### POI

- `onPoisForFloorChanged(level)` → POIs updated for current floor
- `onPoisForLocationLoaded(standard, service)` → Nearby POIs loaded

### Sensors & Reliability

- `onIndoorPositionReliabilityChanged(reliability)` → The estimator's own confidence changed
- `onCalibrationNeeded(pct, level)` → Sensor calibration required. `pct == -1` means "verifying / absent / stale": never read it as a score
- `onCalibrationStatus(status)` → Full calibration state (optional). Its default implementation forwards to `onCalibrationNeeded`
- `resetCalibration()` → Calibration reset requested

### Phone handling

All optional, delivered on the main thread:

- `onNavigationMotionBlockChanged(blocked)` → The way the phone is being held or moved prevents the position from being shown. The default implementation calls `onNavigationMotionBlocked()` when `blocked` becomes true
- `onNavigationMotionBlocked()` → Called once per episode, e.g. to give haptic feedback. The SDK never triggers any hardware action by itself
- `onNavigationMotionRecoveryChanged(waitingForWalking)` → Keep your "hold the phone correctly" hint visible while this is true: the block has ended but the SDK is waiting for the user to start walking again

---

## Positioning Availability

`onPositioningAvailabilityChanged` tells whether the SDK is able to position the user in one of the license's locations, and if not, why. It does not judge the indoor position — that is what [IndoorPositionReliability](#indoorpositionreliability) is for. Same API as the iOS SDK.

`PositioningAvailability` is a sealed interface:

- `PositioningAvailability.Determining` → not known yet: the first fix, or the backend's answer about which location contains it, is still missing. It stays here while those requests fail (no network, server error)
- `PositioningAvailability.Available` → permission granted and location found: positioning has started
- `PositioningAvailability.Unavailable(reason: PositioningUnavailableReason)`
- `isAvailable: Boolean` → convenience accessor

`PositioningUnavailableReason`:

- `LOCATION_PERMISSION_DENIED` → neither FINE nor COARSE location granted, or location disabled for the whole device
- `LOCATION_PERMISSION_RESTRICTED` → iOS only, never emitted on Android (it exists to keep the enum identical)
- `NO_LOCATION_NEARBY` → there is a fix, but the backend knows no location near it. Not final: the SDK asks again on every new fix

Delivery rules:

- Called on the main thread
- Called once with `Determining` right after a session starts with a valid license (never with an invalid one)
- `Available` always comes before `onLocationDetected`
- A newly set listener immediately receives the current value

---

## Position Reliability and Calibration

### IndoorPositionReliability

`onIndoorPositionReliabilityChanged` publishes whether the indoor position can be trusted right now. It is a sealed interface (package `com.hidonix.ion.sdk.internal`):

- `IndoorPositionReliability.Reliable`
- `IndoorPositionReliability.NotReliable(reason: IndoorPositionUnreliableReason)`
- `isReliable: Boolean` → convenience accessor

`IndoorPositionUnreliableReason`:

- `WAITING_FOR_INDOOR_POSITIONING` → indoor positioning has not taken over yet
- `WAITING_FOR_USER_TO_WALK` → the estimator needs motion before it can converge
- `FLOOR_TRANSITION` → a floor change is in progress
- `DEGRADED` → the estimate has lost quality
- `CONVERGING` → the filter is converging

The intended use is UI: show a "locating…" state instead of a puck that is not yet meaningful.

### CalibrationLevel

`onCalibrationNeeded(pct, level)` reports the magnetic calibration quality as a percentage plus a synthetic level:

- `CalibrationLevel.GOOD`
- `CalibrationLevel.WARNING`
- `CalibrationLevel.BAD`

`pct` is `-1` while the calibration is being verified, is absent, or is stale.

Use `IONLocationManager.getLastCalibrationGestureMs()` to know whether the user has actually performed the figure-of-eight gesture since you asked them to.

### CalibrationStatus

`onCalibrationStatus(status)` carries the full state, with the same trust decision the navigation uses:

- `percentage: Int?` → quality score, `null` when absent, stale or not yet verified. It is a quality score, not an absolute heading accuracy
- `issue: CalibrationIssue` → why the calibration is in this state
- `trusted: Boolean` → usable for navigation. Even a stable compass can have a constant magnetic bias
- `calibrating: Boolean` → presentation only: a recognised gesture is being checked, before the percentage rises
- `headingTrusted: Boolean` → navigation can remain usable while the relative orientation confidence is reduced
- `orientationDisagreement: Boolean` → the orientation sources disagree. It detects uncertainty; it does not tell which sensor is right

`CalibrationIssue`: `VERIFYING`, `NONE`, `INTERFERENCE`, `CALIBRATION_REQUIRED`, `RECOVERING`, `ORIENTATION_CHECK`, `ORIENTATION_UNCERTAIN`.

If you override `onCalibrationStatus`, `onCalibrationNeeded` is no longer called for you: the default implementation of `onCalibrationStatus` is what forwards to it.

---

## Routing and Wayfinding

### Simple wayfinding

```
val wayfinding = ionLocationManager.getWayfindingTo(poiId, poiType)
// wayfinding.totalDistance
// wayfinding.polylines : Map<floorId, List<GeoCoordinate>>
```

### Turn-by-turn route

```
ionLocationManager.getRoute(
    startPoint,               // nullable: null means "use the current position"
    destination,
    destinationFloorId,
    accessibilityMode,        // true avoids stairs and escalators
    waypoints,                // List<Waypoint>, may be empty
    object : RouteCallback {
        override fun onSuccess(result: Route?) {}
        override fun onError(error: Throwable?) {}
        override fun onNoRouteFound() {}
    }
)
```

`Route` exposes two helpers for drawing:

- `getAllPaths(): List<List<GeoCoordinate>>` → one polyline per `LineString` feature
- `getFlattenedPath(): List<GeoCoordinate>` → all of them concatenated

### Following a route

Two helpers are available to build the navigation UI on top of a `Route`.

**`RouteProgressTracker`** projects the user's position onto the route. It is stateful and floor-aware: keep one per navigation session and call `reset()` when the session ends.

```
val tracker = RouteProgressTracker()

val match = tracker.match(route, floorId, position)   // lookaheadMeters defaults to 4.0
// match?.position          → the position snapped onto the route
// match?.crossTrackMeters  → distance from the route
// match?.remainingMeters   → distance still to walk
// match?.alongMeters       → distance already walked
// match?.bearingDegrees    → route direction at that point
// match?.featureIndex / segmentIndex
```

After a floor change, progress restarts from where it was on the same route, not from the start of the path.

**`RouteDeviationGate`** decides when to recompute the route, so a single noisy fix does not trigger a reroute:

```
val gate = RouteDeviationGate()

if (gate.shouldReroute(match?.crossTrackMeters, location.accuracy, usable, SystemClock.elapsedRealtime())) {
    // request a new route
}
```

It returns true only when the user has been off route for at least 3 seconds, by more than 5 m (or 1.5× the accuracy, whichever is larger), with an accuracy of 5 m or better, and no more than once every 10 seconds. Feed it unsnapped, fresh positions. Call `reset()` when navigation ends.

---

## AR-Assisted Navigation

AR-assisted navigation lets the SDK use ARCore's visual-inertial odometry and printed AR markers ("augmented images") to improve the indoor position. It is **disabled by default** and entirely optional.

### 1. Enable the AR channel

```
ionLocationManager.setArNavigationEnabled(true)   // call right after create(), before startLocationUpdates()
```

When enabled, the SDK accepts AR marker fixes as **seeds of the particle filter's prior** and can use ARCore's odometry as the transition model. When disabled, AR fixes are ignored outright and the pipeline behaves exactly as it does without any AR at all.

This flag decides whether the AR channel *exists* — it is not a choice of which odometry source drives the position at a given instant. The neural dead reckoning stays alive as a reserve for the whole session, and ownership moves between the two on the pipeline's own cycle.

Enabling AR does not make ARCore a requirement of your app: a device without ARCore support simply never produces fixes.

### 2. Choose the AR role

```
ionLocationManager.setArRole(IONArRole.MOTION)       // default
ionLocationManager.setArRole(IONArRole.POSITION)
```

`IONArRole` decides what ARCore does *inside* the estimator. It is orthogonal to `setArNavigationEnabled`: with AR navigation on, the starting floor is declared by a scanned marker and by nothing else, regardless of the role.

- **`MOTION`** — ARCore is the transition model: it provides distance travelled and direction of travel, and the neural dead reckoning falls back to reserve. This is the historical behaviour and what you get by calling only `setArNavigationEnabled(true)`.
- **`POSITION`** — the neural dead reckoning stays the transition model and the VIO position enters as a high-weight observation. The displayed position follows the VIO closely because the observation dominates the likelihood, not because the prediction comes from it — the two are kept separate on purpose, since the same source in both prediction and correction would make the filter more confident than the data allows.

  Practical consequence: in this role there is **no relay between sources**. If the camera is covered or tracking is lost, the observation switches off and navigation is the classic one; getting back in requires scanning a new marker.

`setArRole` can be called before or after `startLocationUpdates()`; it takes effect on the next filter cycle.

### 3. Show the camera preview

`IONArPreview` is a `FrameLayout` (not a Composable, so it works with and without Compose — wrap it in `AndroidView { }` if you use Compose). It owns the ARCore session, because the session has to be pumped from the GL thread that the view itself runs.

```
val preview = IONArPreview(context)
// add it to your layout, then:

if (preview.openSession(activity)) {
    preview.loadMarkers(buildingId, object : IONArPreview.MarkerLoadListener {
        override fun onMarkersReady(usable: Int, declared: Int, missingImages: Int) {}
        override fun onMarkersUnavailable(reason: String) {}
    })
    ionLocationManager.attachArPreview(preview)
}
```

Attach the preview **after** `openSession()` and **after** `setArNavigationEnabled(true)`. Detach by passing `null` before releasing it.

#### Lifecycle

Forward the Activity (or Fragment) lifecycle callbacks to the preview:

```
override fun onResume()  { super.onResume();  preview.onResume() }
override fun onPause()   { preview.onPause();  super.onPause() }
override fun onDestroy() { ionLocationManager.attachArPreview(null); preview.release(); super.onDestroy() }
```

`release()` is mandatory, not optional: ARCore allows several constructed `Session` objects but not two in the resumed state, so a session left alive prevents the next one from starting.

#### Border colour

The preview draws a diagnostic border, because a covered lens or a lost tracking would otherwise be invisible failures:

- orange — camera covered or tracking lost
- green — anchored to a marker
- blue — anchored to the filter (warm-up)
- grey — tracking, but not anchored yet

#### Inspection

- `session(): Session?` → the underlying ARCore session
- `isTracking(): Boolean` → ARCore is tracking, so a framed marker can now be recognised. Opening the session is not enough: recognition needs tracking, which starts a few frames after the view gets its surface. Asking the user to frame a marker before this is asking for a scan that cannot work
- `markersIndexed(): Int` → markers actually indexed in ARCore's image database. Zero means no scan can ever succeed — images arrive over the network, so it stays zero until the download and the session configuration are done
- `setArState(state: ArState)` → updates the diagnostic border, callable from any thread

#### Callbacks

Properties you can assign on the preview. They are invoked from the GL thread: switch to the main thread before touching the UI.

- `onFrame: ((Frame) -> Unit)?` → per-frame consumer
- `onScanResult: ((ScanResult, markerId: Int) -> Unit)?` → outcome of each marker scan, so the user knows why nothing happened:
  - `ACCEPTED` → the scan was accepted and the position is anchored
  - `UNSTABLE_ORIENTATION` → the marker is visible but its pose is not steady enough — typically it is framed at too sharp an angle, or the phone is moving. Ask the user to stand in front of it
- `onRecoveryStatus: ((ArRecoveryStatus) -> Unit)?` → see [Recovery](#7-recovery). Replayed after the preview is re-attached
- `onRouteGuidance: ((ArRouteGuidance) -> Unit)?` → see [Route arrows](#6-route-arrows). Called only when the value changes
- `onAnchorLost: ((Boolean) -> Unit)?` → compatibility callback, equivalent to `ArRecoveryStatus.required`. Prefer `onRecoveryStatus`

### 4. Marker loading

```
preview.loadMarkers(buildingId, listener)
```

Downloads the building's markers and rebuilds the session's image database. The sequence is mandatory and cannot be shortened: catalogue from the server, images downloaded, database built, `session.configure`, `resume`. Images cannot be added to an already-configured database without reconfiguring, so the time between opening the screen and the first possible fix is network time. Call it after `openSession()`, and again after a floor or building change. It does not block.

`onMarkersReady(usable, declared, missingImages)` — a `missingImages` greater than zero is not a detail: those markers will **never** be recognised, and ARCore starts anyway, so the only symptom would be a fix that never arrives.

Individual images can also be rejected by ARCore when they carry too few features; that is a problem with the marker, not with the session, and it is logged per marker.

### 5. Feeding fixes manually

If you run your own AR pipeline instead of `IONArPreview`, push fixes with:

```
ionLocationManager.updateArPosition(x, y, accuracyMeters, isTracking)
```

- `x`, `y` — floor-local **metres**, in the current floor's coordinate system
- `accuracyMeters` — the fix's own uncertainty; it becomes the seed's sigma, subject to a floor, because a fix is never exact
- `isTracking` — pass `false` as soon as the marker is lost (x/y/accuracy are then ignored). Doing so is what arms the next seed: the following fix is treated as new

The fix is consumed as a **seed of the particle filter's prior**, not as a per-cycle observation. Seeds are rare: taken on the *rising edge* of tracking and at most once every few metres walked. Calling this at the AR session's frame rate is expected and safe — it will not produce one seed per frame.

The reason is statistical. After a fix, the AR pose is "fix plus integrated displacement", so all the new absolute information is in the instant of the fix. Feeding it to the likelihood every cycle would count one measurement as as many independent witnesses as there are cycles, and the filter would collapse with a confidence it has not earned. Between fixes, an AR source's authority is over *displacement* only.

A fix is ignored — silently, by design — when it is older than the pipeline's freshness limit, when its accuracy is worse than the SDK's reliability threshold, when too few metres have been walked since the previous seed, or when it falls outside the floor's walkable area.

> **IMPORTANT.** `(x, y)` must already be expressed in the same floor-local metre coordinate system the SDK uses internally for the current floor — i.e. the markers' known positions must be surveyed against the venue's real floor plan, not an arbitrary local system. Mismatched coordinate systems silently pull the position toward the wrong place rather than failing loudly.

### 6. Route arrows

```
ionLocationManager.setArRoute(route)   // when navigation starts and whenever the route is recomputed
ionLocationManager.setArRoute(null)    // when navigation ends
```

Draws the route as arrows on the floor in the attached `IONArPreview`. Only the part on the current floor is drawn, from the user's position up to about 12 m ahead or shortly after the next turn, and only while AR is anchored and not in recovery. The floor is assumed to be about 1.35 m below the phone. Safe to call from any thread.

`preview.onRouteGuidance` tells you where the route is relative to what the camera is framing, so you can prompt the user:

- `ArRouteGuidance.HIDDEN` → no arrows on screen: no route, route on another floor, off route, or AR not anchored
- `ArRouteGuidance.IN_VIEW` → the route ahead is in the camera's view
- `ArRouteGuidance.LOOK_LEFT` / `LOOK_RIGHT` → turn the phone left / right to see it

### 7. Recovery

When the AR reference is lost, the SDK enters **recovery**: while `ArRecoveryStatus.required` is true, **no position is published** and no geofence is checked. Recovery ends when a new marker fix is successfully applied while AR is usable.

`preview.onRecoveryStatus` reports an `ArRecoveryStatus`:

- `required` → recovery in progress: ask the user to scan a marker
- `markerScanned` → a marker has been scanned
- `calibrationComplete` → informational only, it is no longer a condition for ending recovery

#### Pausing when the app goes to the background

```
val paused = ionLocationManager.requireArMarkerRescan()
```

With the camera off the AR pose is not followed, and when ARCore resumes its world frame is not guaranteed to match the one the position was anchored to. This call puts navigation in recovery, exactly as if the SDK had started it, until a new marker is scanned. It returns `false` when there was nothing to pause (AR navigation disabled, or no position shown yet). The status is delivered by the positioning loop on its next cycle, not from inside the call. Safe to call from any thread.

#### Recovering without a marker

```
ionLocationManager.setArRouteHint(snappedPosition, floorId, bearingDeg)
ionLocationManager.clearArRouteHint()
```

While navigating and while AR is **not** guiding, pass on every fix where the app places the user on the route (`snappedPosition`, e.g. `RouteProgressTracker.Match.position`) and the route direction there (`bearingDeg`, degrees clockwise from north). Typical case: a lift ride, where ARCore loses tracking. Once the user has walked a few metres along a straight stretch, the route gives the position and the route direction versus the walk measured by ARCore gives the alignment, so AR is put back in service without a marker. It is also used when AR is anchored but too uncertain to guide. The hint is ignored while AR is guiding, or on a floor other than the current one. Call `clearArRouteHint()` when navigation ends or when you have no reliable hint.

### AR-only licenses

With the `AR_ONLY` license feature and AR navigation enabled, the position comes from AR alone: there is no fallback to the filter (dead reckoning, beacons, magnetic field, Wi-Fi). When AR is lost the dot stops and the user is asked to frame a marker. The floor is changed only by markers and by the barometer, not by beacons or the floor classifier.

---

## Diagnostics

```
val report: String = NavigationDiagnostics.read(context)
```

Returns a JSON report of the resources actually loaded in the last navigation session (location, building, floors, models), kept across process restarts. Useful to attach to bug reports. Returns `"No navigation session recorded."` if there is none.

---

## Models Reference

The following sections provide descriptions for the public models used across the ION SDK, grouped by their respective categories.

### Geofence Models

- **IONGeofence**: A single geofence, indoor or system-level. Indoor areas are created as *"Security Geofence"* in the ION CMS, system areas as *"System Area"*.
  - `id`: Unique identifier.
  - `title` / `msg`: Title and message associated with the trigger.
  - `triggerEvents`: List of `GeofenceEvent` that activate this geofence.
  - `dwellTime`: Time in seconds required for a dwell trigger.
  - `category`: `1` crowd management, `2` restricted, `3` system.
  - `systemGeofenceType` / `systemGeofenceTypeName`: Type of transition for system areas.
  - `systemGeofenceUse` / `systemGeofenceUseName`: Intended use for system areas.
  - `triggerNotMappedBeacon`: UUID of the beacon triggering unmapped area entry.
  - `nextBuildingId` / `nextFloorId`: Target IDs for transitions.
  - `geofence`: List of `GeoCoordinate` defining the boundary.
  - `relatedGeofence`: Optional link to another `IONGeofence`.
- **IONGeofence.GeofenceEvent**: `ON_ENTER`, `ON_EXIT`, `ON_DWELL`.
- **GeofenceEventRequest**: Event to report to the backend.
  - `geofenceId`, `eventType`.
- **GeofenceEventResponse**: Contains status info when a geofence trigger occurs.
  - `responseStatus`: `1` OK, `2` NOT_AUTHORIZED, `3` OVER_CAPACITY.
  - `responseStatusName`: Human-readable status name.
  - `title` / `msg`: Response text.

### IONPOI (Point of Interest) Models

- **IONPOI**: Sealed base class for all POI types.
  - `id`, `name`: Basic POI info.
  - `type` / `category`: Type and category codes.
  - `imageUrl`: URL for the POI image.
  - `latitude` / `longitude`: Physical coordinates.
  - `floorMapId` / `floorNumber`: Floor identification.
  - `buildingId` / `buildingName`: Building identification.
  - `getTypeName()` / `getCategoryName()`: Resolve the numeric codes into the venue's own labels, using the typology tables the SDK downloads at startup.

- **IONStandardPOI**: A user-facing POI. Everything in `IONPOI`, plus:
  - `description`, `website`, `status`.
  - `poiLocation`: Detailed address information (`IONStandardPOI.PoiLocation`).
  - `webSocialLinks`: Social media profile links (`IONStandardPOI.WebSocialLinks`).
  - `imageURLs`: List of URLs for POI images.
  - `hoursOfOperation`: Weekly schedule (`IONStandardPOI.HoursOfOperation`).

- **IONServicePOI**: A service POI (lifts, stairs, exits, …). Same fields as `IONPOI`.

- **PoiLocation**: `address`, `addressDetails`, `city`, `country`, `zipCode`.
- **WebSocialLinks**: `instagram`, `facebook`, `yelp`, `linkedIn`, `x`, `other`.
- **HoursOfOperation**: Operational status for a specific day.
  - `daySlug`: String identifier for the day.
  - `isOpen`: Boolean status.
  - `timeSlots`: List of `TimeSlot` segments.
- **TimeSlot**: A time range (`from` / `to`).

### Positioning Models

The positioning models follow a hierarchical structure, from the overall site location down to the individual sensors (beacons).

- **IONLocation**: A published position.
  - `latitude` / `longitude`, `floorLevel`, `accuracy`, `time`, `provider`.
  - `getElapsedRealtimeNanos()`: Boot-clock timestamp of the position.
  - `getQuality()`: `IONLocation.Quality` — how fresh and how absolute the estimate is: `TRACKING`, `DEAD_RECKONING`, `FLOOR_TRANSITION`, `STALE`.
  - `getSource()`: `IONLocation.Source` — which source produced the *motion* of this position: `FILTER`, `DEAD_RECKONING`, `ARCORE`, or `UNKNOWN` for positions that do not go through the indoor engine (GPS, outdoor dead reckoning; treat them as "not AR"). It is a different question from the quality: a position can be driven by ARCore and be `STALE` at the same time. Use it, for example, to change the dot colour when AR navigation hands over between ARCore and the fallback.
  - `isMaskConstrained()`: The indoor coordinate and the continuous segment leading to it have passed the walkable-area check.
  - `isPositionDiscontinuous()`: The position is a relocation: display it directly, never animate through the wall in between.
  - `toLocation()`: Converts to an Android `Location`. It also stamps `elapsedRealtimeNanos`, which is the clock that consumers use to decide how *old* a position is — without it a fresh position can be rendered as stale.

- **LocationRoot**: The top-level model representing a site.
  - `name`, `id`, `address`.
  - `coordinate`: Geographical center (`GeoCoordinate`).
  - `geofence_radius`: The radius in meters used for geographical geofencing triggers.
  - `light_buildings`: A list of `Building` objects belonging to this location.

- **Building**: Represents a full physical structure within a location.
  - `name`, `id`, `address`.
  - `beacons_base_uuid` / `beacons_base_uuids`: Base UUIDs for beacons in this building.
  - `dict_major_beacons`: Map of beacon major values to floors.
  - `ground_level`: The floor number designated as ground level.
  - `levels`: List of `Level` objects representing the floors.
  - `outdoorMap`: Optional `OutdoorMap` for the outdoor area.
  - `ota` / `ia`: Boundary coordinates of the Outdoor Transition Area and Indoor Area.
  - `floor_tcn_model_url` / `floor_tcn_metadata_url`: Floor-classifier model for the building.

- **Level**: Represents a specific floor within a building.
  - `id`, `name`, `number`: Identity and floor index (e.g. 0 for ground floor).
  - `height`: Altitude of the floor in meters.
  - `magDeclination`: Magnetic declination at this floor.
  - `mapUrl` / `flatMapUrl`: JSON and FlatBuffer sources of the floor's positioning data.
  - `map`: A `LevelMap` object containing the floorplan and positioning details.
  - `particle_filter_config`: Per-floor tuning of the estimator.
  - `standardPois` / `servicePois`: Lists of POIs available on this floor.
  - `indoor_geofences`: Geofences associated with the floor.
  - `mapName`, `mapLastUpdate`: Internal map name and timestamp of the last update.
  - `loadMapNew(context, downloadGeofences)`: Downloads and parses the floor's FlatBuffer. The result is memoised for the session, so the same floor is not downloaded twice; failures are not memoised. A download is also shared for one minute between different `Level` objects of the same floor (e.g. two `getBuilding` calls), as long as `mapLastUpdate` matches.

- **LevelInfo**: Lightweight floor descriptor returned by `IONSdk.getFloor`.
  - `id`, `number`, `name`, `buildingId`, `buildingName`, `imageUrl`, `rotation`, `georeference`.

- **IONLevel**: The floor as published to the app by the listener callbacks.
  - `id`, `floorLevel`, `floorName`, `mapName`, `imageUrl`.
  - `width` / `height`, `rotation`, `altitude`, `georeferences`.
  - `getGeofences()`, `getStandardPois()`, `getServicePOIS()`.

- **LevelMap**: Contains technical specifications for a floor's positioning.
  - `map_floor_url`, `floorplan_svg`, `mask_svg` / `mask_path` / `mask_url`: Floorplan and walkable-area sources.
  - `width` / `height`: Dimensions of the map in meters.
  - `rotation`: Map rotation relative to geographical north.
  - `georeference`: Reference points for alignment (`List<GeoCoordinate>`).
  - `beacons`: List of `Beacon` objects deployed on this floor.
  - `wifis` / `wifi_rssi`: Access point identifiers and their reference RSSI, parallel by index.
  - `sparse_map`, `map_size_x/y/h`, `feature_size`, `spatial_granularity`, `heading_quantum`, `magnetic_range`: The magnetic fingerprint and its discretisation.
  - `tcn_model_url` / `tcn_metadata_url`, `floor_tcn_model_url` / `floor_tcn_metadata_url`: Neural models for position and floor.

- **OutdoorMap**: The outdoor counterpart of `LevelMap`.
  - `floorplan_png`, `width`, `height`, `rotation`, `georeference`.

- **Beacon**: Represents a BLE sensor used for indoor positioning.
  - `uuid`, `major`, `minor`: Proximity UUID and identification numbers.

- **GeoCoordinate**: A basic structure for geographical positioning.
  - `lat`: Latitude.
  - `lng`: Longitude.

- **PositionRequest**: A position sample sent to the backend.
  - `timestamp`, `lat`, `lng`, `heading`, `floorId`, `buildingId`, `locationId`, `batteryLevel`.

### Availability Models

- **PositioningAvailability**: Sealed interface — `Determining`, `Available`, or `Unavailable(reason)`, plus `isAvailable`.
- **PositioningUnavailableReason**: `LOCATION_PERMISSION_DENIED`, `LOCATION_PERMISSION_RESTRICTED` (iOS only), `NO_LOCATION_NEARBY`.

### License Models

- **IONLicenseFeature**: `POI`, `SAFETY`, `WAYFINDING`, `AR_NAVIGATION`, `TURN_BY_TURN_AR`, `AR_ONLY`.
  - `id`: The integer the backend uses for the feature.
  - `fromId(id)`: Resolves an id, or returns `null` if this SDK version does not know it.

### AR Models

- **ARMarker**: A printed AR marker (augmented image) registered for a building.
  - `id`: Unique identifier. It is also the name under which the image is indexed in ARCore, so it is what comes back in `AugmentedImage.name`.
  - `imageUrl`: URL of the marker image.
  - `physicalWidth`: Real-world width of the printed marker, in meters.
  - `x` / `y`: Floor-local **meters** in `floorId`'s own coordinate system — the same one the SDK's indoor position pipeline uses internally, not an arbitrary local system. The origin is bottom-left, with +y pointing toward the floor plan's top edge.
  - `heading`: The marker's orientation relative to that floor's local axes, in **radians**, in the server's convention: zero on the floor's +x axis, **counter-clockwise**, range `[0, 2π)`. The server normalises, and the value in the response is the authoritative one.

    This is **not** the compass convention the SDK's AR code works in (zero on +y, clockwise). The conversion between the two is a reflection: `ours = π/2 - theirs`.
  - `floorId`: The floor the marker belongs to.

- **IONArRole**: `MOTION` or `POSITION`. See [Choose the AR role](#2-choose-the-ar-role).
- **IONArPreview.ArState**: `NO_TRACKING`, `TRACKING`, `ANCHORED_FILTER`, `ANCHORED_MARKER`.
- **IONArPreview.ScanResult**: `ACCEPTED`, `UNSTABLE_ORIENTATION`.
- **IONArPreview.MarkerLoadListener**: `onMarkersReady(usable, declared, missingImages)`, `onMarkersUnavailable(reason)`.
- **ArRecoveryStatus**: `required`, `markerScanned`, `calibrationComplete`. See [Recovery](#7-recovery).
- **ArRouteGuidance**: `HIDDEN`, `IN_VIEW`, `LOOK_LEFT`, `LOOK_RIGHT`. See [Route arrows](#6-route-arrows).

### Reliability and Calibration Models

- **IndoorPositionReliability** (`com.hidonix.ion.sdk.internal`): Sealed interface — `Reliable`, or `NotReliable(reason)`, plus `isReliable`.
- **IndoorPositionUnreliableReason**: `WAITING_FOR_INDOOR_POSITIONING`, `WAITING_FOR_USER_TO_WALK`, `FLOOR_TRANSITION`, `DEGRADED`, `CONVERGING`.
- **CalibrationLevel**: `GOOD`, `WARNING`, `BAD`.
- **CalibrationStatus**: `percentage`, `issue`, `trusted`, `calibrating`, `headingTrusted`, `orientationDisagreement`. See [CalibrationStatus](#calibrationstatus).
- **CalibrationIssue**: `VERIFYING`, `NONE`, `INTERFERENCE`, `CALIBRATION_REQUIRED`, `RECOVERING`, `ORIENTATION_CHECK`, `ORIENTATION_UNCERTAIN`.

### Routing Models

- **Route**: Represents a computed path following a GeoJSON-like structure.
  - `type`: The type of the route collection.
  - `features`: An array of `Feature` objects, each representing a segment of the route.
  - `meta`: Metadata about the entire route.
  - `getAllPaths()` / `getFlattenedPath()`: Convenience accessors for drawing.
- **Feature**: A segment of a route.
  - `type`: The type of the feature.
  - `geometry`: The geographical shape of the segment (`Geometry`).
  - `properties`: Custom attributes for the segment (`Properties`).
- **Geometry**: The spatial component of a route segment.
  - `type`: The geometry type (`"LineString"` or `"Point"`).
  - `coordinates`: Raw coordinates; use `asPoint()` and `asLineString()` instead of casting them yourself.
- **Properties**: Navigation attributes for a route segment.
  - `instructionType`: The type of navigation instruction (`NavInstructionType`).
  - `instruction`: Human-readable navigation instruction.
  - `distanceMeters`: Optional length of the segment in meters.
  - `floorMapId`: ID of the floor map where this segment is located.
  - `floorFrom` / `floorTo`: Optional floor names for transitions.
  - `connectorSubType`: Optional type of transition connector (`ConnectorSubType`).
  - `waypointIndex`: Optional index of the associated waypoint.
- **Meta**: Summary information for the route.
  - `estimatedTimeSeconds`: Total estimated time.
  - `totalDistanceMeters`: Total distance.
  - `floorChanges`: Number of floor changes.
  - `floorMapIds`: List of all floor map IDs involved.
- **Waypoint**: Represents a point on a route.
  - `coordinate`: Geographical position (`GeoCoordinate`).
  - `floorId`: Unique identifier of the level.
- **Wayfinding**: The lightweight result of `getWayfindingTo`.
  - `totalDistance`: Total distance in meters.
  - `polylines`: Map of floor ID to the polyline on that floor.
- **RouteCallback**: `onSuccess(route)`, `onError(throwable)`, `onNoRouteFound()`.
- **RouteProgressTracker** / **RouteProgressTracker.Match**: See [Following a route](#following-a-route).
- **RouteDeviationGate**: See [Following a route](#following-a-route).

#### Routing Enums

- **NavInstructionType**: `Walk`, `TurnLeft`, `TurnRight`, `TakeElevator`, `TakeStairs`, `TakeEscalator`, `TakeRamp`, `TakeCrossBuilding`, `Arrive`, `Waypoint`.
- **ConnectorSubType**: `Stairs`, `Elevator`, `Escalator`, `Ramp`, `CrossBuilding`.

---

## Migration Notes

If you are updating from a previous version, the following public API changes require action.

### IonLocationListener

| Before | Now |
| --- | --- |
| `onEnterVenue(IONVenue venue)` | `onEnterBuilding(Building building, IONLevel outdoorLevel)` |
| `onCalibrationNeeded()` | `onCalibrationNeeded(int pct, CalibrationLevel level)` |
| — | `onLocationDetected(LocationRoot location)` *(new)* |
| — | `onIndoorPositionReliabilityChanged(IndoorPositionReliability reliability)` *(new)* |
| — | `onPositioningAvailabilityChanged(PositioningAvailability availability)` *(new, optional)* |
| — | `onCalibrationStatus(CalibrationStatus status)` *(new, optional)* |
| — | `onNavigationMotionBlocked()`, `onNavigationMotionBlockChanged(boolean)`, `onNavigationMotionRecoveryChanged(boolean)` *(new, optional)* |

`onLevelChanged(int level, List<GeoCoordinate> georeference)`, `onExitVenue()`, `onExitBuilding()`, `onGeofenceEvent`, `onGeofencesLoaded`, `onPoisForFloorChanged`, `onPoisForLocationLoaded` and `resetCalibration()` are unchanged.

The callbacks marked *optional* have a default implementation, so existing listeners keep compiling.

`onCalibrationNeeded` can now receive `pct == -1` (verifying, absent or stale): do not display it as a score.

### Geofence models

`IndoorGeofence` and `SystemGeofence` are not separate types: there is a single `IONGeofence`, and the `category` field distinguishes the two (`1` crowd management, `2` restricted, `3` system). The trigger enum is `IONGeofence.GeofenceEvent` with values `ON_ENTER` / `ON_EXIT` / `ON_DWELL`.

### IONLocationManager lifecycle

Positioning starts in `IONLocationManager.create()`. `startLocationUpdates()` is kept for compatibility and does not start anything by itself. `stopLocationUpdates()` is final for that instance: create a new manager to start again.

### New APIs

- AR-assisted navigation: `setArNavigationEnabled`, `setArRole`, `attachArPreview`, `updateArPosition`, `IONArPreview`, `IONArRole`, `ARMarker`, `IONSdk.getArMarkersByBuilding`.
- AR route arrows and recovery: `setArRoute`, `requireArMarkerRescan`, `setArRouteHint`, `clearArRouteHint`, `ArRouteGuidance`, `ArRecoveryStatus`, `IONArPreview.onRouteGuidance` / `onRecoveryStatus` / `onScanResult`.
- Licensing: `IONSdk.getLicenseFeatures`, `IONLicenseFeature`.
- Availability: `onPositioningAvailabilityChanged`, `PositioningAvailability`, `PositioningUnavailableReason`.
- Reliability: `onIndoorPositionReliabilityChanged`, `IndoorPositionReliability`, `IndoorPositionUnreliableReason`.
- Calibration: `getLastCalibrationGestureMs`, `revalidateCalibration`, `CalibrationLevel`, `onCalibrationStatus`, `CalibrationStatus`, `CalibrationIssue`.
- Positions: `IONLocation.getQuality`, `getSource`, `isMaskConstrained`, `isPositionDiscontinuous`; `IONLocationManager.isWalkable`.
- Navigation helpers: `RouteProgressTracker`, `RouteDeviationGate`.
- Diagnostics: `NavigationDiagnostics.read`.
- Permissions: `IONSdk.checkPermissions`.
- Content API on `IONSdk`: `getMyLocations`, `getFloor`, `getNavigationPois`, `getBuildingByBeacon`, `getSystemGeofencesByFloor`, `getStandardPoiDetails`, `sendPosition`, `checkLicense`.

### New dependencies

`org.altbeacon:android-beacon-library` must now be declared by the app, and OkHttp should be aligned to `4.12.0`. See [Android Project Configuration](#2-android-project-configuration).

### New permissions

`CAMERA`, `POST_NOTIFICATIONS`, `NEARBY_WIFI_DEVICES`, `ACCESS_WIFI_STATE`, `CHANGE_WIFI_STATE`, `ACCESS_NETWORK_STATE`, `FOREGROUND_SERVICE`, `BLUETOOTH_CONNECT` are now declared by the SDK manifest and merged into your app. Request `CAMERA` and `NEARBY_WIFI_DEVICES` at runtime where relevant.

---

## Best Practices

- Request all runtime permissions before initializing the SDK (`IONSdk.checkPermissions` tells you which are missing)
- Initialize the SDK once per app lifecycle
- Check license validity before using any feature, and create `IONLocationManager` only after a valid license
- Gate your features on `IONSdk.getLicenseFeatures()`; an empty list means nothing is enabled
- Stop updates when your map screen goes away, and create a new manager to start again
- Use `onPositioningAvailabilityChanged` and `onIndoorPositionReliabilityChanged` to avoid showing a position the SDK cannot produce or does not yet trust
- Call `revalidateCalibration()` when the app returns to the foreground
- If you use AR, forward `onResume` / `onPause` to the preview, call `requireArMarkerRescan()` when the app goes to the background, and always call `IONArPreview.release()` — a session left alive blocks the next one
- AR preview callbacks run on the GL thread: switch to the main thread for UI updates

---

## Summary

1. Configure API Key
2. Request runtime permissions
3. Initialize SDK
4. Validate license (wait for `isValid == true`) and read the license features
5. Create IONLocationManager (positioning starts here)
6. Register listener
7. (Optional) Enable AR, choose the role, attach the preview
8. Start updates
9. Stop updates; to start again, create a new IONLocationManager
