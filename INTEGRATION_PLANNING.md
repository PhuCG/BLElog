# Planning: Tích hợp BLE Logging vào Vehicle App (Kotlin) và Driver App (Flutter)

## Tổng quan

Sử dụng codebase BLE demo hiện tại để tích hợp vào 2 project:
1. **Vehicle App (Kotlin)**: Feature BLE Advertising để ghi log kết nối
2. **Driver App (Flutter Plugin)**: Feature BLE Scanning + Auto-connect để ghi log kết nối

**Mục đích**: Đo lường tín hiệu BLE giữa 2 thiết bị (vehicle ↔ driver)

## Auto-Connect BLE - Câu trả lời

### ✅ CÓ, Android cho phép tự động kết nối BLE mà KHÔNG CẦN pairing!

**Giải thích:**
- **BLE (Bluetooth Low Energy)** khác với **Bluetooth Classic**
- BLE **KHÔNG yêu cầu pairing/bonding** để kết nối
- Pairing chỉ cần khi:
  - GATT characteristic có security requirements (encrypted, authenticated)
  - App explicitly request bonding
- **Use case của bạn (chỉ đọc RSSI)**: KHÔNG CẦN pairing
- Auto-connect hoàn toàn khả thi và đã có sẵn trong codebase

**Flow tự động:**
```
Driver App: Start scan with UUID
    ↓
Tìm thấy Vehicle advertising với UUID
    ↓
Auto connect (KHÔNG cần user nhấn pair)
    ↓
Đọc RSSI mỗi 2 giây
    ↓
Ghi log
```

---

## 1. Vehicle App (Kotlin) - BLE Advertising Feature

### 1.1. Yêu cầu

**Core Features:**
- Tích hợp thành feature/service trong Kotlin project hiện có
- Advertising với UUID cố định
- GATT Server để accept connections
- Ghi log BLE cho duy nhất 1 thiết bị connected
- Export CSV khi kết thúc

**API Endpoints:**
```kotlin
// Start advertising và begin logging
fun startBleAdvertising(vehicleId: String, name: String)

// Stop advertising và export CSV
fun stopBleAdvertisingAndExport(): File? // Return CSV file
```

**CSV Format với 2 trường mới:**
```csv
time,type,value,name,vehicle_id
2024-01-15 10:30:00,status,"ADVERTISING_STARTED","Vehicle ABC","VH001"
2024-01-15 10:30:05,status,"CONNECTED AA:BB:CC:DD:EE:FF","Vehicle ABC","VH001"
2024-01-15 10:30:07,rssi,"-65","Vehicle ABC","VH001"
2024-01-15 10:30:09,rssi,"-67","Vehicle ABC","VH001"
```

**Logging Rules:**
- Chỉ log cho 1 device duy nhất (first connected device)
- Reject connections từ devices khác khi đã có 1 device connected
- Log tất cả events: ADVERTISING_STARTED, CONNECTED, DISCONNECTED, RSSI samples

### 1.2. Architecture

```kotlin
// Module structure
vehicle-app/
  └── ble/
      ├── BleAdvertisingService.kt       // Main service
      ├── BleAdvertisingController.kt    // BLE logic
      ├── BleSessionLogger.kt            // Session logging
      ├── BleCsvExporter.kt              // CSV export
      └── models/
          ├── BleConstants.kt            // UUID, constants
          └── BleLogEntry.kt             // Log data model
```

### 1.3. Implementation

#### BleAdvertisingService.kt
```kotlin
package com.yourapp.ble

import android.app.Service
import android.content.Intent
import android.os.Binder
import android.os.IBinder
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch
import java.io.File

/**
 * BLE Advertising Service for Vehicle App
 * 
 * Usage:
 *   val intent = Intent(context, BleAdvertisingService::class.java)
 *   bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)
 *   
 *   binder.startBleAdvertising(vehicleId = "VH001", name = "Vehicle ABC")
 *   ...
 *   val csvFile = binder.stopBleAdvertisingAndExport()
 */
class BleAdvertisingService : Service() {
    
    private val serviceJob = SupervisorJob()
    private val serviceScope = CoroutineScope(serviceJob + Dispatchers.Main.immediate)
    
    private lateinit var controller: BleAdvertisingController
    private lateinit var sessionLogger: BleSessionLogger
    private lateinit var csvExporter: BleCsvExporter
    
    private val _isAdvertising = MutableStateFlow(false)
    val isAdvertising: StateFlow<Boolean> = _isAdvertising
    
    private val _connectedDevice = MutableStateFlow<String?>(null)
    val connectedDevice: StateFlow<String?> = _connectedDevice
    
    private var currentVehicleId: String? = null
    private var currentName: String? = null
    
    inner class BleServiceBinder : Binder() {
        fun getService(): BleAdvertisingService = this@BleAdvertisingService
    }
    
    private val binder = BleServiceBinder()
    
    override fun onCreate() {
        super.onCreate()
        controller = BleAdvertisingController(applicationContext, serviceScope)
        sessionLogger = BleSessionLogger()
        csvExporter = BleCsvExporter(applicationContext)
        
        // Observe controller states
        serviceScope.launch {
            controller.isAdvertising.collect { advertising ->
                _isAdvertising.value = advertising
            }
        }
        
        serviceScope.launch {
            controller.connectedDevice.collect { device ->
                _connectedDevice.value = device?.address
                if (device != null) {
                    sessionLogger.logEvent(
                        type = "status",
                        value = "CONNECTED ${device.address}",
                        name = currentName,
                        vehicleId = currentVehicleId
                    )
                }
            }
        }
        
        // Observe RSSI updates
        serviceScope.launch {
            controller.rssiUpdates.collect { rssi ->
                sessionLogger.logRssi(
                    rssi = rssi,
                    name = currentName,
                    vehicleId = currentVehicleId
                )
            }
        }
    }
    
    override fun onBind(intent: Intent?): IBinder = binder
    
    override fun onDestroy() {
        super.onDestroy()
        controller.clear()
        serviceJob.cancel()
    }
    
    /**
     * Start BLE advertising và begin session logging
     * 
     * @param vehicleId ID của vehicle (e.g., "VH001")
     * @param name Tên của vehicle (e.g., "Vehicle ABC")
     */
    fun startBleAdvertising(vehicleId: String, name: String) {
        if (_isAdvertising.value) {
            throw IllegalStateException("Already advertising. Stop first.")
        }
        
        currentVehicleId = vehicleId
        currentName = name
        
        // Begin session
        sessionLogger.beginSession()
        sessionLogger.logEvent(
            type = "status",
            value = "SESSION_START",
            name = name,
            vehicleId = vehicleId
        )
        
        // Start advertising
        controller.startAdvertising(advertiseName = name)
        
        sessionLogger.logEvent(
            type = "status",
            value = "ADVERTISING_STARTED",
            name = name,
            vehicleId = vehicleId
        )
    }
    
    /**
     * Stop BLE advertising và export CSV
     * 
     * @return CSV file or null if no data
     */
    suspend fun stopBleAdvertisingAndExport(): File? {
        if (!_isAdvertising.value) {
            return null
        }
        
        // Stop advertising
        controller.stopAdvertising()
        
        sessionLogger.logEvent(
            type = "status",
            value = "ADVERTISING_STOPPED",
            name = currentName,
            vehicleId = currentVehicleId
        )
        
        // Wait for log to settle
        kotlinx.coroutines.delay(350)
        
        // End session
        sessionLogger.endSession()
        
        // Export CSV
        val logs = sessionLogger.getSessionLogs()
        val csvFile = csvExporter.exportToCsv(
            logs = logs,
            vehicleId = currentVehicleId ?: "unknown",
            name = currentName ?: "unknown"
        )
        
        // Clear session
        sessionLogger.clearSession()
        currentVehicleId = null
        currentName = null
        
        return csvFile
    }
    
    /**
     * Get current advertising status
     */
    fun isCurrentlyAdvertising(): Boolean = _isAdvertising.value
    
    /**
     * Get connected device address (null if no device connected)
     */
    fun getConnectedDeviceAddress(): String? = _connectedDevice.value
}
```

