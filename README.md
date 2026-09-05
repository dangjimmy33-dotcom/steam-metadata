# Steam Metadata & Depot Manifest Historical Tracker

Bộ công cụ tự động quét, lưu trữ và theo dõi biến động lịch sử phiên bản (`BuildID`, `Branches`, `Depot Manifest GID`) cho toàn bộ game trên Steam (~184k game).

---

## Tính Năng Cốt Lõi

1. **Lưu trữ Append-Only (Bảo toàn lịch sử vĩnh viễn)**:
   - Khi phát hiện bản cập nhật mới (`BuildID` mới hoặc Depot Manifest GID thay đổi), công cụ sẽ bổ sung bản ghi vào danh sách `history` thay vì ghi đè.
   - Giữ nguyên toàn bộ các bản vá cũ của game để Launcher / Depot Downloader sau này có thể tra cứu và tải lại các phiên bản cũ tùy thích.

2. **Modulo Sharding (`appid % 1000`)**:
   - Dữ liệu được chia đều vào 1,000 thư mục con (`data/000/` đến `data/999/`).
   - Mỗi thư mục chỉ chứa trung bình ~180 file `.json`, hoàn toàn tránh được hiện tượng đơ giật file explorer trên Windows và lỗi quá tải index của Git / GitHub / Hugging Face.
   - Cho phép Launcher tra cứu file trực tiếp với tốc độ $O(1)$:
     ```javascript
     const shard = String(appId % 1000).padStart(3, '0');
     const url = `https://raw.githubusercontent.com/YOUR_REPO/main/data/${shard}/${appId}.json`;
     ```

3. **Chống Rate Limit & Checkpoint/Resume An Toàn**:
   - Tự động bắt mã lỗi HTTP 429 và backoff lũy thừa (Exponential Backoff với Jitter) để không bị chặn IP.
   - Lưu tiến độ quét liên tục vào `cache/checkpoint.json`. Người dùng có thể nhấn `Ctrl+C` dừng bất kỳ lúc nào và chạy tiếp mà không bị quét lặp lại.

---

## Cấu Trúc Thư Mục

```text
tools/steam_manifest_tracker/
├── data/                    # Dữ liệu phân mảnh theo shard (appid % 1000)
│   ├── 360/                 # Shard 360
│   │   └── 945360.json      # File metadata Among Us
│   └── ...
├── cache/
│   ├── applist.json         # Danh sách toàn bộ AppID Steam (~165k-184k)
│   └── checkpoint.json      # Trạng thái checkpoint để resume
├── tracker.py               # Script chính
├── fetcher.py               # Bộ gọi mạng async có backoff & semaphore
├── storage.py               # Quản lý lưu trữ shard & append-only
├── applist_provider.py      # Bộ tải & cache danh sách AppID
├── run_tracker.bat          # File chạy tương tác 1-click
└── README.md
```

---

## Hướng Dẫn Sử Dụng

### Cách 1: Sử Dụng Menu 1-Click
Nhấp đúp vào file `run_tracker.bat`. Bạn sẽ có các lựa chọn:
- `1`: Quét thử các game tiêu biểu (Among Us, CS2, Terraria, Dota 2, RE Requiem).
- `2`: Chạy quét toàn bộ kho Steam (Tự động resume từ vị trí đã dừng).
- `3`: Quét thử nghiệm 50 game đầu tiên.
- `4`: Làm mới danh sách AppList từ server.
- `5`: Xem thống kê số lượng file và shard đã lưu.

### Cách 2: Chạy Bằng Command Line

1. **Quét thử nghiệm game chỉ định**:
   ```bash
   python tracker.py --appids 945360,730,105600
   ```

2. **Quét 100 game mẫu**:
   ```bash
   python tracker.py --limit 100 --concurrency 5
   ```

3. **Chạy quét toàn bộ thư viện (Background Scan)**:
   ```bash
   python tracker.py --run --concurrency 5
   ```

4. **Xem thống kê dữ liệu hiện tại**:
   ```bash
   python tracker.py --stats
   ```

5. **Làm mới AppList với Steam Web API Key của bạn (nếu có)**:
   ```bash
   python tracker.py --fetch-applist --steam-key YOUR_STEAM_API_KEY
   ```

---

## Cấu Trúc JSON Chuẩn Của Mỗi Game (`{appid}.json`)

Tool lưu trữ theo mô hình **Hybrid (100% Full Raw + `_history`)**:

```json
{
  "_lastChecked": 1788614553,
  "_history": [
    {
      "buildId": "24302054",
      "branch": "public",
      "timeUpdated": 1787072415,
      "firstSeen": 1788614553,
      "manifests": {
        "945361": {
          "gid": "1397756378225229500",
          "size": 1115879348,
          "download": 656543488
        }
      }
    }
  ],
  "appid": "945360",
  "common": {
    "name": "Among Us",
    "type": "Game",
    "icon": "b82c3f46da...",
    "clienticon": "4096637...",
    "oslist": "windows,macos",
    "languages": { ... }
  },
  "config": {
    "installdir": "Among Us",
    "launch": {
      "0": {
        "executable": "Among Us.exe",
        "config": { "oslist": "windows" }
      }
    }
  },
  "depots": { ... },
  "extended": { ... },
  "ufs": { ... }
}
```
- **`_history`**: Bảo toàn vĩnh viễn toàn bộ lịch sử các version cũ khi game update (Append-Only).
- **Phần Raw (`common`, `config`, `depots`, `ufs`, `extended`)**: Giữ nguyên 100% dữ liệu gốc của Steam để Launcher đọc tên file `.exe`, thư mục cài, Cloud Save, icon/logo nét căng!
