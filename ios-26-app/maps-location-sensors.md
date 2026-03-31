# Maps, Location, and Sensor Frameworks

## Table of contents
1. CoreLocation Services
2. MapKit (SwiftUI)
3. GeoToolbox (NEW in iOS 26)
4. Core Bluetooth
5. NearbyInteraction
6. HealthKit
7. WeatherKit
8. Anti-Patterns

---

## 1. CoreLocation Services

### CLLocationManager setup and delegate pattern

CLLocationManager is the central object for all location services. Create it on the main thread, assign a delegate, and request authorization before starting updates.

```swift
import CoreLocation

final class LocationService: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
    }

    // MARK: – Authorization

    func requestPermission() {
        manager.requestWhenInUseAuthorization()
    }

    func requestAlwaysPermission() {
        manager.requestAlwaysAuthorization()
    }

    // MARK: – Delegate callbacks

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        switch manager.authorizationStatus {
        case .authorizedWhenInUse:
            manager.startUpdatingLocation()
        case .authorizedAlways:
            manager.startUpdatingLocation()
            manager.allowsBackgroundLocationUpdates = true
        case .denied, .restricted:
            // Direct user to Settings
            break
        case .notDetermined:
            break
        @unknown default:
            break
        }
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last else { return }
        // Use location.coordinate, location.horizontalAccuracy, etc.
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        // Handle errors (e.g., kCLErrorDenied, kCLErrorLocationUnknown)
    }
}
```

### Authorization levels

| Authorization | Info.plist key | Use case |
|---|---|---|
| When In Use | `NSLocationWhenInUseUsageDescription` | Foreground-only features (maps, nearby search) |
| Always | `NSLocationAlwaysAndWhenInUseUsageDescription` | Geofencing, background tracking, significant location changes |

**Important**: Always provide both keys if you request Always authorization. iOS shows the "When In Use" prompt first, then upgrades to "Always" later via a system prompt.

### Continuous updates with AsyncSequence (iOS 17+)

The modern way to receive location updates — no delegate needed.

```swift
import CoreLocation

func startContinuousUpdates() async {
    let updates = CLLocationUpdate.liveUpdates()

    for try await update in updates {
        if let location = update.location {
            print("Lat: \(location.coordinate.latitude), Lon: \(location.coordinate.longitude)")
            print("Accuracy: \(location.horizontalAccuracy)m")
        }

        // Check if updates are accuracy-reduced (approximate location)
        if update.isAccuracyReduced {
            // User granted approximate location only
        }
    }
}
```

### Single location fetch (iOS 17+)

```swift
func fetchSingleLocation() async throws -> CLLocation? {
    let update = try await CLLocationUpdate
        .liveUpdates(.default)
        .first(where: { !$0.isAccuracyReduced })

    return update?.location
}
```

### Activity type configuration

| Activity type | Behavior |
|---|---|
| `.other` | Default, no special optimization |
| `.automotiveNavigation` | Pauses updates when stationary, GPS priority |
| `.fitness` | Pedestrian pace, uses pedometer data |
| `.airborne` | Disables altitude filtering, high-speed GPS |
| `.otherNavigation` | Boat, train — disables auto-pause |

```swift
manager.activityType = .fitness
manager.distanceFilter = 10 // meters between updates
```

### Geocoding / reverse geocoding

**iOS 26 CHANGE**: `CLGeocoder` and `CLPlacemark` are deprecated. Geocoding is now handled through MapKit. See Section 2 for the new API.

Legacy code (pre-iOS 26):
```swift
// DEPRECATED in iOS 26 — use MapKit geocoding instead
let geocoder = CLGeocoder()
let placemarks = try await geocoder.reverseGeocodeLocation(location)
if let placemark = placemarks.first {
    let city = placemark.locality
    let country = placemark.country
}
```

### Geofencing

Monitor entry/exit of up to 20 circular regions simultaneously.