#### BleAdvertisingController.kt
```kotlin
package com.yourapp.ble

import android.bluetooth.*
import android.bluetooth.le.*
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.os.Handler
import android.os.Looper
import android.os.ParcelUuid
import androidx.core.content.ContextCompat
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asSharedFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import java.nio.charset.StandardCharsets
import java.util.UUID

/**
 * BLE Advertising Controller
 * Handles advertising, GATT server, and single device connection
 */
class BleAdvertisingController(
    context: Context,
    private val scope: CoroutineScope
) {
    private val appContext = context.applicationContext
    private val bluetoothManager =
        appContext.getSystemService(Context.BLUETOOTH_SERVICE) as? BluetoothManager
    private val bluetoothAdapter: BluetoothAdapter? = bluetoothManager?.adapter
    private val advertiser: BluetoothLeAdvertiser? = bluetoothAdapter?.bluetoothLeAdvertiser
    
    // States
    private val _isAdvertising = MutableStateFlow(false)
    val isAdvertising: StateFlow<Boolean> = _isAdvertising.asStateFlow()
    
    private val _connectedDevice = MutableStateFlow<ConnectedDevice?>(null)
    val connectedDevice: StateFlow<ConnectedDevice?> = _connectedDevice.asStateFlow()
    
    private val _rssiUpdates = MutableSharedFlow<Int>(extraBufferCapacity = 16)
    val rssiUpdates = _rssiUpdates.asSharedFlow()
    
    private val _errors = MutableSharedFlow<String>(extraBufferCapacity = 8)
    val errors = _errors.asSharedFlow()
    
    // GATT Server
    private var gattServer: BluetoothGattServer? = null
    private var infoCharacteristic: BluetoothGattCharacteristic? = null
    private var advertiseCallback: AdvertiseCallback? = null
    
    // RSSI reading (for connected device)
    private var connectedGatt: BluetoothGatt? = null
    private val mainHandler = Handler(Looper.getMainLooper())
    private val rssiReader = object : Runnable {
        override fun run() {
            val gatt = connectedGatt
            if (gatt != null && _connectedDevice.value != null) {
                gatt.readRemoteRssi()
                mainHandler.postDelayed(this, RSSI_POLL_INTERVAL_MS)
            }
        }
    }
    
    private val bluetoothStateReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context?, intent: Intent?) {
            if (intent?.action == BluetoothAdapter.ACTION_STATE_CHANGED) {
                val state = intent.getIntExtra(BluetoothAdapter.EXTRA_STATE, BluetoothAdapter.ERROR)
                if (state != BluetoothAdapter.STATE_ON) {
                    stopAdvertising()
                }
            }
        }
    }
    
    init {
        ContextCompat.registerReceiver(
            appContext,
            bluetoothStateReceiver,
            IntentFilter(BluetoothAdapter.ACTION_STATE_CHANGED),
            ContextCompat.RECEIVER_NOT_EXPORTED
        )
    }
    
    @Suppress("MissingPermission")
    fun startAdvertising(advertiseName: String) {
        if (_isAdvertising.value) return
        if (advertiser == null) {
            _errors.tryEmit("BLE advertiser unavailable on this device.")
            return
        }
        
        ensureGattServer(advertiseName)
        
        val settings = AdvertiseSettings.Builder()
            .setAdvertiseMode(AdvertiseSettings.ADVERTISE_MODE_LOW_LATENCY)
            .setTxPowerLevel(AdvertiseSettings.ADVERTISE_TX_POWER_HIGH)
            .setConnectable(true)
            .build()
        
        val serviceBytes = advertiseName
            .toByteArray(StandardCharsets.UTF_8)
            .let { if (it.size > MAX_SERVICE_DATA_BYTES) it.copyOf(MAX_SERVICE_DATA_BYTES) else it }
        
        val advertiseData = AdvertiseData.Builder()
            .setIncludeDeviceName(true)
            .addServiceUuid(ParcelUuid(SERVICE_UUID))
            .apply {
                if (serviceBytes.isNotEmpty()) {
                    addServiceData(ParcelUuid(SERVICE_UUID), serviceBytes)
                }
            }
            .build()
        
        val scanResponse = AdvertiseData.Builder()
            .setIncludeTxPowerLevel(true)
            .build()
        
        advertiseCallback = object : AdvertiseCallback() {
            override fun onStartSuccess(settingsInEffect: AdvertiseSettings) {
                _isAdvertising.value = true
            }
            
            override fun onStartFailure(errorCode: Int) {
                _errors.tryEmit("Advertise start failed: ${advertiseErrorDescription(errorCode)}")
                _isAdvertising.value = false
            }
        }
        
        try {
            advertiser.startAdvertising(settings, advertiseData, scanResponse, advertiseCallback)
        } catch (sec: SecurityException) {
            _errors.tryEmit("Missing permission for advertising.")
        }
    }
    
    @Suppress("MissingPermission")
    fun stopAdvertising() {
        val callback = advertiseCallback ?: return
        try {
            advertiser?.stopAdvertising(callback)
        } catch (_: SecurityException) {
        }
        advertiseCallback = null
        _isAdvertising.value = false
        closeGattServer()
        
        // Disconnect connected device
        connectedGatt?.disconnect()
        connectedGatt?.close()
        connectedGatt = null
        mainHandler.removeCallbacks(rssiReader)
        _connectedDevice.value = null
    }
    
    @Suppress("MissingPermission")
    private fun ensureGattServer(advertiseName: String) {
        if (gattServer != null) {
            infoCharacteristic?.value = advertiseName.toByteArray(StandardCharsets.UTF_8)
            return
        }
        val server = bluetoothManager?.openGattServer(appContext, gattServerCallback)
        if (server == null) {
            _errors.tryEmit("Unable to open GATT server.")
            return
        }
        val service = BluetoothGattService(
            SERVICE_UUID,
            BluetoothGattService.SERVICE_TYPE_PRIMARY
        )
        val infoChar = BluetoothGattCharacteristic(
            INFO_CHARACTERISTIC_UUID,
            BluetoothGattCharacteristic.PROPERTY_READ or BluetoothGattCharacteristic.PROPERTY_NOTIFY,
            BluetoothGattCharacteristic.PERMISSION_READ
        )
        infoChar.value = advertiseName.toByteArray(StandardCharsets.UTF_8)
        service.addCharacteristic(infoChar)
        server.addService(service)
        gattServer = server
        infoCharacteristic = infoChar
    }
    
    @Suppress("MissingPermission")
    private val gattServerCallback = object : BluetoothGattServerCallback() {
        override fun onConnectionStateChange(device: BluetoothDevice?, status: Int, newState: Int) {
            device ?: return
            val address = device.address
            
            when (newState) {
                BluetoothProfile.STATE_CONNECTED -> {
                    // Check if already have a connected device
                    if (_connectedDevice.value != null) {
                        // Already have a connection, reject this one
                        scope.launch {
                            _errors.emit("Connection rejected: Already connected to another device")
                        }
                        gattServer?.cancelConnection(device)
                        return
                    }
                    
                    // Accept this connection
                    _connectedDevice.value = ConnectedDevice(
                        address = address,
                        name = device.name ?: address,
                        connectedAt = System.currentTimeMillis()
                    )
                    
                    // Start RSSI monitoring
                    // Note: GATT server doesn't directly provide RSSI
                    // We need to connect as client to read RSSI
                    connectedGatt = device.connectGatt(appContext, false, gattClientCallback)
                    
                    scope.launch {
                        _errors.emit("Device connected: ${device.name ?: address}")
                    }
                }
                BluetoothProfile.STATE_DISCONNECTED -> {
                    if (_connectedDevice.value?.address == address) {
                        _connectedDevice.value = null
                        mainHandler.removeCallbacks(rssiReader)
                        connectedGatt?.close()
                        connectedGatt = null
                        scope.launch {
                            _errors.emit("Device disconnected: ${device.name ?: address}")
                        }
                    }
                }
            }
        }
        
        override fun onCharacteristicReadRequest(
            device: BluetoothDevice,
            requestId: Int,
            offset: Int,
            characteristic: BluetoothGattCharacteristic
        ) {
            val server = gattServer ?: return
            if (characteristic.uuid == INFO_CHARACTERISTIC_UUID) {
                val value = infoCharacteristic?.value ?: ByteArray(0)
                server.sendResponse(device, requestId, BluetoothGatt.GATT_SUCCESS, offset, value)
            } else {
                server.sendResponse(device, requestId, BluetoothGatt.GATT_FAILURE, offset, null)
            }
        }
    }
    
    // GATT client callback để đọc RSSI
    @Suppress("MissingPermission")
    private val gattClientCallback = object : BluetoothGattCallback() {
        override fun onConnectionStateChange(gatt: BluetoothGatt?, status: Int, newState: Int) {
            when (newState) {
                BluetoothProfile.STATE_CONNECTED -> {
                    gatt?.discoverServices()
                    mainHandler.post(rssiReader)
                }
                BluetoothProfile.STATE_DISCONNECTED -> {
                    mainHandler.removeCallbacks(rssiReader)
                }
            }
        }
        
        override fun onReadRemoteRssi(gatt: BluetoothGatt, rssi: Int, status: Int) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                _rssiUpdates.tryEmit(rssi)
            }
        }
    }
    
    private fun closeGattServer() {
        gattServer?.close()
        gattServer = null
        infoCharacteristic = null
    }
    
    fun clear() {
        stopAdvertising()
        runCatching {
            appContext.unregisterReceiver(bluetoothStateReceiver)
        }
    }
    
    private fun advertiseErrorDescription(code: Int): String = when (code) {
        AdvertiseCallback.ADVERTISE_FAILED_DATA_TOO_LARGE -> "data too large (error 1)"
        AdvertiseCallback.ADVERTISE_FAILED_TOO_MANY_ADVERTISERS -> "too many advertisers (error 2)"
        AdvertiseCallback.ADVERTISE_FAILED_ALREADY_STARTED -> "already started (error 3)"
        AdvertiseCallback.ADVERTISE_FAILED_INTERNAL_ERROR -> "internal error (error 4)"
        AdvertiseCallback.ADVERTISE_FAILED_FEATURE_UNSUPPORTED -> "feature unsupported (error 5)"
        else -> "unknown error ($code)"
    }
    
    companion object {
        val SERVICE_UUID: UUID = UUID.fromString("0000feed-0000-1000-8000-00805f9b34fb")
        val INFO_CHARACTERISTIC_UUID: UUID = UUID.fromString("0000beef-0000-1000-8000-00805f9b34fb")
        const val RSSI_POLL_INTERVAL_MS = 2000L
        const val MAX_SERVICE_DATA_BYTES = 12
    }
}

data class ConnectedDevice(
    val address: String,
    val name: String,
    val connectedAt: Long
)
```

