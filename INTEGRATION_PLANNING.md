# Planning: Tích hợp BLE Logging vào Vehicle App (Kotlin) và Driver App (Flutter)

## Tổng quan

Sử dụng codebase BLE demo hiện tại để tích hợp vào 2 project:
1. **Vehicle App (Kotlin)**: Feature BLE Advertising để ghi log kết nối
2. **Driver App (Flutter Plugin)**: Feature BLE Scanning + Auto-connect để ghi log kết nối

**Mục đích**: Đo lường tín hiệu BLE giữa 2 thiết bị (vehicle ↔ driver) thông qua vehicle_id duy nhất

---

## Key Requirements

### 1. Vehicle ID
- **vehicle_id** là string thường (ví dụ: `"vehicle_123456"`), không phải UUID format
- **Giải pháp**: Convert vehicle_id thành UUID bằng cách hash hoặc dùng UUID cố định + vehicle_id trong service data
- **UUID conversion**: Dùng `UUID.nameUUIDFromBytes()` để convert vehicle_id string thành UUID format

### 2. Device Names
- **Vehicle App**: Name mặc định là `"Vehicle"` (không cần text input)
- **Driver App**: Name mặc định là `"Driver"` (không cần text input)

### 3. Permissions
- Request permissions khi gọi hàm `startAdvertising()` hoặc `startScanning()`
- Permissions cần thiết (theo AndroidManifest.xml hiện tại):
  - Android 12+: `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`, `BLUETOOTH_ADVERTISE`
  - Android 11 trở xuống: `ACCESS_FINE_LOCATION`

### 4. CSV Logging
- CSV format: `time,type,value,name,vehicle_id`
- **vehicle_id** và **name** được truyền vào khi gọi `startAdvertising()` hoặc `startScanning()`
- Logging bắt đầu khi kết nối thành công
- Export CSV khi gọi hàm `stop()`

### 5. Connection Logic
- **Driver App**: Auto-connect khi scan thấy device với vehicle_id đúng
- **Vehicle App**: Chỉ accept connection từ device có vehicle_id đúng (reject các device khác)
- Cả 2 app chỉ kết nối với 1 device duy nhất trong 1 session

### 6. RSSI Reading
- **Vấn đề**: GATT Server (advertising side) KHÔNG THỂ đọc RSSI trực tiếp
- **Giải pháp**: 
  - Driver App (client): Đọc RSSI từ connection ✅
  - Vehicle App (server): Cũng cần connect như client để đọc RSSI từ driver device
  - Cả 2 app đều cần dual connection: Server + Client để đọc RSSI

### 7. Project Structure
- **Vehicle App**: Tạo folder `feature/ble` trong Kotlin project
- **Driver App**: Tạo Flutter plugin `flutter_ble_logger` để tích hợp

---

## 1. Vehicle App (Kotlin) - BLE Advertising Feature

### 1.1. Project Structure

```
vehicle-app/
  └── feature/
      └── ble/
          ├── BleAdvertisingFeature.kt          // Main feature class
          ├── BleAdvertisingController.kt       // BLE logic
          ├── BleSessionLogger.kt               // Session logging
          ├── BleCsvExporter.kt                // CSV export
          └── models/
              ├── BleConstants.kt              // UUID conversion utils
              └── BleLogEntry.kt               // Log data model
```

### 1.2. API Design

```kotlin
class BleAdvertisingFeature(private val context: Context) {
    
    /**
     * Start BLE advertising và begin session logging
     * 
     * @param vehicleId ID của vehicle (e.g., "vehicle_123456")
     * @param name Tên mặc định: "Vehicle"
     * 
     * Permissions sẽ được request tự động nếu chưa có
     */
    suspend fun startAdvertising(
        vehicleId: String,
        name: String = "Vehicle"
    ): Result<Unit>
    
    /**
     * Stop BLE advertising và export CSV
     * 
     * @return CSV file path or null if no data
     */
    suspend fun stopAdvertisingAndExport(): Result<String?>
    
    /**
     * Get current advertising status
     */
    fun isAdvertising(): Boolean
    
    /**
     * Get connected device address (null if no device connected)
     */
    fun getConnectedDeviceAddress(): String?
}
```