```swift
func setupGeofence(center: CLLocationCoordinate2D, identifier: String) {
    let region = CLCircularRegion(
        center: center,
        radius: 200, // meters
        identifier: identifier
    )
    region.notifyOnEntry = true
    region.notifyOnExit = true

    manager.startMonitoring(for: region)
}

// Delegate callbacks
func locationManager(_ manager: CLLocationManager, didEnterRegion region: CLRegion) {
    // User entered the geofence
}

func locationManager(_ manager: CLLocationManager, didExitRegion region: CLRegion) {
    // User exited the geofence
}

func locationManager(_ manager: CLLocationManager, monitoringDidFailFor region: CLRegion?,
                     withError error: Error) {
    // Handle monitoring failure
}
```

**Limits**: Maximum 20 monitored regions per app. If you need more, dynamically register the nearest 20 based on user location.

### Background location

1. Enable in `project.yml` or Xcode: `UIBackgroundModes` → `location`
2. Set `manager.allowsBackgroundLocationUpdates = true`
3. Set `manager.showsBackgroundLocationIndicator = true` (blue status bar pill)

```swift
// In project.yml (XcodeGen)
settings:
  INFOPLIST_KEY_UIBackgroundModes: location
```

| Accuracy | Battery impact | Update frequency |
|---|---|---|
| `kCLLocationAccuracyBestForNavigation` | Very high | Continuous, GPS + sensors |
| `kCLLocationAccuracyBest` | High | ~1-5 seconds |
| `kCLLocationAccuracyNearestTenMeters` | Moderate | ~5-15 seconds |
| `kCLLocationAccuracyHundredMeters` | Low | ~15-60 seconds |
| `kCLLocationAccuracyKilometer` | Very low | Minutes |
| `kCLLocationAccuracyThreeKilometers` | Minimal | Minutes |

### Significant location changes

Ultra-low-power alternative — wakes app for cell tower transitions (~500m moves).

```swift
manager.startMonitoringSignificantLocationChanges()
// Delivers updates via the same didUpdateLocations delegate method
// Works even after app termination — iOS relaunches the app
```

---

## 2. MapKit (SwiftUI)

### Map view with MapContentBuilder (iOS 17+)

```swift
import MapKit

struct MapExplorer: View {
    @State private var position: MapCameraPosition = .automatic

    let homeCoord = CLLocationCoordinate2D(latitude: 37.7749, longitude: -122.4194)
    let route: [CLLocationCoordinate2D] = [
        CLLocationCoordinate2D(latitude: 37.7749, longitude: -122.4194),
        CLLocationCoordinate2D(latitude: 37.7849, longitude: -122.4094)
    ]

    var body: some View {
        Map(position: $position) {
            // System marker with SF Symbol
            Marker("Home", systemImage: "house.fill", coordinate: homeCoord)
                .tint(.blue)

            // Custom annotation
            Annotation("Coffee", coordinate: CLLocationCoordinate2D(latitude: 37.78, longitude: -122.41)) {
                Image(systemName: "cup.and.saucer.fill")
                    .padding(6)
                    .background(.brown)
                    .foregroundStyle(.white)
                    .clipShape(Circle())
            }

            // Circle overlay
            MapCircle(center: homeCoord, radius: 500)
                .foregroundStyle(.blue.opacity(0.2))
                .stroke(.blue, lineWidth: 2)

            // Polyline overlay
            MapPolyline(coordinates: route)
                .stroke(.blue, lineWidth: 3)

            // User location
            UserAnnotation()
        }
        .mapStyle(.standard(pointsOfInterest: .including([.cafe, .restaurant])))
        .mapControls {
            MapUserLocationButton()
            MapCompass()
            MapScaleView()
        }
    }
}
```

### MapCameraPosition options

| Position | Description |
|---|---|
| `.automatic` | Fits all map content |
| `.region(MKCoordinateRegion)` | Specific region |
| `.camera(MapCamera)` | Full camera control (center, distance, heading, pitch) |
| `.userLocation(fallback: .automatic)` | Centers on user, with fallback |
| `.rect(MKMapRect)` | Specific map rect |