#### BleSessionLogger.kt
```kotlin
package com.yourapp.ble

import java.util.concurrent.locks.ReentrantLock
import kotlin.concurrent.withLock

/**
 * Session logger for BLE events and RSSI samples
 */
class BleSessionLogger {
    
    private val lock = ReentrantLock()
    private val sessionLog = mutableListOf<BleLogEntry>()
    private var sessionActive = false
    
    fun beginSession() {
        lock.withLock {
            sessionLog.clear()
            sessionActive = true
        }
    }
    
    fun endSession() {
        lock.withLock {
            sessionActive = false
        }
    }
    
    fun logEvent(type: String, value: String, name: String?, vehicleId: String?) {
        if (!sessionActive) return
        lock.withLock {
            sessionLog.add(
                BleLogEntry(
                    timestampMillis = System.currentTimeMillis(),
                    type = type,
                    value = value,
                    name = name,
                    vehicleId = vehicleId
                )
            )
        }
    }
    
    fun logRssi(rssi: Int, name: String?, vehicleId: String?) {
        if (!sessionActive) return
        lock.withLock {
            sessionLog.add(
                BleLogEntry(
                    timestampMillis = System.currentTimeMillis(),
                    type = "rssi",
                    value = rssi.toString(),
                    name = name,
                    vehicleId = vehicleId
                )
            )
        }
    }
    
    fun getSessionLogs(): List<BleLogEntry> {
        return lock.withLock {
            sessionLog.toList()
        }
    }
    
    fun clearSession() {
        lock.withLock {
            sessionLog.clear()
            sessionActive = false
        }
    }
}

data class BleLogEntry(
    val timestampMillis: Long,
    val type: String, // "status" or "rssi"
    val value: String,
    val name: String?,
    val vehicleId: String?
)
```