### 1.3. Implementation Details

#### UUID Conversion
```kotlin
object BleConstants {
    // Base UUID để convert vehicle_id thành UUID format
    private const val BASE_UUID_NAMESPACE = "6ba7b810-9dad-11d1-80b4-00c04fd430c8"
    
    /**
     * Convert vehicle_id string thành UUID format
     * Ví dụ: "vehicle_123456" -> UUID
     */
    fun vehicleIdToUuid(vehicleId: String): UUID {
        return UUID.nameUUIDFromBytes(vehicleId.toByteArray(Charsets.UTF_8))
    }
}
```

#### Dual Connection for RSSI
- **GATT Server**: Accept connection từ driver device
- **GATT Client**: Connect ngược lại driver device để đọc RSSI
- Cả 2 connections cần thiết để đọc RSSI từ cả 2 phía

#### Single Device Connection
- Chỉ accept 1 connection duy nhất
- Reject connections từ devices khác khi đã có connection

---

## 2. Driver App (Flutter Plugin) - BLE Scanning + Auto-Connect

### 2.1. Plugin Structure

```
flutter_ble_logger/
  ├── android/
  │   └── src/main/kotlin/com/yourapp/flutter_ble_logger/
  │       ├── FlutterBleLoggerPlugin.kt
  │       ├── BleScannerController.kt
  │       ├── BleSessionLogger.kt
  │       └── BleCsvExporter.kt
  ├── ios/
  │   └── Classes/
  │       └── FlutterBleLoggerPlugin.swift (iOS implementation - optional)
  ├── lib/
  │   └── flutter_ble_logger.dart
  └── pubspec.yaml
```

### 2.2. API Design (Dart)

```dart
class FlutterBleLogger {
  /// Start BLE scanning và auto-connect khi tìm thấy vehicle_id
  /// 
  /// [vehicleId] - ID của vehicle cần connect (e.g., "vehicle_123456")
  /// [name] - Tên mặc định: "Driver"
  /// 
  /// Permissions sẽ được request tự động nếu chưa có
  /// Auto-connect khi scan thấy device với vehicle_id đúng
  static Future<void> startScanning({
    required String vehicleId,
    String name = "Driver",
  }) async;
  
  /// Stop BLE scanning và export CSV
  /// 
  /// Returns: CSV file path or null if no data
  static Future<String?> stopScanningAndExport() async;
  
  /// Check if currently scanning
  static Future<bool> isScanning() async;
  
  /// Get connected device address (null if not connected)
  static Future<String?> getConnectedDevice() async;
  
  /// Stream of BLE events (connected, disconnected, error, etc.)
  static Stream<BleEvent> get eventsStream;
}
```

### 2.3. Implementation Details

#### Auto-Connect Logic
```kotlin
// Trong BleScannerController.kt
private val scanCallback = object : ScanCallback() {
    override fun onScanResult(callbackType: Int, result: ScanResult?) {
        result ?: return
        
        // Check vehicle_id từ service data
        val advertisedVehicleId = extractVehicleId(result)
        if (advertisedVehicleId == targetVehicleId) {
            // Auto-connect
            stopScan()
            connect(result.device)
        }
    }
}
```

#### UUID Conversion
- Convert vehicle_id thành UUID để scan filter
- Sử dụng cùng logic như Vehicle App

---

## 3. CSV Format

### 3.1. Format Specification

```csv
time,type,value,name,vehicle_id
2024-01-15 10:30:00,status,"ADVERTISING_STARTED","Vehicle","vehicle_123456"
2024-01-15 10:30:05,status,"CONNECTED AA:BB:CC:DD:EE:FF","Vehicle","vehicle_123456"
2024-01-15 10:30:07,rssi,"-65","Vehicle","vehicle_123456"
2024-01-15 10:30:09,rssi,"-67","Vehicle","vehicle_123456"
```