```swift
// Camera with heading and pitch
@State private var position: MapCameraPosition = .camera(
    MapCamera(centerCoordinate: coord, distance: 1000, heading: 90, pitch: 45)
)
```

### MapStyle options

```swift
.mapStyle(.standard)                                    // Default
.mapStyle(.standard(elevation: .realistic))             // 3D terrain
.mapStyle(.standard(pointsOfInterest: .excluding([.airport])))
.mapStyle(.standard(showsTraffic: true))
.mapStyle(.imagery)                                     // Satellite
.mapStyle(.imagery(elevation: .realistic))              // 3D satellite
.mapStyle(.hybrid)                                      // Satellite + labels
.mapStyle(.hybrid(elevation: .realistic, showsTraffic: true))
```

### Look Around

```swift
struct LookAroundExample: View {
    @State private var lookAroundScene: MKLookAroundScene?

    var body: some View {
        VStack {
            if let scene = lookAroundScene {
                LookAroundPreview(initialScene: scene)
                    .frame(height: 200)
            }
        }
        .task {
            let request = MKLookAroundSceneRequest(coordinate: coordinate)
            lookAroundScene = try? await request.scene
        }
    }
}
```

### Directions

```swift
func getDirections(from source: CLLocationCoordinate2D,
                   to destination: CLLocationCoordinate2D) async throws -> MKDirections.Response {
    let request = MKDirections.Request()
    request.source = MKMapItem(placemark: MKPlacemark(coordinate: source))
    request.destination = MKMapItem(placemark: MKPlacemark(coordinate: destination))
    request.transportType = .automobile // .walking, .transit, .cycling (iOS 26)

    let directions = MKDirections(request: request)
    return try await directions.calculate()
}

// Display route on map
struct RouteView: View {
    @State private var route: MKRoute?
    @State private var position: MapCameraPosition = .automatic

    var body: some View {
        Map(position: $position) {
            if let route {
                MapPolyline(route.polyline)
                    .stroke(.blue, lineWidth: 5)
            }
        }
        .task {
            let response = try? await getDirections(from: start, to: end)
            route = response?.routes.first
        }
    }
}
```

### MKLocalSearch

```swift
func searchNearby(query: String, region: MKCoordinateRegion) async throws -> [MKMapItem] {
    let request = MKLocalSearch.Request()
    request.naturalLanguageQuery = query
    request.region = region
    request.resultTypes = .pointOfInterest // .address, .physicalFeature

    let search = MKLocalSearch(request: request)
    let response = try await search.start()
    return response.mapItems
}
```

### Map interaction and selection

```swift
Map(position: $position, selection: $selectedMarker) {
    ForEach(places) { place in
        Marker(place.name, coordinate: place.coordinate)
            .tag(place.id)
    }
}
.onMapCameraChange(frequency: .continuous) { context in
    // context.camera, context.region, context.rect
    visibleRegion = context.region
}
.onMapCameraChange(frequency: .onEnd) { context in
    // Fetch data for new region
    await searchInRegion(context.region)
}
```

### iOS 26 MapKit additions

- **Cycling directions**: `request.transportType = .cycling` now supported
- **watchOS MapKit expansion**: search for POIs, display routes, overlays — full MapKit feature parity
- **Geocoding moved here**: CLGeocoder deprecated; use MapKit-based geocoding APIs

---

## 3. GeoToolbox (NEW in iOS 26)

GeoToolbox is a new framework introduced in iOS 26 for rich place data.

### PlaceDescriptor

`PlaceDescriptor` provides detailed information about a geographic location — richer than what CLPlacemark offered.

```swift
import GeoToolbox

func describePlaceAt(coordinate: CLLocationCoordinate2D) async throws {
    let descriptor = try await PlaceDescriptor(coordinate: coordinate)

    // Access rich place metadata
    print(descriptor.name)          // e.g., "Golden Gate Park"
    print(descriptor.category)      // e.g., .park
    print(descriptor.locality)      // e.g., "San Francisco"
    print(descriptor.country)       // e.g., "United States"
}
```