#### BleCsvExporter.kt
```kotlin
package com.yourapp.ble

import android.content.Context
import java.io.File
import java.time.Instant
import java.time.ZoneId
import java.time.format.DateTimeFormatter
import java.util.Locale

/**
 * CSV Exporter for BLE logs
 */
class BleCsvExporter(private val context: Context) {
    
    fun exportToCsv(logs: List<BleLogEntry>, vehicleId: String, name: String): File? {
        if (logs.isEmpty()) return null
        
        return runCatching {
            val exportsDir = File(context.getExternalFilesDir(null), "ble_exports")
            if (!exportsDir.exists()) {
                exportsDir.mkdirs()
            }
            
            val timestamp = DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss", Locale.US)
                .format(Instant.now().atZone(ZoneId.systemDefault()))
            val file = File(exportsDir, "ble_vehicle_${vehicleId}_$timestamp.csv")
            
            val formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss", Locale.getDefault())
            
            file.bufferedWriter().use { writer ->
                // Header with name and vehicle_id columns
                writer.appendLine("time,type,value,name,vehicle_id")
                
                logs.forEach { entry ->
                    val time = Instant.ofEpochMilli(entry.timestampMillis)
                        .atZone(ZoneId.systemDefault())
                        .format(formatter)
                    val escapedValue = entry.value.replace("\"", "\"\"")
                    val escapedName = entry.name?.replace("\"", "\"\"") ?: ""
                    val escapedVehicleId = entry.vehicleId?.replace("\"", "\"\"") ?: ""
                    
                    writer.appendLine("$time,${entry.type},\"$escapedValue\",\"$escapedName\",\"$escapedVehicleId\"")
                }
            }
            
            file
        }.getOrNull()
    }
}
```

### 1.4. Usage Example

```kotlin
// In your Activity/Fragment
class VehicleActivity : AppCompatActivity() {
    
    private var bleService: BleAdvertisingService? = null
    private val serviceConnection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            val binder = service as BleAdvertisingService.BleServiceBinder
            bleService = binder.getService()
        }
        
        override fun onServiceDisconnected(name: ComponentName?) {
            bleService = null
        }
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Bind to service
        val intent = Intent(this, BleAdvertisingService::class.java)
        bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)
        
        // Start advertising
        findViewById<Button>(R.id.btnStart).setOnClickListener {
            bleService?.startBleAdvertising(
                vehicleId = "VH001",
                name = "Vehicle ABC"
            )
        }
        
        // Stop and export
        findViewById<Button>(R.id.btnStop).setOnClickListener {
            lifecycleScope.launch {
                val csvFile = bleService?.stopBleAdvertisingAndExport()
                if (csvFile != null) {
                    Toast.makeText(this@VehicleActivity, "Exported: ${csvFile.absolutePath}", Toast.LENGTH_LONG).show()
                    // Share file or upload to server
                    shareFile(csvFile)
                }
            }
        }
    }
    
    override fun onDestroy() {
        super.onDestroy()
        unbindService(serviceConnection)
    }
    
    private fun shareFile(file: File) {
        val uri = FileProvider.getUriForFile(this, "${packageName}.fileprovider", file)
        val intent = Intent(Intent.ACTION_SEND).apply {
            type = "text/csv"
            putExtra(Intent.EXTRA_STREAM, uri)
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }
        startActivity(Intent.createChooser(intent, "Share BLE Log"))
    }
}
```

### 1.5. AndroidManifest.xml

```xml
<manifest>
    <!-- Permissions -->
    <uses-permission android:name="android.permission.BLUETOOTH" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
    <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE" />
    
    <application>
        <!-- Service -->
        <service
            android:name=".ble.BleAdvertisingService"
            android:exported="false" />
        
        <!-- FileProvider for CSV sharing -->
        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="${applicationId}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>
    </application>
</manifest>
```

---

## 2. Driver App (Flutter Plugin) - BLE Scanning + Auto-Connect

### 2.1. Yêu cầu

**Core Features:**
- Flutter plugin để tích hợp vào Flutter project
- Scanning với UUID filter
- Auto-connect khi tìm thấy UUID (KHÔNG CẦN pairing)
- Ghi log BLE cho duy nhất 1 thiết bị connected
- Export CSV khi kết thúc

