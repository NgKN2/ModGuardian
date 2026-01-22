# ModGuardian v2.0

**Advanced Mod Detection Plugin for Minecraft**  
By NKN

---

## 🎯 Tổng quan

ModGuardian là plugin phát hiện mod nâng cao sử dụng kỹ thuật **Translation Key Analysis** để xác định người chơi đang sử dụng mod bị cấm.

### Đặc điểm nổi bật
-  Phát hiện mod thông qua Translation Key
-  Kiểm tra đa lớp (Multi-layer detection)
-  Cache kết quả để tối ưu hiệu suất
-  Whitelist hệ thống
-  Lịch sử phát hiện chi tiết
-  Thống kê đầy đủ
-  Hỗ trợ Fork: Paper/Spigot/... (unsupported folia)

---

## 🔧 Cơ chế kỹ thuật

### 1. Translation Key là gì?

**Translation Key** là chuỗi định danh mà Minecraft sử dụng để dịch văn bản sang nhiều ngôn ngữ.

Ví dụ:
```
item.minecraft.diamond_sword → "Diamond Sword"
key.meteor-client.open-gui → "Open GUI" (nếu có mod)
```

### 2. Cách phát hiện hoạt động

#### Bước 1: Gửi Client-Side Sign
```
Server → Client: Tạo sign ảo với translation keys
```

Plugin tạo một sign block chỉ client thấy (không ảnh hưởng server) với nội dung là các translation keys của mod:

```
Line 1: {translatable:"key.meteor-client.open-gui"} {translatable:"key.wurst.zoom"}
Line 2: {translatable:"baritone.overlay.cancel"} ...
```

#### Bước 2: Mở Sign Editor
```
Server → Client: WrapperPlayServerOpenSignEditor
```

Yêu cầu client mở giao diện chỉnh sửa sign. Minecraft client sẽ tự động dịch tất cả translation keys.

#### Bước 3: Client Response
```
Client → Server: WrapperPlayClientUpdateSign
```

Client gửi lại nội dung sign sau khi đã dịch:
- **Nếu KHÔNG có mod**: Keys được dịch → "key.meteor-client.open-gui"
- **Nếu CÓ mod**: Keys được dịch → "Open GUI"

#### Bước 4: Phân tích
```
if (receivedText.contains("key.meteor-client.open-gui")) {
    // Key CHƯA được dịch → mod KHÔNG tồn tại
} else {
    // Key ĐÃ được dịch → mod ĐANG được cài
    detectedMods.add("MeteorClient");
}
```

### 3. Multi-Layer Check

Plugin sử dụng nhiều lớp kiểm tra để tăng độ chính xác:

1. **Primary Key Check**: Kiểm tra key chính của mod
2. **Secondary Keys Check**: Kiểm tra các key phụ
3. **Pattern Matching**: Phát hiện key bị chia nhỏ qua nhiều dòng

### 4. Tối ưu hóa

#### Cache System
```
Player join → Kiểm tra cache
├─ Có cache → Sử dụng kết quả cũ (30 phút)
└─ Không cache → Thực hiện kiểm tra mới
```

#### Batch Processing
```
40+ mods → Chia thành nhiều batch
├─ Batch 1: Mod 1-40
├─ Batch 2: Mod 41-80
└─ ...
```

---

## 📦 Cài đặt

### Yêu cầu
- Java 17+
- Spigot/Paper 1.20.4+ (hoặc các fork khác)
- PacketEvents 2.9.5+

### Các bước cài đặt

1. **Download PacketEvents**
```bash
https://github.com/retrooper/packetevents/releases
```

2. **Cài đặt**
```
/plugins/
├── PacketEvents.jar
└── ModGuardian-1.0.jar
```

3. **Khởi động server**

---

## 🎮 Sử dụng

### Lệnh cơ bản

```bash
# Kiểm tra player
/mg check <player>

# Xem thông tin
/mg info <player>

# Xem lịch sử
/mg history <player>

# Thống kê
/mg stats

# Reload config
/mg reload

# Xóa cache
/mg clearcache <player>

# Xóa lịch sử
/mg clearhistory <player>

# Test (chỉ player)
/mg test
```

### Permissions

```yaml
modguardian.* - Tất cả quyền
modguardian.admin - Sử dụng lệnh admin
modguardian.notify - Nhận thông báo
modguardian.bypass - Bypass kiểm tra
```

---

## ⚙️ Cấu hình

### Auto Check On Join