### Key capabilities

| Feature | Description |
|---|---|
| Place resolution | Determine what is at a given coordinate |
| Category data | Structured place categories (park, restaurant, etc.) |
| Provider agnostic | Works with MapKit or other mapping providers |
| Replaces CLPlacemark | Richer data model than the deprecated CLPlacemark |

### Geocoding replacement

GeoToolbox together with MapKit replaces the deprecated CLGeocoder:

```swift
// OLD (deprecated in iOS 26):
// let placemarks = try await CLGeocoder().reverseGeocodeLocation(location)

// NEW (iOS 26+):
import GeoToolbox
let descriptor = try await PlaceDescriptor(
    coordinate: location.coordinate
)
```

---

## 4. Core Bluetooth

### CBCentralManager setup

```swift
import CoreBluetooth

final class BluetoothService: NSObject, CBCentralManagerDelegate, CBPeripheralDelegate {
    private var centralManager: CBCentralManager!
    private var discoveredPeripheral: CBPeripheral?

    // Custom service and characteristic UUIDs
    let serviceUUID = CBUUID(string: "12345678-1234-1234-1234-123456789ABC")
    let characteristicUUID = CBUUID(string: "ABCDEFAB-1234-1234-1234-123456789ABC")

    override init() {
        super.init()
        centralManager = CBCentralManager(delegate: self, queue: nil)
    }

    // MARK: – Central manager delegate

    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        switch central.state {
        case .poweredOn:
            // Safe to scan
            centralManager.scanForPeripherals(
                withServices: [serviceUUID],
                options: [CBCentralManagerScanOptionAllowDuplicatesKey: false]
            )
        case .poweredOff:
            // Bluetooth is off — prompt user
            break
        case .unauthorized:
            // Permission denied
            break
        case .unsupported:
            // Device doesn't support BLE
            break
        case .resetting, .unknown:
            break
        @unknown default:
            break
        }
    }

    func centralManager(_ central: CBCentralManager,
                        didDiscover peripheral: CBPeripheral,
                        advertisementData: [String: Any],
                        rssi RSSI: NSNumber) {
        discoveredPeripheral = peripheral
        centralManager.stopScan()
        centralManager.connect(peripheral, options: nil)
    }

    func centralManager(_ central: CBCentralManager,
                        didConnect peripheral: CBPeripheral) {
        peripheral.delegate = self
        peripheral.discoverServices([serviceUUID])
    }

    func centralManager(_ central: CBCentralManager,
                        didFailToConnect peripheral: CBPeripheral,
                        error: Error?) {
        // Handle connection failure
    }

    // MARK: – Peripheral delegate

    func peripheral(_ peripheral: CBPeripheral,
                    didDiscoverServices error: Error?) {
        guard let services = peripheral.services else { return }
        for service in services {
            peripheral.discoverCharacteristics([characteristicUUID], for: service)
        }
    }

    func peripheral(_ peripheral: CBPeripheral,
                    didDiscoverCharacteristicsFor service: CBService,
                    error: Error?) {
        guard let characteristics = service.characteristics else { return }
        for characteristic in characteristics {
            if characteristic.uuid == characteristicUUID {
                // Read value
                peripheral.readValue(for: characteristic)

                // Or subscribe to notifications
                peripheral.setNotifyValue(true, for: characteristic)
            }
        }
    }

    func peripheral(_ peripheral: CBPeripheral,
                    didUpdateValueFor characteristic: CBCharacteristic,
                    error: Error?) {
        guard let data = characteristic.value else { return }
        // Process data from the peripheral
        let value = String(data: data, encoding: .utf8)
    }
}
```

### CBPeripheralManager (advertising)