**API Endpoints (Dart):**
```dart
// Start scanning và auto-connect
Future<void> startBleScanning({
  required String targetUuid,
  required String vehicleId,
  required String name,
});

// Stop scanning và export CSV
Future<String?> stopBleScanningAndExport(); // Return CSV file path
```

**CSV Format:**
```csv
time,type,value,name,vehicle_id
2024-01-15 10:30:00,status,"SCAN_STARTED","Driver XYZ","VH001"
2024-01-15 10:30:05,status,"DEVICE_FOUND AA:BB:CC:DD:EE:FF","Driver XYZ","VH001"
2024-01-15 10:30:06,status,"CONNECTED AA:BB:CC:DD:EE:FF","Driver XYZ","VH001"
2024-01-15 10:30:08,rssi,"-65","Driver XYZ","VH001"
2024-01-15 10:30:10,rssi,"-67","Driver XYZ","VH001"
```

**Logging Rules:**
- Chỉ log cho 1 device duy nhất (target device)
- Reject/ignore connections từ devices khác
- Auto-connect khi tìm thấy UUID
- Log tất cả events: SCAN_STARTED, DEVICE_FOUND, CONNECTED, DISCONNECTED, RSSI samples

### 2.2. Plugin Structure

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
  │       └── FlutterBleLoggerPlugin.swift (iOS implementation)
  ├── lib/
  │   └── flutter_ble_logger.dart
  └── pubspec.yaml
```

### 2.3. Implementation

#### flutter_ble_logger.dart (Dart side)
```dart
import 'dart:async';
import 'package:flutter/services.dart';

class FlutterBleLogger {
  static const MethodChannel _channel =
      MethodChannel('com.yourapp.flutter_ble_logger');
  
  static const EventChannel _eventChannel =
      EventChannel('com.yourapp.flutter_ble_logger/events');
  
  /// Start BLE scanning và auto-connect khi tìm thấy target UUID
  /// 
  /// [targetUuid] - UUID của vehicle cần connect (e.g., "0000feed-0000-1000-8000-00805f9b34fb")
  /// [vehicleId] - ID của vehicle (e.g., "VH001")
  /// [name] - Tên của driver/session (e.g., "Driver XYZ")
  static Future<void> startBleScanning({
    required String targetUuid,
    required String vehicleId,
    required String name,
  }) async {
    try {
      await _channel.invokeMethod('startBleScanning', {
        'targetUuid': targetUuid,
        'vehicleId': vehicleId,
        'name': name,
      });
    } on PlatformException catch (e) {
      throw Exception('Failed to start BLE scanning: ${e.message}');
    }
  }
  
  /// Stop BLE scanning và export CSV
  /// 
  /// Returns: CSV file path or null if no data
  static Future<String?> stopBleScanningAndExport() async {
    try {
      final String? filePath = await _channel.invokeMethod('stopBleScanningAndExport');
      return filePath;
    } on PlatformException catch (e) {
      throw Exception('Failed to stop BLE scanning: ${e.message}');
    }
  }
  
  /// Check if currently scanning
  static Future<bool> isScanning() async {
    try {
      final bool isScanning = await _channel.invokeMethod('isScanning');
      return isScanning;
    } on PlatformException catch (e) {
      throw Exception('Failed to check scanning status: ${e.message}');
    }
  }
  
  /// Get connected device address (null if not connected)
  static Future<String?> getConnectedDevice() async {
    try {
      final String? address = await _channel.invokeMethod('getConnectedDevice');
      return address;
    } on PlatformException catch (e) {
      throw Exception('Failed to get connected device: ${e.message}');
    }
  }
  
  /// Stream of BLE events (connected, disconnected, error, etc.)
  static Stream<BleEvent> get eventsStream {
    return _eventChannel.receiveBroadcastStream().map((event) {
      final Map<dynamic, dynamic> map = event as Map<dynamic, dynamic>;
      return BleEvent(
        type: map['type'] as String,
        message: map['message'] as String?,
        data: map['data'] as Map<dynamic, dynamic>?,
      );
    });
  }
}

class BleEvent {
  final String type; // "connected", "disconnected", "error", "rssi_update"
  final String? message;
  final Map<dynamic, dynamic>? data;
  
  BleEvent({required this.type, this.message, this.data});
  
  @override
  String toString() => 'BleEvent(type: $type, message: $message, data: $data)';
}
```

#### FlutterBleLoggerPlugin.kt (Android side)
```kotlin
package com.yourapp.flutter_ble_logger

import android.content.Context
import androidx.annotation.NonNull
import io.flutter.embedding.engine.plugins.FlutterPlugin
import io.flutter.plugin.common.EventChannel
import io.flutter.plugin.common.MethodCall
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugin.common.MethodChannel.MethodCallHandler
import io.flutter.plugin.common.MethodChannel.Result
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.launch
import kotlinx.coroutines.cancel

class FlutterBleLoggerPlugin : FlutterPlugin, MethodCallHandler, EventChannel.StreamHandler {
    
    private lateinit var methodChannel: MethodChannel
    private lateinit var eventChannel: EventChannel
    private lateinit var context: Context
    
    private val pluginScope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
    private var scannerController: BleScannerController? = null
    private var sessionLogger: BleSessionLogger? = null
    private var csvExporter: BleCsvExporter? = null
    
    private var eventSink: EventChannel.EventSink? = null
    
    private var currentVehicleId: String? = null
    private var currentName: String? = null
    
    override fun onAttachedToEngine(@NonNull flutterPluginBinding: FlutterPlugin.FlutterPluginBinding) {
        context = flutterPluginBinding.applicationContext
        
        methodChannel = MethodChannel(flutterPluginBinding.binaryMessenger, "com.yourapp.flutter_ble_logger")
        methodChannel.setMethodCallHandler(this)
        
        eventChannel = EventChannel(flutterPluginBinding.binaryMessenger, "com.yourapp.flutter_ble_logger/events")
        eventChannel.setStreamHandler(this)
    }
    