### 3.2. Columns
- **time**: Timestamp (yyyy-MM-dd HH:mm:ss)
- **type**: "status" hoặc "rssi"
- **value**: Event value hoặc RSSI value
- **name**: "Vehicle" hoặc "Driver" (mặc định)
- **vehicle_id**: vehicle_id được truyền vào khi start

---

## 4. Permissions Configuration

### 4.1. AndroidManifest.xml

```xml
<!-- Vehicle App & Driver App (Android) -->
<manifest>
    <uses-feature
        android:name="android.hardware.bluetooth_le"
        android:required="true" />
    
    <!-- Legacy permissions -->
    <uses-permission android:name="android.permission.BLUETOOTH" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    
    <!-- Android 12+ permissions -->
    <uses-permission android:name="android.permission.BLUETOOTH_SCAN"
        android:usesPermissionFlags="neverForLocation" />
    <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
    
    <!-- Foreground service (nếu cần) -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE" />
</manifest>
```

### 4.2. Permission Request Flow

```kotlin
// Trong startAdvertising() hoặc startScanning()
suspend fun startAdvertising(vehicleId: String, name: String = "Vehicle") {
    // 1. Check permissions
    if (!hasAllPermissions()) {
        // 2. Request permissions
        requestPermissions() // Suspend until granted/denied
    }
    
    // 3. Start advertising/scanning
    // ...
}
```

---

## 5. Flow Diagram

### 5.1. Vehicle App Flow

```
User calls startAdvertising(vehicleId: "vehicle_123456", name: "Vehicle")
    ↓
Request permissions (nếu chưa có)
    ↓
Convert vehicle_id → UUID
    ↓
Start BLE advertising với UUID
    ↓
GATT Server lắng nghe connections
    ↓
Driver device connect đến
    ↓
Verify vehicle_id match
    ↓
Accept connection (reject nếu vehicle_id không match)
    ↓
Connect như client để đọc RSSI
    ↓
Start RSSI monitoring (mỗi 2 giây)
    ↓
Log events: CONNECTED, RSSI samples
    ↓
User calls stopAdvertisingAndExport()
    ↓
Stop advertising
    ↓
Disconnect
    ↓
Export CSV với vehicle_id và name
```

### 5.2. Driver App Flow

```
User calls startScanning(vehicleId: "vehicle_123456", name: "Driver")
    ↓
Request permissions (nếu chưa có)
    ↓
Convert vehicle_id → UUID
    ↓
Start BLE scanning với UUID filter
    ↓
Tìm thấy Vehicle advertising với vehicle_id đúng
    ↓
Auto-connect (KHÔNG cần user nhấn)
    ↓
Connection established
    ↓
Start RSSI monitoring (mỗi 2 giây)
    ↓
Log events: SCAN_STARTED, DEVICE_FOUND, CONNECTED, RSSI samples
    ↓
User calls stopScanningAndExport()
    ↓
Stop scanning
    ↓
Disconnect
    ↓
Export CSV với vehicle_id và name
```

---

## 6. Key Implementation Points

### 6.1. Vehicle ID to UUID Conversion

```kotlin
// Cả 2 app đều dùng logic này
fun vehicleIdToUuid(vehicleId: String): UUID {
    // Sử dụng nameUUIDFromBytes để convert string thành UUID
    return UUID.nameUUIDFromBytes(vehicleId.toByteArray(Charsets.UTF_8))
}

// Usage
val serviceUuid = vehicleIdToUuid("vehicle_123456")
// Result: UUID format có thể dùng cho advertising/scanning
```

### 6.2. Vehicle ID Verification

```kotlin
// Vehicle App: Verify vehicle_id từ driver device
// Có thể lưu vehicle_id trong service data hoặc characteristic

// Driver App: Verify vehicle_id từ scan result
val advertisedVehicleId = extractVehicleIdFromScanResult(result)
if (advertisedVehicleId == targetVehicleId) {
    // Match! Auto-connect
}
```

### 6.3. Dual Connection for RSSI (Vehicle App)