```swift
final class PeripheralService: NSObject, CBPeripheralManagerDelegate {
    private var peripheralManager: CBPeripheralManager!

    override init() {
        super.init()
        peripheralManager = CBPeripheralManager(delegate: self, queue: nil)
    }

    func peripheralManagerDidUpdateState(_ peripheral: CBPeripheralManager) {
        guard peripheral.state == .poweredOn else { return }

        let characteristic = CBMutableCharacteristic(
            type: characteristicUUID,
            properties: [.read, .notify],
            value: nil,
            permissions: [.readable]
        )

        let service = CBMutableService(type: serviceUUID, primary: true)
        service.characteristics = [characteristic]

        peripheralManager.add(service)
    }

    func peripheralManager(_ peripheral: CBPeripheralManager,
                           didAdd service: CBService, error: Error?) {
        peripheralManager.startAdvertising([
            CBAdvertisementDataServiceUUIDsKey: [serviceUUID],
            CBAdvertisementDataLocalNameKey: "MyDevice"
        ])
    }
}
```

### Background Bluetooth

Enable `UIBackgroundModes` → `bluetooth-central` and/or `bluetooth-peripheral`.

State restoration keeps connections alive after app termination:

```swift
// Init with restoration identifier
centralManager = CBCentralManager(
    delegate: self,
    queue: nil,
    options: [CBCentralManagerOptionRestoreIdentifierKey: "myCentralManager"]
)

// Implement restoration callback
func centralManager(_ central: CBCentralManager,
                    willRestoreState dict: [String: Any]) {
    if let peripherals = dict[CBCentralManagerRestoredStatePeripheralsKey] as? [CBPeripheral] {
        for peripheral in peripherals {
            peripheral.delegate = self
            discoveredPeripheral = peripheral
        }
    }
}
```

### Required permissions

| Key | Required for |
|---|---|
| `NSBluetoothAlwaysUsageDescription` | All BLE operations (iOS 13+) |
| `NSBluetoothPeripheralUsageDescription` | Peripheral mode (deprecated iOS 13, still needed for older targets) |

---

## 5. NearbyInteraction

### NISession setup

NearbyInteraction measures distance and direction to nearby devices using the U1/U2 ultra-wideband chip.

```swift
import NearbyInteraction

final class NearbyService: NSObject, NISessionDelegate {
    private var niSession: NISession?

    func start() {
        guard NISession.isSupported else {
            // Device lacks UWB chip — provide fallback
            return
        }

        niSession = NISession()
        niSession?.delegate = self
        // Share niSession.discoveryToken with the peer via
        // MultipeerConnectivity, WebSocket, or other transport
    }

    func didReceivePeerToken(_ peerToken: NIDiscoveryToken) {
        let config = NINearbyPeerConfiguration(peerToken: peerToken)
        niSession?.run(config)
    }

    // MARK: – NISessionDelegate

    func session(_ session: NISession,
                 didUpdate nearbyObjects: [NINearbyObject]) {
        guard let object = nearbyObjects.first else { return }

        // Distance in meters
        if let distance = object.distance {
            print("Distance: \(distance)m")
        }

        // Direction as a 3D unit vector (simd_float3)
        if let direction = object.direction {
            print("Direction: \(direction)")
        }
    }

    func session(_ session: NISession,
                 didRemove nearbyObjects: [NINearbyObject],
                 reason: NINearbyObject.RemovalReason) {
        switch reason {
        case .peerEnded:
            // Peer stopped the session
            break
        case .timeout:
            // Lost contact — restart if needed
            niSession?.run(NINearbyPeerConfiguration(peerToken: lastPeerToken))
        @unknown default:
            break
        }
    }

    func sessionWasSuspended(_ session: NISession) {
        // App went to background
    }

    func sessionSuspensionEnded(_ session: NISession) {
        // App returned to foreground — session auto-resumes
    }

    func session(_ session: NISession,
                 didInvalidateWith error: Error) {
        // Session is permanently invalid — create a new one
        niSession = NISession()
        niSession?.delegate = self
    }
}
```