    override fun onMethodCall(@NonNull call: MethodCall, @NonNull result: Result) {
        when (call.method) {
            "startBleScanning" -> {
                val targetUuid = call.argument<String>("targetUuid")
                val vehicleId = call.argument<String>("vehicleId")
                val name = call.argument<String>("name")
                
                if (targetUuid == null || vehicleId == null || name == null) {
                    result.error("INVALID_ARGUMENTS", "Missing required arguments", null)
                    return
                }
                
                startBleScanning(targetUuid, vehicleId, name)
                result.success(null)
            }
            "stopBleScanningAndExport" -> {
                pluginScope.launch {
                    try {
                        val filePath = stopBleScanningAndExport()
                        result.success(filePath)
                    } catch (e: Exception) {
                        result.error("EXPORT_FAILED", e.message, null)
                    }
                }
            }
            "isScanning" -> {
                result.success(scannerController?.isScanning() ?: false)
            }
            "getConnectedDevice" -> {
                result.success(scannerController?.getConnectedDevice())
            }
            else -> {
                result.notImplemented()
            }
        }
    }
    
    private fun startBleScanning(targetUuid: String, vehicleId: String, name: String) {
        // Initialize components
        scannerController = BleScannerController(context, pluginScope)
        sessionLogger = BleSessionLogger()
        csvExporter = BleCsvExporter(context)
        
        currentVehicleId = vehicleId
        currentName = name
        
        // Begin session
        sessionLogger?.beginSession()
        sessionLogger?.logEvent("status", "SESSION_START", name, vehicleId)
        
        // Observe events
        pluginScope.launch {
            scannerController?.events?.collect { event ->
                when (event) {
                    is BleScannerEvent.ScanStarted -> {
                        sessionLogger?.logEvent("status", "SCAN_STARTED", currentName, currentVehicleId)
                        sendEvent("scan_started", "Scanning started", null)
                    }
                    is BleScannerEvent.DeviceFound -> {
                        sessionLogger?.logEvent("status", "DEVICE_FOUND ${event.address}", currentName, currentVehicleId)
                        sendEvent("device_found", "Device found: ${event.address}", mapOf("address" to event.address))
                    }
                    is BleScannerEvent.Connected -> {
                        sessionLogger?.logEvent("status", "CONNECTED ${event.address}", currentName, currentVehicleId)
                        sendEvent("connected", "Connected to ${event.address}", mapOf("address" to event.address))
                    }
                    is BleScannerEvent.Disconnected -> {
                        sessionLogger?.logEvent("status", "DISCONNECTED ${event.address}", currentName, currentVehicleId)
                        sendEvent("disconnected", "Disconnected from ${event.address}", mapOf("address" to event.address))
                    }
                    is BleScannerEvent.RssiUpdate -> {
                        sessionLogger?.logRssi(event.rssi, currentName, currentVehicleId)
                        sendEvent("rssi_update", "RSSI: ${event.rssi} dBm", mapOf("rssi" to event.rssi))
                    }
                    is BleScannerEvent.Error -> {
                        sessionLogger?.logEvent("status", "ERROR: ${event.message}", currentName, currentVehicleId)
                        sendEvent("error", event.message, null)
                    }
                }
            }
        }
        
        // Start scanning
        scannerController?.startScan(targetUuid)
    }
    
    private suspend fun stopBleScanningAndExport(): String? {
        // Stop scanning
        scannerController?.stopScan()
        
        sessionLogger?.logEvent("status", "SCAN_STOPPED", currentName, currentVehicleId)
        
        // Wait for log to settle
        kotlinx.coroutines.delay(350)
        
        // End session
        sessionLogger?.endSession()
        
        // Export CSV
        val logs = sessionLogger?.getSessionLogs() ?: emptyList()
        val csvFile = csvExporter?.exportToCsv(
            logs = logs,
            vehicleId = currentVehicleId ?: "unknown",
            name = currentName ?: "unknown"
        )
        
        // Clear session
        sessionLogger?.clearSession()
        scannerController?.clear()
        scannerController = null
        sessionLogger = null
        currentVehicleId = null
        currentName = null
        
        return csvFile?.absolutePath
    }
    
    private fun sendEvent(type: String, message: String?, data: Map<String, Any?>?) {
        pluginScope.launch(Dispatchers.Main) {
            eventSink?.success(mapOf(
                "type" to type,
                "message" to message,
                "data" to data
            ))
        }
    }
    
    override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
        eventSink = events
    }
    
    override fun onCancel(arguments: Any?) {
        eventSink = null
    }
    
    override fun onDetachedFromEngine(@NonNull binding: FlutterPlugin.FlutterPluginBinding) {
        methodChannel.setMethodCallHandler(null)
        eventChannel.setStreamHandler(null)
        pluginScope.cancel()
    }
}
```

#### BleScannerController.kt (Android - Scanner + Auto-Connect)
```kotlin
package com.yourapp.flutter_ble_logger

import android.bluetooth.*
import android.bluetooth.le.*
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.os.Handler
import android.os.Looper
import android.os.ParcelUuid
import androidx.core.content.ContextCompat
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.asSharedFlow
import kotlinx.coroutines.launch
import java.util.UUID

sealed class BleScannerEvent {
    object ScanStarted : BleScannerEvent()
    data class DeviceFound(val address: String, val name: String?) : BleScannerEvent()
    data class Connected(val address: String) : BleScannerEvent()
    data class Disconnected(val address: String) : BleScannerEvent()
    data class RssiUpdate(val rssi: Int) : BleScannerEvent()
    data class Error(val message: String) : BleScannerEvent()
}

