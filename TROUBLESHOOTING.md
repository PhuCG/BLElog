# Hướng dẫn khắc phục sự cố kết nối BLE

## Các vấn đề đã được sửa

### 1. **AndroidManifest.xml - Permissions cho Android 12+**
- ✅ Đã thêm `usesPermissionFlags="neverForLocation"` cho `BLUETOOTH_SCAN`
- ✅ Điều này bắt buộc trên Android 12+ để tránh yêu cầu quyền Location không cần thiết

### 2. **Xử lý lỗi kết nối**
- ✅ Đã cải thiện xử lý lỗi trong `gattCallback` để hiển thị mã lỗi chi tiết
- ✅ Thêm logging chi tiết để debug các vấn đề kết nối

### 3. **Hỗ trợ Android 12+**
- ✅ Đã cập nhật `connectGatt()` để sử dụng `TRANSPORT_LE` trên Android 12+
- ✅ Cải thiện xử lý exception khi kết nối

## Cách sử dụng app để kết nối 2 thiết bị

### Bước 1: Cài đặt và cấp quyền
1. Cài đặt app trên cả 2 thiết bị
2. Mở app trên cả 2 thiết bị
3. **Quan trọng**: App sẽ tự động yêu cầu permissions. Bạn cần:
   - Chấp nhận tất cả permissions được yêu cầu
   - Trên Android 12+: Chấp nhận `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`, `BLUETOOTH_ADVERTISE`
   - Trên Android 11 trở xuống: Chấp nhận `ACCESS_FINE_LOCATION`

### Bước 2: Bật Bluetooth
- Đảm bảo Bluetooth đã được bật trên cả 2 thiết bị
- Nếu chưa bật, app sẽ hiển thị nút "Enable" để bật Bluetooth

### Bước 3: Thiết lập Advertising và Scanning

**Trên Thiết bị A (Advertising):**
1. Đặt tên thiết bị (tùy chọn, mặc định sẽ có tên tự động)
2. **Bật switch "Advertising"** (nút màu xanh)
3. Đợi thông báo "ADVERTISING_STARTED" trong History Timeline

**Trên Thiết bị B (Scanning):**
1. **Bật switch "Scanning"** (nút màu tím)
2. Đợi vài giây để app quét và tìm thiết bị A
3. Thiết bị A sẽ xuất hiện trong danh sách "Nearby devices"

### Bước 4: Kết nối
1. Trên Thiết bị B, tìm thiết bị A trong danh sách "Nearby devices"
2. **Nhấn nút "Connect"** trên card của thiết bị A
3. Đợi kết nối thành công (sẽ hiển thị icon Bluetooth Connected)

## Lưu ý quan trọng

### ⚠️ Cả 2 thiết bị phải:
- ✅ Cùng sử dụng app này
- ✅ Cùng bật Bluetooth
- ✅ Cùng cấp đầy đủ permissions
- ✅ Cùng trong phạm vi gần nhau (khoảng cách tối đa ~10m)

### ⚠️ Quy trình kết nối:
1. **Thiết bị A**: Bật **Advertising** TRƯỚC
2. **Thiết bị B**: Bật **Scanning** SAU ĐÓ
3. **Thiết bị B**: Nhấn **Connect** khi thấy thiết bị A

### ⚠️ Nếu không thấy thiết bị:
1. Kiểm tra cả 2 thiết bị đã bật Bluetooth chưa
2. Kiểm tra cả 2 thiết bị đã cấp đủ permissions chưa
3. Đảm bảo thiết bị A đã bật Advertising TRƯỚC khi thiết bị B bật Scanning
4. Thử tắt và bật lại Advertising/Scanning
5. Kiểm tra khoảng cách giữa 2 thiết bị (nên trong vòng 5-10m)

## Debug và xử lý lỗi

### Xem History Timeline
- Scroll xuống phần "History Timeline" để xem các sự kiện:
  - `ADVERTISING_STARTED`: Advertising đã bắt đầu
  - `SCAN_STARTED`: Scanning đã bắt đầu
  - `CONNECT_REQUEST`: Đã yêu cầu kết nối
  - `CONNECTING`: Đang kết nối
  - `CONNECTED`: Kết nối thành công
  - `CONNECT_FAILED`: Kết nối thất bại (sẽ có mã lỗi chi tiết)

### Các mã lỗi thường gặp:
- **GATT_SUCCESS (0)**: Thành công
- **GATT_INSUFFICIENT_AUTHENTICATION (5)**: Thiếu xác thực
- **GATT_INSUFFICIENT_ENCRYPTION (15)**: Thiếu mã hóa
- **GATT_INTERNAL_ERROR (129)**: Lỗi nội bộ
- **GATT_WRONG_STATE (134)**: Trạng thái sai
- **GATT_BUSY (136)**: Đang bận

### Nếu vẫn không kết nối được:
1. Kiểm tra History Timeline để xem mã lỗi cụ thể
2. Thử restart cả 2 thiết bị
3. Thử xóa cache của app và cài đặt lại
4. Kiểm tra xem có app khác đang sử dụng Bluetooth không

## Yêu cầu hệ thống

- **Android**: 8.0 (API 26) trở lên
- **Bluetooth**: Hỗ trợ BLE (Bluetooth Low Energy)
- **Permissions**: 
  - Android 12+: BLUETOOTH_SCAN, BLUETOOTH_CONNECT, BLUETOOTH_ADVERTISE
  - Android 11 trở xuống: ACCESS_FINE_LOCATION

## Tính năng tự động reconnect

- Khi kết nối thành công, app sẽ tự động bật tính năng auto-reconnect
- Nếu kết nối bị ngắt (không phải do người dùng disconnect), app sẽ tự động thử kết nối lại sau 5 giây
- Để tắt auto-reconnect, nhấn nút "Disconnect"