### Chip requirements

| Chip | Devices | Capabilities |
|---|---|---|
| U1 | iPhone 11–14, Apple Watch Series 6-8 | Distance + direction |
| U2 | iPhone 15+, Apple Watch Series 9+ | Improved precision, lower power |
| None | Older devices, iPads without UWB | Not supported — provide fallback |

### Peer discovery token exchange

The discovery token must be exchanged out-of-band (NearbyInteraction does not handle discovery). Common transports:
- **MultipeerConnectivity** — local Wi-Fi/Bluetooth
- **WebSocket / server** — over the internet
- **Core Bluetooth** — custom BLE characteristic

```swift
// Encode token for transmission
let tokenData = try NSKeyedArchiver.archivedData(
    withRootObject: niSession.discoveryToken!,
    requiringSecureCoding: true
)

// Decode received token
let peerToken = try NSKeyedUnarchiver.unarchivedObject(
    ofClass: NIDiscoveryToken.self,
    from: receivedData
)!
```

---

## 6. HealthKit

### HKHealthStore setup

```swift
import HealthKit

final class HealthService {
    let store = HKHealthStore()

    func requestAuthorization() async throws {
        guard HKHealthStore.isHealthDataAvailable() else {
            // iPad, Mac Catalyst without Health — not supported
            return
        }

        let readTypes: Set<HKObjectType> = [
            HKQuantityType(.stepCount),
            HKQuantityType(.heartRate),
            HKQuantityType(.activeEnergyBurned),
            HKCategoryType(.sleepAnalysis)
        ]

        let writeTypes: Set<HKSampleType> = [
            HKQuantityType(.stepCount),
            HKQuantityType(.activeEnergyBurned)
        ]

        try await store.requestAuthorization(toShare: writeTypes, read: readTypes)
    }
}
```

### Reading data: queries

**Single fetch with HKSampleQuery:**

```swift
func fetchRecentHeartRates() async throws -> [HKQuantitySample] {
    let heartRateType = HKQuantityType(.heartRate)
    let sortDescriptor = SortDescriptor(\HKSample.startDate, order: .reverse)
    let predicate = HKQuery.predicateForSamples(
        withStart: Calendar.current.date(byAdding: .hour, value: -24, to: .now),
        end: .now,
        options: .strictStartDate
    )

    let descriptor = HKSampleQueryDescriptor(
        predicates: [.quantitySample(type: heartRateType, predicate: predicate)],
        sortDescriptors: [sortDescriptor],
        limit: 100
    )

    return try await descriptor.result(for: store)
}
```

**Aggregates with HKStatisticsQuery:**

```swift
func fetchTodaySteps() async throws -> Double {
    let stepsType = HKQuantityType(.stepCount)
    let startOfDay = Calendar.current.startOfDay(for: .now)
    let predicate = HKQuery.predicateForSamples(
        withStart: startOfDay, end: .now, options: .strictStartDate
    )

    let descriptor = HKStatisticsQueryDescriptor(
        predicate: .quantitySample(type: stepsType, predicate: predicate),
        options: .cumulativeSum
    )

    let result = try await descriptor.result(for: store)
    return result?.sumQuantity()?.doubleValue(for: .count()) ?? 0
}
```

**Time series with HKStatisticsCollectionQuery:**

```swift
func fetchWeeklySteps() async throws -> [(date: Date, steps: Double)] {
    let stepsType = HKQuantityType(.stepCount)
    let calendar = Calendar.current
    let endDate = Date.now
    let startDate = calendar.date(byAdding: .day, value: -7, to: endDate)!

    let predicate = HKQuery.predicateForSamples(
        withStart: startDate, end: endDate, options: .strictStartDate
    )

    let descriptor = HKStatisticsCollectionQueryDescriptor(
        predicate: .quantitySample(type: stepsType, predicate: predicate),
        options: .cumulativeSum,
        anchorDate: calendar.startOfDay(for: endDate),
        intervalComponents: DateComponents(day: 1)
    )

    let collection = try await descriptor.result(for: store)
    var results: [(date: Date, steps: Double)] = []

    collection.enumerateStatistics(from: startDate, to: endDate) { statistics, _ in
        let steps = statistics.sumQuantity()?.doubleValue(for: .count()) ?? 0
        results.append((statistics.startDate, steps))
    }

    return results
}
```