class BleScannerController(
    context: Context,
    private val scope: CoroutineScope
) {
    private val appContext = context.applicationContext
    private val bluetoothManager =
        appContext.getSystemService(Context.BLUETOOTH_SERVICE) as? BluetoothManager
    private val bluetoothAdapter: BluetoothAdapter? = bluetoothManager?.adapter
    
    private val _events = MutableSharedFlow<BleScannerEvent>(extraBufferCapacity = 16)
    val events = _events.asSharedFlow()
    
    private var isScanning = false
    private var connectedAddress: String? = null
    private var targetUuid: String? = null
    
    private var currentGatt: BluetoothGatt? = null
    private val mainHandler = Handler(Looper.getMainLooper())
    
    private val rssiReader = object : Runnable {
        override fun run() {
            val gatt = currentGatt
            if (gatt != null && connectedAddress != null) {
                gatt.readRemoteRssi()
                mainHandler.postDelayed(this, RSSI_POLL_INTERVAL_MS)
            }
        }
    }
    
    private val bluetoothStateReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context?, intent: Intent?) {
            if (intent?.action == BluetoothAdapter.ACTION_STATE_CHANGED) {
                val state = intent.getIntExtra(BluetoothAdapter.EXTRA_STATE, BluetoothAdapter.ERROR)
                if (state != BluetoothAdapter.STATE_ON) {
                    stopScan()
                }
            }
        }
    }
    
    private val scanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult?) {
            result ?: return
            
            // Check if already connected
            if (connectedAddress != null) return
            
            val device = result.device
            _events.tryEmit(BleScannerEvent.DeviceFound(device.address, device.name))
            
            // Auto-connect to this device
            stopScan()
            connect(device)
        }
        
        override fun onScanFailed(errorCode: Int) {
            isScanning = false
            _events.tryEmit(BleScannerEvent.Error("Scan failed: $errorCode"))
        }
    }
    
    private val gattCallback = object : BluetoothGattCallback() {
        override fun onConnectionStateChange(gatt: BluetoothGatt?, status: Int, newState: Int) {
            when (newState) {
                BluetoothProfile.STATE_CONNECTED -> {
                    // Check if already connected to another device
                    if (connectedAddress != null && connectedAddress != gatt?.device?.address) {
                        // Reject this connection
                        _events.tryEmit(BleScannerEvent.Error("Already connected to another device"))
                        gatt?.disconnect()
                        return
                    }
                    
                    connectedAddress = gatt?.device?.address
                    gatt?.discoverServices()
                    mainHandler.post(rssiReader)
                    _events.tryEmit(BleScannerEvent.Connected(connectedAddress!!))
                }
                BluetoothProfile.STATE_DISCONNECTED -> {
                    mainHandler.removeCallbacks(rssiReader)
                    val address = connectedAddress ?: gatt?.device?.address ?: "unknown"
                    connectedAddress = null
                    currentGatt?.close()
                    currentGatt = null
                    _events.tryEmit(BleScannerEvent.Disconnected(address))
                }
            }
        }
        
        override fun onReadRemoteRssi(gatt: BluetoothGatt, rssi: Int, status: Int) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                _events.tryEmit(BleScannerEvent.RssiUpdate(rssi))
            }
        }
    }
    
    init {
        ContextCompat.registerReceiver(
            appContext,
            bluetoothStateReceiver,
            IntentFilter(BluetoothAdapter.ACTION_STATE_CHANGED),
            ContextCompat.RECEIVER_NOT_EXPORTED
        )
    }
    
    @Suppress("MissingPermission")
    fun startScan(targetUuidString: String) {
        if (isScanning) return
        
        targetUuid = targetUuidString
        val scanner = bluetoothAdapter?.bluetoothLeScanner
        if (scanner == null) {
            _events.tryEmit(BleScannerEvent.Error("BLE scanner unavailable"))
            return
        }
        
        val uuid = UUID.fromString(targetUuidString)
        val filter = ScanFilter.Builder()
            .setServiceUuid(ParcelUuid(uuid))
            .build()
        
        val settings = ScanSettings.Builder()
            .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
            .build()
        
        try {
            scanner.startScan(listOf(filter), settings, scanCallback)
            isScanning = true
            _events.tryEmit(BleScannerEvent.ScanStarted)
        } catch (sec: SecurityException) {
            _events.tryEmit(BleScannerEvent.Error("Missing permission for scanning"))
        }
    }
    
    @Suppress("MissingPermission")
    fun stopScan() {
        if (!isScanning) return
        val scanner = bluetoothAdapter?.bluetoothLeScanner ?: return
        try {
            scanner.stopScan(scanCallback)
        } catch (_: SecurityException) {
        }
        isScanning = false
    }
    
    @Suppress("MissingPermission")
    private fun connect(device: BluetoothDevice) {
        // Check if already connected
        if (connectedAddress != null) {
            _events.tryEmit(BleScannerEvent.Error("Already connected to a device"))
            return
        }
        
        try {
            currentGatt = device.connectGatt(appContext, false, gattCallback)
        } catch (sec: SecurityException) {
            _events.tryEmit(BleScannerEvent.Error("Missing permission to connect"))
        }
    }
    
    @Suppress("MissingPermission")
    fun clear() {
        stopScan()
        currentGatt?.disconnect()
        currentGatt?.close()
        currentGatt = null
        mainHandler.removeCallbacks(rssiReader)
        connectedAddress = null
        runCatching {
            appContext.unregisterReceiver(bluetoothStateReceiver)
        }
    }
    
    fun isScanning(): Boolean = isScanning
    
    fun getConnectedDevice(): String? = connectedAddress
    
    companion object {
        const val RSSI_POLL_INTERVAL_MS = 2000L
    }
}
```

### 2.4. Usage Example (Flutter)

```dart
import 'package:flutter/material.dart';
import 'package:flutter_ble_logger/flutter_ble_logger.dart';

class DriverScreen extends StatefulWidget {
  @override
  _DriverScreenState createState() => _DriverScreenState();
}

class _DriverScreenState extends State<DriverScreen> {
  bool _isScanning = false;
  String? _connectedDevice;
  String _status = 'Idle';
  
  @override
  void initState() {
    super.initState();
    
    // Listen to BLE events
    FlutterBleLogger.eventsStream.listen((event) {
      setState(() {
        _status = '${event.type}: ${event.message ?? ''}';
        if (event.type == 'connected') {
          _connectedDevice = event.data?['address'];
        } else if (event.type == 'disconnected') {
          _connectedDevice = null;
        }
      });
    });
  }
  
  Future<void> _startScanning() async {
    try {
      await FlutterBleLogger.startBleScanning(
        targetUuid: '0000feed-0000-1000-8000-00805f9b34fb',
        vehicleId: 'VH001',
        name: 'Driver XYZ',
      );
      setState(() => _isScanning = true);
    } catch (e) {
      print('Error: $e');
    }
  }
  
