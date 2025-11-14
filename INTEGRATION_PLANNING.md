# Planning: Tích hợp BLE Logging vào Vehicle App (Kotlin) và Driver App (Flutter)

## Tổng quan

Sử dụng codebase BLE demo hiện tại để tích hợp vào 2 project:
1. **Vehicle App (Kotlin)**: Feature BLE Advertising đơn giản (chỉ start/stop với vehicle_id, không ghi log)
2. **Driver App (Flutter Plugin)**: Feature BLE Scanning + Auto-connect + Logging + CSV Export

**Mục đích**: Đo lường tín hiệu BLE giữa 2 thiết bị (vehicle ↔ driver) thông qua vehicle_id duy nhất

---

## Key Requirements

### 1. Vehicle ID & UUID Conversion
- **vehicle_id** là string thường (ví dụ: `"vehicle_123456"`), không phải UUID format
- **UUID conversion**: Dùng `UUID.nameUUIDFromBytes()` để convert vehicle_id string thành UUID format
- **Advertising/Scanning**: Chỉ sử dụng UUID (không có service data, không có device name) để tránh lỗi "data too large"
- **Hàm conversion** (cần implement trong cả 2 app):
  ```kotlin
  fun vehicleIdToUuid(vehicleId: String): UUID {
      return UUID.nameUUIDFromBytes(vehicleId.toByteArray(StandardCharsets.UTF_8))
  }
  ```

### 2. Device Names
- **Vehicle App**: Name mặc định là `"Vehicle"` (không cần text input)
- **Driver App**: Name mặc định là `"Driver"` (không cần text input)

### 3. Permissions
- Request permissions khi gọi hàm `startAdvertising()` hoặc `startScanning()`
- Permissions cần thiết (theo AndroidManifest.xml hiện tại):
  - Android 12+: `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`, `BLUETOOTH_ADVERTISE`
  - Android 11 trở xuống: `ACCESS_FINE_LOCATION`

### 4. CSV Logging
- **Chỉ Driver App ghi log và export CSV**
- **Vehicle App**: Không cần ghi log, chỉ đơn giản start/stop advertising
- CSV format: `time,type,value,name,vehicle_id`
- **vehicle_id** và **name** được truyền vào khi gọi `startScanning()`
- Logging bắt đầu khi scan started và kết nối thành công
- Export CSV khi gọi hàm `stopScanningAndExport()`

### 5. Connection Logic
- **Driver App**: Auto-connect khi scan thấy device với UUID match (UUID được convert từ vehicle_id)
- **Vehicle App**: Chỉ accept connection từ device (UUID match qua scan filter)
- Cả 2 app chỉ kết nối với 1 device duy nhất trong 1 session
- **Auto-connect**: Khi scan thấy device với UUID đúng → tự động connect (không cần user action)

### 6. RSSI Reading
- **Driver App**: Đọc RSSI từ connection (GATT Client) ✅
- **Vehicle App**: Không cần đọc RSSI (chỉ advertise, không ghi log)

### 7. Project Structure
- **Vehicle App**: Tạo folder `feature/ble` trong Kotlin project
- **Driver App**: Tạo Flutter plugin `flutter_ble_logger` để tích hợp

---

## 1. Vehicle App (Kotlin) - BLE Advertising Feature (Simple)

### 1.1. Project Structure

```
vehicle-app/
  └── feature/
      └── ble/
          ├── BleAdvertisingFeature.kt          // Main feature class
          ├── BleAdvertisingController.kt       // BLE logic (simplified)
          └── models/
              └── BleConstants.kt              // UUID conversion utils
```

### 1.2. API Design