### Writing data

```swift
func saveSteps(count: Double, start: Date, end: Date) async throws {
    let stepsType = HKQuantityType(.stepCount)
    let quantity = HKQuantity(unit: .count(), doubleValue: count)
    let sample = HKQuantitySample(
        type: stepsType,
        quantity: quantity,
        start: start,
        end: end
    )

    try await store.save(sample)
}
```

### Background delivery

Register in app delegate for updates even when app is terminated.

```swift
// In App init or AppDelegate
func enableBackgroundDelivery() {
    let stepsType = HKQuantityType(.stepCount)

    store.enableBackgroundDelivery(for: stepsType, frequency: .hourly) { success, error in
        if let error {
            print("Background delivery failed: \(error)")
        }
    }
}

// HKObserverQuery fires when new data arrives
func observeSteps() {
    let stepsType = HKQuantityType(.stepCount)

    let query = HKObserverQuery(sampleType: stepsType, predicate: nil) { _, completionHandler, error in
        // New step data is available — fetch it
        Task {
            let steps = try? await self.fetchTodaySteps()
            // Update UI / widget
        }
        completionHandler()
    }

    store.execute(query)
}
```

### iOS 26: Medications API

```swift
// Read medications the user has added in the Health app
let medicationType = HKClinicalType(.medicationRecord)

// Request authorization to read medications
try await store.requestAuthorization(toShare: [], read: [medicationType])

// Query medication records
let descriptor = HKSampleQueryDescriptor(
    predicates: [.clinicalRecord(type: medicationType)],
    sortDescriptors: [SortDescriptor(\.startDate, order: .reverse)],
    limit: 50
)
let records = try await descriptor.result(for: store)
```

### Workouts

```swift
// Start a workout session
let workoutConfig = HKWorkoutConfiguration()
workoutConfig.activityType = .running
workoutConfig.locationType = .outdoor

// Build a workout with route data
let builder = HKWorkoutBuilder(healthStore: store, configuration: workoutConfig, device: .local())
try await builder.beginCollection(at: .now)

// Add route data
let routeBuilder = HKWorkoutRouteBuilder(healthStore: store, device: .local())
routeBuilder.insertRouteData(locations) { success, error in }

try await builder.endCollection(at: .now)
let workout = try await builder.finishWorkout()
try await routeBuilder.finishRoute(with: workout, metadata: nil)
```

### Required privacy keys

| Key | Purpose |
|---|---|
| `NSHealthShareUsageDescription` | Reading health data |
| `NSHealthUpdateUsageDescription` | Writing health data |

---

## 7. WeatherKit

### Setup requirements

1. Enable **WeatherKit** capability in App Store Connect
2. Add the WeatherKit entitlement to your app
3. Rate limit: **500,000 calls/month** free per Apple Developer Program membership

### Fetching current weather

```swift
import WeatherKit
import CoreLocation

func fetchWeather(for location: CLLocation) async throws {
    let weatherService = WeatherService.shared

    let weather = try await weatherService.weather(for: location)

    // Current conditions
    let current = weather.currentWeather
    print("Temperature: \(current.temperature)")              // e.g., 22.0°C
    print("Feels like: \(current.apparentTemperature)")
    print("Condition: \(current.condition)")                   // .clear, .rain, etc.
    print("Humidity: \(current.humidity)")                     // 0.0–1.0
    print("Wind: \(current.wind.speed), \(current.wind.direction)")
    print("UV Index: \(current.uvIndex.value)")
}
```

### Forecasts