```kotlin
// Vehicle App cần cả 2 connections:
// 1. GATT Server (accept từ driver)
gattServer = bluetoothManager.openGattServer(context, gattServerCallback)

// 2. GATT Client (connect đến driver để đọc RSSI)
gattClient = driverDevice.connectGatt(context, false, gattClientCallback)
gattClient.readRemoteRssi() // Đọc RSSI từ driver device
```

### 6.4. Auto-Connect (Driver App)

```kotlin
// Trong scanCallback
override fun onScanResult(callbackType: Int, result: ScanResult?) {
    val vehicleId = extractVehicleId(result)
    if (vehicleId == targetVehicleId && connectedAddress == null) {
        // Auto-connect
        stopScan()
        connect(result.device)
    }
}
```

---

## 7. Integration Checklist

### Vehicle App (Kotlin)
- [ ] Tạo folder `feature/ble`
- [ ] Copy và adapt `BleController.kt` → `BleAdvertisingController.kt`
- [ ] Implement UUID conversion từ vehicle_id
- [ ] Implement dual connection (server + client) để đọc RSSI
- [ ] Implement vehicle_id verification
- [ ] Implement permission request trong `startAdvertising()`
- [ ] Implement CSV export với vehicle_id và name
- [ ] Test advertising với vehicle_id
- [ ] Test RSSI reading từ cả 2 phía
- [ ] Test CSV export

### Driver App (Flutter Plugin)
- [ ] Tạo Flutter plugin structure
- [ ] Implement Dart API (`flutter_ble_logger.dart`)
- [ ] Implement Android side (`FlutterBleLoggerPlugin.kt`)
- [ ] Implement UUID conversion từ vehicle_id
- [ ] Implement auto-connect logic
- [ ] Implement vehicle_id verification trong scan
- [ ] Implement permission request trong `startScanning()`
- [ ] Implement CSV export với vehicle_id và name
- [ ] Test scanning với vehicle_id
- [ ] Test auto-connect
- [ ] Test RSSI reading
- [ ] Test CSV export

### Integration Testing
- [ ] Test Vehicle advertising + Driver scanning trên 2 thiết bị
- [ ] Test vehicle_id matching
- [ ] Test auto-connect (verify NO pairing dialog)
- [ ] Test RSSI monitoring trên cả 2 app
- [ ] Test CSV export trên cả 2 app với vehicle_id và name
- [ ] Test single device connection (reject multiple connections)
- [ ] Test permission request flow
- [ ] Test Bluetooth on/off handling

---

## 8. Notes

### 8.1. Vehicle ID Format
- vehicle_id là string thường, không cần UUID format
- Convert thành UUID chỉ để dùng cho BLE advertising/scanning
- vehicle_id gốc vẫn được lưu trong CSV log

### 8.2. RSSI Reading Limitation
- **GATT Server không thể đọc RSSI trực tiếp**
- Vehicle App cần connect như client để đọc RSSI
- Cả 2 app đều cần dual connection để đọc RSSI từ cả 2 phía

### 8.3. Name Defaults
- Vehicle App: `"Vehicle"` (hardcoded, không cần input)
- Driver App: `"Driver"` (hardcoded, không cần input)
- Name được truyền vào khi start và lưu trong CSV

### 8.4. Permission Request
- Permissions được request khi gọi `startAdvertising()` hoặc `startScanning()`
- Nếu user deny permissions, return error
- Không start advertising/scanning nếu thiếu permissions

---

## Kết luận

Codebase hiện tại đã có đủ foundation để implement:
- ✅ Advertising với UUID
- ✅ Scanning với UUID filter
- ✅ GATT Server/Client logic
- ✅ RSSI reading
- ✅ Session logging
- ✅ CSV export

Cần thêm:
- ✅ UUID conversion từ vehicle_id string
- ✅ Auto-connect cho Driver App
- ✅ Dual connection cho Vehicle App để đọc RSSI
- ✅ Vehicle ID verification
- ✅ Permission request flow
- ✅ CSV format với vehicle_id và name