```yaml
settings:
  check-on-join:
    enabled: true
    delay-ticks: 60  # 3 giây sau khi join
```

**Lý do delay**:
- Client cần thời gian để load hoàn toàn
- Tránh false positive
- Giảm lag khi nhiều người join cùng lúc

### Freeze While Checking

```yaml
settings:
  freeze-while-checking: true
```

Đóng băng player trong quá trình kiểm tra để:
- Tránh player thoát giữa chừng
- Chống exploit teleport
- Đảm bảo packet được nhận đầy đủ

### Cache System

```yaml
settings:
  cache:
    enabled: true
    duration-minutes: 30
```

**Hoạt động**:
- Lưu kết quả kiểm tra trong 30 phút
- Tự động xóa cache hết hạn
- Giảm tải server khi check nhiều lần

### Thêm mod mới

```yaml
banned-mods:
  TenModCuaBan:
    primary-key: "translation.key.main"
    secondary-keys:
      - "translation.key.secondary1"
      - "translation.key.secondary2"
    severity: "HIGH"
    description: "Mô tả mod"
```

**Tìm translation key**:
1. Cài mod vào client
2. Mở `.minecraft/mods/yourmod.jar`
3. Tìm file `assets/modid/lang/en_us.json`
4. Copy key từ file JSON

---

## 📊 Thống kê & Database

### Cấu trúc database (data.yml)

```yaml
detection-history:
  uuid-player-1:
    - player: "PlayerName"
      mods: ["MeteorClient", "Wurst"]
      timestamp: 1234567890
  uuid-player-2:
    - player: "Player2"
      mods: ["Baritone"]
      timestamp: 1234567891

statistics:
  mods:
    MeteorClient: 15
    Wurst: 8
    Baritone: 3
```

---

## 🐛 Troubleshooting

### Vấn đề: False Positive

**Nguyên nhân**: Translation key trùng với mod khác

**Giải pháp**:
1. Thêm secondary keys để kiểm tra chính xác hơn
2. Enable `multi-layer-check: true`
3. Kiểm tra lại translation key

### Vấn đề: Player timeout

**Nguyên nhân**: Client lag hoặc packet loss

**Giải pháp**:
```yaml
settings:
  check-timeout-seconds: 15  # Tăng timeout
```

### Vấn đề: Cache không hoạt động

**Kiểm tra**:
```yaml
settings:
  cache:
    enabled: true  # Đảm bảo đã bật
```

---

## 🔒 Bảo mật

### Bypass Protection

Plugin bảo vệ khỏi các bypass attempts:
- ✅ Fake translation (client gửi fake response)
- ✅ Packet manipulation
- ✅ Sign edit exploit
- ✅ Timeout exploit

### Recommendations

1. **Sử dụng whitelist** cho staff/admin
2. **Kết hợp với anti-cheat khác** để tăng hiệu quả
3. **Thường xuyên update** danh sách mod
4. **Monitor logs** để phát hiện pattern mới

---

## 📈 Performance

### Benchmark

Tested on Paper 1.20.4:
- **RAM Usage**: ~10MB
- **CPU Impact**: < 1% (idle)
- **Check Time**: 100-500ms per player
- **Concurrent Checks**: 20+ players

### Optimization Tips

```yaml
advanced:
  max-keys-per-check: 40  # Giảm nếu server lag
  max-batches: 10         # Giới hạn batch
```

---

## 🤝 Contributing

### Thêm mod mới

Tạo Pull Request với format:

```yaml
ModName:
  primary-key: "mod.key.main"
  secondary-keys: []
  severity: "HIGH/MEDIUM/LOW"
  description: "Short description"
```

### Report bugs

Tạo Issue với thông tin:
- Server version
- Plugin version
- Error log
- Steps to reproduce

---

## 📝 Changelog

### v2.0 (Current)
- ✨ Hoàn toàn viết lại từ đầu
- ✨ Multi-layer detection
- ✨ Cache system
- ✨ Database manager
- ✨ Advanced statistics
- ✨ Better performance

### v1.0
- ✅ Basic translation key detection
- ✅ Simple punishment system

---

## 📄 License

MIT License - Free to use and modify

---

## 👤 Author

**NKN**  
Discord: br.justdoit_
GitHub: Unknown

---

## ⭐ Credits

- **PacketEvents**: retrooper
- **Testing**: ModGuardian Community

---

**Made with ❤️ for Minecraft Server Owners**