```kotlin
class BleAdvertisingFeature(private val context: Context) {
    
    /**
     * Start BLE advertising với vehicle_id
     * 
     * @param vehicleId ID của vehicle (e.g., "vehicle_123456")
     * 
     * Permissions sẽ được request tự động nếu chưa có
     * Chỉ advertise với UUID (không có service data, không có device name)
     */
    suspend fun startAdvertising(vehicleId: String): Result<Unit>
    
    /**
     * Stop BLE advertising
     */
    suspend fun stopAdvertising(): Result<Unit>
    
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

#### UUID Conversion (REQUIRED)
```kotlin
object BleConstants {
    /**
     * Convert vehicle_id string thành UUID format
     * Ví dụ: "vehicle_123456" -> UUID
     * 
     * IMPORTANT: Cả 2 app (Vehicle & Driver) phải dùng cùng hàm này
     * để đảm bảo UUID match chính xác
     */
    fun vehicleIdToUuid(vehicleId: String): UUID {
        return UUID.nameUUIDFromBytes(
            vehicleId.toByteArray(StandardCharsets.UTF_8)
        )
    }
}
```

#### Advertising Data (UUID Only)
- **Chỉ advertise với UUID**: Không có service data, không có device name
- **Lý do**: Tránh lỗi "data too large" (BLE limit: 31 bytes)
- **UUID size**: 16 bytes (vừa đủ, không lo lỗi)

#### GATT Server
- **GATT Server**: Accept connection từ driver device
- **Single Device Connection**: Chỉ accept 1 connection duy nhất
- **Không cần**: RSSI reading, logging, CSV export

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

#### UUID Conversion (REQUIRED)
```kotlin
// IMPORTANT: Phải dùng cùng hàm như Vehicle App
fun vehicleIdToUuid(vehicleId: String): UUID {
    return UUID.nameUUIDFromBytes(
        vehicleId.toByteArray(StandardCharsets.UTF_8)
    )
}
```

#### Auto-Connect Logic
```kotlin
// Trong BleScannerController.kt
private val scanCallback = object : ScanCallback() {
    override fun onScanResult(callbackType: Int, result: ScanResult?) {
        result ?: return
        
        // Scan filter đã đảm bảo UUID match
        // Nếu thấy device này, nghĩa là UUID đã match → Auto-connect
        if (targetVehicleIdForScan != null && connectedAddress == null) {
            // Auto-connect to this device (UUID already matched via scan filter)
            stopScan()
            delay(100) // Small delay to ensure scan stopped
            connect(result.device.address)
        }
    }
}
```

#### Scanning với UUID Filter
- Convert vehicle_id thành UUID để tạo scan filter
- Chỉ scan devices có UUID match
- Auto-connect khi tìm thấy device với UUID đúng

---

## 3. CSV Format

### 3.1. Format Specification

```csv
time,type,value,name,vehicle_id
2024-01-15 10:30:00,status,"SCAN_STARTED","Driver","vehicle_123456"
2024-01-15 10:30:05,status,"DEVICE_FOUND AA:BB:CC:DD:EE:FF","Driver","vehicle_123456"
2024-01-15 10:30:06,status,"CONNECTED AA:BB:CC:DD:EE:FF","Driver","vehicle_123456"
2024-01-15 10:30:08,rssi,"-65","Driver","vehicle_123456"
2024-01-15 10:30:10,rssi,"-67","Driver","vehicle_123456"
2024-01-15 10:30:12,status,"DISCONNECTED","Driver","vehicle_123456"
```

**Note**: Chỉ Driver App ghi log và export CSV. Vehicle App không ghi log.

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

### 5.1. Vehicle App Flow (Simple)

```
User calls startAdvertising(vehicleId: "vehicle_123456")
    ↓
Request permissions (nếu chưa có)
    ↓
Convert vehicle_id → UUID (dùng vehicleIdToUuid())
    ↓
Start BLE advertising với UUID only (không có service data, device name)
    ↓
GATT Server lắng nghe connections
    ↓
Driver device connect đến (UUID đã match qua scan filter)
    ↓
Accept connection
    ↓
[Advertising continues...]
    ↓
User calls stopAdvertising()
    ↓
Stop advertising
    ↓
Disconnect (nếu có connection)
```

**Note**: Vehicle App không ghi log, không export CSV. Chỉ đơn giản start/stop advertising.

### 5.2. Driver App Flow

```
User calls startScanning(vehicleId: "vehicle_123456", name: "Driver")
    ↓
Request permissions (nếu chưa có)
    ↓
Convert vehicle_id → UUID (dùng vehicleIdToUuid())
    ↓
Start BLE scanning với UUID filter
    ↓
Tìm thấy Vehicle advertising với UUID match
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
    ↓
Return CSV file path
```

---

## 6. Key Implementation Points

### 6.1. Vehicle ID to UUID Conversion (CRITICAL)

```kotlin
// IMPORTANT: Cả 2 app (Vehicle & Driver) PHẢI dùng cùng hàm này
// Để đảm bảo UUID match chính xác khi scan/advertise

import java.nio.charset.StandardCharsets
import java.util.UUID

fun vehicleIdToUuid(vehicleId: String): UUID {
    return UUID.nameUUIDFromBytes(
        vehicleId.toByteArray(StandardCharsets.UTF_8)
    )
}

// Usage trong Vehicle App
val serviceUuid = vehicleIdToUuid("vehicle_123456")
// Advertise với UUID này

// Usage trong Driver App
val serviceUuid = vehicleIdToUuid("vehicle_123456")
// Scan với UUID filter này
```

**⚠️ Lưu ý**: 
- Phải dùng `StandardCharsets.UTF_8` (không dùng `Charsets.UTF_8`)
- Phải dùng cùng encoding để đảm bảo UUID match

### 6.2. Advertising Data (UUID Only)

```kotlin
// Vehicle App: Chỉ advertise với UUID
val serviceUuid = vehicleIdToUuid(vehicleId)
val advertiseData = AdvertiseData.Builder()
    .addServiceUuid(ParcelUuid(serviceUuid))
    .build()
// Không có service data, không có device name
```

### 6.3. Scanning với UUID Filter

```kotlin
// Driver App: Scan với UUID filter
val serviceUuid = vehicleIdToUuid(vehicleId)
val filter = ScanFilter.Builder()
    .setServiceUuid(ParcelUuid(serviceUuid))
    .build()