```swift
func fetchForecasts(for location: CLLocation) async throws {
    let weather = try await WeatherService.shared.weather(for: location)

    // Hourly forecast (up to 240 hours)
    for hour in weather.hourlyForecast.prefix(24) {
        print("\(hour.date): \(hour.temperature), \(hour.condition)")
    }

    // Daily forecast (up to 10 days)
    for day in weather.dailyForecast {
        print("\(day.date): \(day.lowTemperature)–\(day.highTemperature)")
        print("  Precipitation chance: \(day.precipitationChance)")
    }

    // Minute forecast (next hour, where available)
    if let minuteForecast = weather.minuteForecast {
        for minute in minuteForecast {
            print("\(minute.date): \(minute.precipitationIntensity)")
        }
    }
}
```

### Severe weather alerts

```swift
func checkAlerts(for location: CLLocation) async throws {
    let weather = try await WeatherService.shared.weather(
        for: location,
        including: .alerts
    )

    if let alerts = weather {
        for alert in alerts {
            print("Alert: \(alert.summary)")
            print("Severity: \(alert.severity)")  // .minor, .moderate, .severe, .extreme
            print("Region: \(alert.region)")
        }
    }
}
```

### Selective data fetching

Fetch only what you need to minimize API calls:

```swift
// Fetch only current weather + daily forecast
let (current, daily) = try await WeatherService.shared.weather(
    for: location,
    including: .current, .daily
)

// Fetch only hourly
let hourly = try await WeatherService.shared.weather(
    for: location,
    including: .hourly
)
```

### Attribution requirement

**Mandatory for App Store approval**: Display the Apple Weather attribution.

```swift
struct WeatherAttributionView: View {
    @State private var attribution: WeatherAttribution?

    var body: some View {
        VStack {
            // Your weather UI...

            // REQUIRED: Apple Weather logo
            if let attribution {
                AsyncImage(url: attribution.combinedMarkLightURL) { image in
                    image.resizable().scaledToFit()
                } placeholder: {
                    EmptyView()
                }
                .frame(height: 12)

                Link("Weather Data", destination: attribution.legalPageURL)
                    .font(.caption2)
            }
        }
        .task {
            attribution = try? await WeatherService.shared.attribution
        }
    }
}
```

---

## 8. Anti-Patterns

| Anti-pattern | Why it's wrong | Correct approach |
|---|---|---|
| Request "Always" location when "When In Use" suffices | Users deny broad permissions; Apple rejects apps with unjustified Always requests | Start with When In Use; upgrade to Always only if geofencing or background tracking is essential |
| Poll CLLocationManager with a Timer | Wastes battery, fights the system's power management | Use `CLLocationUpdate.liveUpdates()` AsyncSequence or delegate callbacks |
| Use CLGeocoder in new iOS 26 code | Deprecated — will be removed in future iOS versions | Use MapKit geocoding or GeoToolbox `PlaceDescriptor` |
| Skip WeatherKit attribution | App Store rejection — Apple requires the Weather logo | Always display `WeatherService.shared.attribution` with the combined mark |
| Forget HealthKit background delivery registration | Observer queries won't fire when app is terminated | Call `enableBackgroundDelivery(for:frequency:)` at app launch |
| Assume Bluetooth is available | iPads, simulators, or powered-off radios will crash | Always check `CBCentralManager.state == .poweredOn` before scanning |
| Ignore NearbyInteraction chip requirements | App crashes or silent failure on devices without UWB | Check `NISession.isSupported` and provide graceful fallback UI |
| Request all HealthKit types at once | Overwhelming permission sheet scares users; Apple may reject | Request types incrementally as the user accesses each feature |
| Hardcode coordinate assumptions | Different regions have different precision needs, datum shifts | Always use `CLLocationCoordinate2D` and respect user's locale for display |
| Forget to stop Bluetooth scanning | Battery drain — scanning is power-intensive | Call `centralManager.stopScan()` once you find the target peripheral |