  Future<void> _stopAndExport() async {
    try {
      final csvPath = await FlutterBleLogger.stopBleScanningAndExport();
      setState(() => _isScanning = false);
      
      if (csvPath != null) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Exported: $csvPath')),
        );
        // Share file or upload to server
      }
    } catch (e) {
      print('Error: $e');
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('BLE Driver App')),
      body: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Text('Status: $_status'),
            SizedBox(height: 16),
            Text('Connected: ${_connectedDevice ?? 'None'}'),
            SizedBox(height: 32),
            ElevatedButton(
              onPressed: _isScanning ? null : _startScanning,
              child: Text('Start Scanning'),
            ),
            SizedBox(height: 16),
            ElevatedButton(
              onPressed: _isScanning ? _stopAndExport : null,
              child: Text('Stop & Export CSV'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## 3. Tóm tắt Flow

### 3.1. Vehicle App Flow (Kotlin)

```
User nhấn Start
  ↓
startBleAdvertising(vehicleId: "VH001", name: "Vehicle ABC")
  ↓
Begin session logging
  ↓
Start BLE advertising với UUID: 0000feed-0000-1000-8000-00805f9b34fb
  ↓
GATT Server lắng nghe incoming connections
  ↓
Driver app connect đến (KHÔNG CẦN pairing)
  ↓
Accept connection (reject nếu đã có connection khác)
  ↓
Start RSSI monitoring (mỗi 2 giây)
  ↓
Log events: CONNECTED, RSSI samples
  ↓
User nhấn Stop
  ↓
stopBleAdvertisingAndExport()
  ↓
Stop advertising
  ↓
Disconnect device
  ↓
Export CSV với columns: time, type, value, name, vehicle_id
  ↓
Return CSV file để share/upload
```

### 3.2. Driver App Flow (Flutter)

```
User nhấn Start
  ↓
startBleScanning(
  targetUuid: "0000feed-0000-1000-8000-00805f9b34fb",
  vehicleId: "VH001",
  name: "Driver XYZ"
)
  ↓
Begin session logging
  ↓
Start BLE scanning với UUID filter
  ↓
Tìm thấy Vehicle advertising với UUID
  ↓
Auto connect đến Vehicle (KHÔNG CẦN pairing)
  ↓
Connection established (reject nếu đã có connection khác)
  ↓
Start RSSI monitoring (mỗi 2 giây)
  ↓
Log events: SCAN_STARTED, DEVICE_FOUND, CONNECTED, RSSI samples
  ↓
User nhấn Stop
  ↓
stopBleScanningAndExport()
  ↓
Stop scanning
  ↓
Disconnect từ Vehicle
  ↓
Export CSV với columns: time, type, value, name, vehicle_id
  ↓
Return CSV file path để share/upload
```

---

## 4. Key Points

### 4.1. Auto-Connect

✅ **Auto-connect KHÔNG CẦN pairing vì:**
- BLE ≠ Bluetooth Classic
- BLE connection không yêu cầu user interaction
- Pairing chỉ cần khi có encryption requirements
- Use case này (chỉ đọc RSSI) không cần encryption

### 4.2. Single Device Connection

**Vehicle App:**
- GATT Server accept 1 connection duy nhất
- Reject connections khi đã có device connected
- Log: `CONNECTION_REJECTED (already connected)`

**Driver App:**
- Scanner chỉ connect đến 1 device duy nhất (target UUID)
- Ignore/reject connections khác
- Log: `CONNECT_REJECTED (already connected)`

### 4.3. CSV Format

Cả 2 app đều export CSV với format:
```csv
time,type,value,name,vehicle_id
2024-01-15 10:30:00,status,"SESSION_START","Vehicle ABC","VH001"
2024-01-15 10:30:05,status,"CONNECTED AA:BB:CC:DD:EE:FF","Vehicle ABC","VH001"
2024-01-15 10:30:07,rssi,"-65","Vehicle ABC","VH001"
```

**Columns:**
- `time`: Timestamp (yyyy-MM-dd HH:mm:ss)
- `type`: "status" hoặc "rssi"
- `value`: Event value hoặc RSSI value
- `name`: Tên được truyền vào khi start (Vehicle ABC / Driver XYZ)
- `vehicle_id`: Vehicle ID được truyền vào khi start (VH001)

### 4.4. Integration

**Vehicle App:**
- Tích hợp như một feature/service trong Kotlin project
- Bind to `BleAdvertisingService`
- Call `startBleAdvertising()` và `stopBleAdvertisingAndExport()`

**Driver App:**
- Tích hợp như Flutter plugin
- Add dependency trong `pubspec.yaml`
- Call `FlutterBleLogger.startBleScanning()` và `stopBleScanningAndExport()`

---

## 5. Implementation Checklist

### Vehicle App (Kotlin)
- [ ] Copy `BleAdvertisingService.kt`, `BleAdvertisingController.kt`, `BleSessionLogger.kt`, `BleCsvExporter.kt`
- [ ] Add permissions trong AndroidManifest.xml
- [ ] Add FileProvider configuration
- [ ] Integrate service binding trong Activity/Fragment
- [ ] Test advertising
- [ ] Test single device connection
- [ ] Test CSV export với name và vehicle_id

### Driver App (Flutter Plugin)
- [ ] Create Flutter plugin structure
- [ ] Implement Dart side (`flutter_ble_logger.dart`)
- [ ] Implement Android side (`FlutterBleLoggerPlugin.kt`, `BleScannerController.kt`)
- [ ] Implement iOS side (if needed)
- [ ] Add permissions trong AndroidManifest.xml
- [ ] Test scanning
- [ ] Test auto-connect (NO pairing required)
- [ ] Test single device connection
- [ ] Test CSV export với name và vehicle_id

### Integration Testing
- [ ] Test Vehicle advertising + Driver scanning trên 2 thiết bị
- [ ] Test auto-connect (verify NO pairing dialog)
- [ ] Test RSSI monitoring
- [ ] Test CSV export trên cả 2 app
- [ ] Test single device connection (reject multiple connections)
- [ ] Test Bluetooth on/off handling

---

## Kết luận

**Auto-connect hoàn toàn khả thi mà KHÔNG CẦN pairing!**

Use case của bạn (Vehicle advertising + Driver scanning/auto-connect để đo RSSI) là một use case điển hình của BLE và không yêu cầu user interaction (pairing).

Codebase demo hiện tại đã có đủ logic để implement:
- Vehicle app: Reuse advertising + GATT server logic
- Driver app: Reuse scanning + auto-connect logic

Chỉ cần thêm:
- 2 fields mới trong CSV (name, vehicle_id)
- API endpoints rõ ràng (start/stop)
- Single device connection enforcement