scanner.startScan(listOf(filter), settings, scanCallback)
```

### 6.4. Auto-Connect (Driver App)

```kotlin
// Trong scanCallback
override fun onScanResult(callbackType: Int, result: ScanResult?) {
    result ?: return
    
    // Scan filter đã đảm bảo UUID match
    // Nếu thấy device này, nghĩa là UUID đã match → Auto-connect
    if (targetVehicleIdForScan != null && connectedAddress == null) {
        // Auto-connect to this device (UUID already matched via scan filter)
        stopScan()
        delay(100) // Small delay to ensure scan stopped
        connect(result.device.address)
    }
}
```

---

## 7. Integration Checklist

### Vehicle App (Kotlin)
- [ ] Tạo folder `feature/ble`
- [ ] Copy và adapt `BleController.kt` → `BleAdvertisingController.kt`
- [ ] **Implement hàm `vehicleIdToUuid()` (REQUIRED)**
- [ ] Implement advertising với UUID only (không có service data, device name)
- [ ] Implement GATT Server để accept connections
- [ ] Implement permission request trong `startAdvertising()`
- [ ] Test advertising với vehicle_id
- [ ] Test connection từ Driver App
- [ ] **Không cần**: RSSI reading, logging, CSV export

### Driver App (Flutter Plugin)
- [ ] Tạo Flutter plugin structure
- [ ] Implement Dart API (`flutter_ble_logger.dart`)
- [ ] Implement Android side (`FlutterBleLoggerPlugin.kt`)
- [ ] **Implement hàm `vehicleIdToUuid()` (REQUIRED - phải giống Vehicle App)**
- [ ] Implement scanning với UUID filter
- [ ] Implement auto-connect logic (khi scan thấy UUID match)
- [ ] Implement permission request trong `startScanning()`
- [ ] Implement RSSI reading từ connection
- [ ] Implement session logging (SCAN_STARTED, CONNECTED, RSSI samples)
- [ ] Implement CSV export với vehicle_id và name
- [ ] Test scanning với vehicle_id
- [ ] Test auto-connect
- [ ] Test RSSI reading
- [ ] Test CSV export

### Integration Testing
- [ ] Test Vehicle advertising + Driver scanning trên 2 thiết bị
- [ ] Test UUID matching (verify cùng vehicle_id tạo ra cùng UUID)
- [ ] Test auto-connect (verify NO pairing dialog)
- [ ] Test RSSI monitoring trên Driver App
- [ ] Test CSV export trên Driver App với vehicle_id và name
- [ ] Test single device connection
- [ ] Test permission request flow
- [ ] Test Bluetooth on/off handling
- [ ] **Verify**: Vehicle App không ghi log, không export CSV

---

## 8. Notes

### 8.1. Vehicle ID Format & UUID Conversion
- vehicle_id là string thường (ví dụ: `"vehicle_123456"`), không cần UUID format
- Convert thành UUID chỉ để dùng cho BLE advertising/scanning
- **CRITICAL**: Cả 2 app phải dùng cùng hàm `vehicleIdToUuid()` để đảm bảo UUID match
- vehicle_id gốc vẫn được lưu trong CSV log (Driver App)

### 8.2. Advertising Data (UUID Only)
- **Chỉ advertise với UUID**: Không có service data, không có device name
- **Lý do**: Tránh lỗi "data too large" (BLE limit: 31 bytes)
- **UUID size**: 16 bytes (vừa đủ, không lo lỗi)
- **Auto-connect**: Dựa vào UUID match qua scan filter (không cần extract vehicle_id từ service data)

### 8.3. Logging & CSV Export
- **Vehicle App**: Không ghi log, không export CSV (chỉ start/stop advertising)
- **Driver App**: Ghi log và export CSV với format `time,type,value,name,vehicle_id`
- Name mặc định: `"Driver"` (hardcoded, không cần input)

### 8.4. RSSI Reading
- **Driver App**: Đọc RSSI từ connection (GATT Client) ✅
- **Vehicle App**: Không cần đọc RSSI (không ghi log)

### 8.5. Permission Request
- Permissions được request khi gọi `startAdvertising()` hoặc `startScanning()`
- Nếu user deny permissions, return error
- Không start advertising/scanning nếu thiếu permissions

---

## Kết luận

Codebase hiện tại đã có đủ foundation để implement:
- ✅ Advertising với UUID only (không có service data, device name)
- ✅ Scanning với UUID filter
- ✅ GATT Server/Client logic
- ✅ RSSI reading (Driver App)
- ✅ Session logging (Driver App)
- ✅ CSV export (Driver App)

### Key Points:
1. **UUID Conversion**: Cả 2 app phải dùng cùng hàm `vehicleIdToUuid()` để đảm bảo UUID match
2. **Advertising**: Chỉ UUID (tránh lỗi "data too large")
3. **Vehicle App**: Đơn giản, chỉ start/stop advertising, không ghi log
4. **Driver App**: Scanning + Auto-connect + Logging + CSV Export
5. **Auto-connect**: Dựa vào UUID match qua scan filter

### Implementation Checklist:
- ✅ UUID conversion function (`vehicleIdToUuid()`)
- ✅ Advertising với UUID only
- ✅ Scanning với UUID filter
- ✅ Auto-connect logic
- ✅ Permission request flow
- ✅ CSV format với vehicle_id và name (Driver App only)
