# Plan Game Module - Version 4 (Guest Storage Strategy)

## Thay đổi so với Version 3

1. **Làm rõ sự khác biệt** giữa Anonymous và Guest
2. **Đề xuất cách lưu Guest** - để guest có thể chơi lại nhiều lần với cùng identity

---

## I. PHÂN BIỆT CÁC LOẠI NGƯỜI CHƠI

### 1.1 Ba loại người chơi (Làm rõ)

| Loại | Có account? | Backend biết là ai? | Public hiện tên? | Lưu history? |
|------|-------------|---------------------|------------------|--------------|
| **Account** | ✅ Có | ✅ Có (user_id) | ✅ Tên thật | ✅ Vào profile |
| **Anonymous** | ✅ Có | ✅ Có (user_id) | ❌ Tên ẩn danh | ❌ Không hiện | 
| **Guest** | ❌ Không | ⚠️ Có guest_id | ✅ Tên tự nhập | ✅ Vào guest profile |

### 1.2 Sự khác biệt quan trọng

```
ANONYMOUS (Đã đăng nhập, chọn ẩn danh):
├── Backend: BIẾT là user_id = "abc123"
├── Leaderboard: Hiện "Người chơi bí ẩn" hoặc tên tự chọn "Siêu Nhân"
├── Sau game: KHÔNG lưu vào profile account
└── Mục đích: User có account nhưng không muốn bạn bè biết điểm thấp 😅

GUEST (Chưa đăng nhập):
├── Backend: CHỈ BIẾT guest_id = "guest_xyz789"
├── Leaderboard: Hiện tên tự nhập "Nguyễn Văn A"
├── Sau game: LƯU vào guest profile (localStorage + optional server)
└── Mục đích: Ai cũng chơi được, không cần đăng ký
```

---

## II. CHIẾN LƯỢC LƯU TRỮ GUEST

### 2.1 Đề xuất: Hybrid Storage (LocalStorage + Server)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     GUEST IDENTITY STORAGE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────┐      ┌─────────────────────┐              │
│  │   LOCAL STORAGE     │      │      SERVER         │              │
│  │   (Browser)         │      │   (PostgreSQL)      │              │
│  ├─────────────────────┤      ├─────────────────────┤              │
│  │ guest_id: "xyz789"  │ ───► │ game_guests table   │              │
│  │ guest_name: "Nam"   │      │ - id                │              │
│  │ guest_avatar: "🐱"  │      │ - guest_token       │              │
│  │ guest_token: "abc"  │      │ - name              │              │
│  └─────────────────────┘      │ - avatar            │              │
│                               │ - created_at        │              │
│  Lưu ngay khi nhập tên        │ - last_played_at    │              │
│  Dùng lại khi quay lại        └─────────────────────┘              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Luồng hoạt động

```
Lần đầu vào game:
1. Check localStorage → Không có guest_id
2. User nhập: Tên "Nam", Avatar "🐱"
3. Frontend tạo: guest_id = "guest_" + uuid()[:8]
4. Frontend tạo: guest_token = uuid() (dùng để verify)
5. Lưu localStorage: {guest_id, guest_name, guest_avatar, guest_token}
6. Gửi server: POST /api/game/guests (lưu vào DB)

Lần sau vào game:
1. Check localStorage → Có guest_id = "guest_xyz789"
2. Hiện popup: "Chào mừng trở lại, Nam! 🐱"
   - [ Tiếp tục với tên này ]
   - [ Đổi tên/avatar ]
3. Nếu chọn tiếp tục → Dùng lại guest_id cũ
4. Nếu đổi → Cập nhật cả localStorage và server
```

### 2.3 Database Schema cho Guest

```sql
-- Bảng lưu guest identity (optional, để sync across devices)
CREATE TABLE game_guests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Identity
    guest_token VARCHAR(64) UNIQUE NOT NULL,  -- Dùng để verify ownership
    guest_name VARCHAR(100) NOT NULL,
    guest_avatar VARCHAR(50) NOT NULL,
    
    -- Metadata
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_played_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    total_games INT DEFAULT 0,
    
    -- Optional: Link to user if they register later
    linked_user_id UUID REFERENCES user_infos(id)
);

-- Index cho lookup nhanh
CREATE INDEX idx_game_guests_token ON game_guests(guest_token);
```

### 2.4 localStorage Structure

```typescript
// Lưu trong localStorage
interface GuestIdentity {
    guest_id: string;      // "guest_xyz789"
    guest_token: string;   // UUID để verify với server
    guest_name: string;    // "Nguyễn Văn Nam"
    guest_avatar: string;  // "🐱"
    created_at: string;    // ISO date
}

// Lưu/đọc
const GUEST_KEY = "gkebook_game_guest";

function saveGuestIdentity(identity: GuestIdentity) {
    localStorage.setItem(GUEST_KEY, JSON.stringify(identity));
}

function getGuestIdentity(): GuestIdentity | null {
    const data = localStorage.getItem(GUEST_KEY);
    return data ? JSON.parse(data) : null;
}
```

---

## III. UI FLOW CẬP NHẬT

### 3.1 Khi Guest quay lại

```
┌─────────────────────────────────────────────────────────────┐
│                  CHÀO MỪNG TRỞ LẠI!                         │
│                                                              │
│     ┌───┐                                                    │
│     │🐱│  Nam                                               │
│     └───┘                                                    │
│                                                              │
│  Bạn đã chơi 5 game trước đó.                               │
│                                                              │
│  ┌──────────────────────────┐                               │
│  │  TIẾP TỤC VỚI TÊN NÀY   │                               │
│  └──────────────────────────┘                               │
│                                                              │
│  ┌──────────────────────────┐                               │
│  │  ĐỔI TÊN / AVATAR        │                               │
│  └──────────────────────────┘                               │
│                                                              │
│  ┌──────────────────────────┐                               │
│  │  TẠO PROFILE MỚI         │ ← Reset localStorage          │
│  └──────────────────────────┘                               │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Guest History Page (Optional feature)

```
┌─────────────────────────────────────────────────────────────┐
│  📊 LỊCH SỬ CHƠI CỦA BẠN                                    │
│                                                              │
│  ┌───┐ Nam 🐱                                               │
│  └───┘ Guest ID: guest_xyz789                               │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Game           │ Ngày       │ Xếp hạng │ Điểm          ││
│  ├─────────────────────────────────────────────────────────┤│
│  │ Quiz Toán Lớp 5│ 31/12/2024 │ 🥇 1/20  │ 850 điểm      ││
│  │ Quiz Văn Lớp 6 │ 30/12/2024 │ 🥉 3/15  │ 720 điểm      ││
│  │ Quiz Sử Việt   │ 29/12/2024 │    5/30  │ 650 điểm      ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  💡 Đăng ký tài khoản để lưu kết quả vĩnh viễn!            │
│  [  ĐĂNG KÝ NGAY  ]                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## IV. API ENDPOINTS CHO GUEST

### 4.1 Guest Management

```go
// Tạo guest mới
POST /api/game/guests
Request: {
    "guest_name": "Nam",
    "guest_avatar": "🐱"
}
Response: {
    "guest_id": "guest_xyz789",
    "guest_token": "abc123...",  // Lưu để verify sau
    "guest_name": "Nam",
    "guest_avatar": "🐱"
}

// Cập nhật guest info
PUT /api/game/guests
Headers: X-Guest-Token: abc123...
Request: {
    "guest_name": "Nam Mới",
    "guest_avatar": "🐶"
}

// Lấy guest history
GET /api/game/guests/history
Headers: X-Guest-Token: abc123...
Response: {
    "guest_info": {...},
    "games": [
        {"room_code": "123456", "rank": 1, "score": 850, "played_at": "..."},
        ...
    ]
}

// Link guest to account (khi đăng ký)
POST /api/game/guests/link
Headers: 
    X-Guest-Token: abc123...
    Authorization: Bearer <jwt>
Response: {
    "linked": true,
    "games_migrated": 5
}
```

---

## V. BẢNG TỔNG HỢP LƯU TRỮ

| Loại | Lưu ở đâu? | ID | Verify bằng? |
|------|------------|-----|--------------|
| **Account** | DB (user_infos) | user_id | JWT |
| **Anonymous** | DB (game_players) | anon_user_id | JWT |
| **Guest** | localStorage + DB (game_guests) | guest_id | guest_token |

---

## VI. MIGRATION PATH: GUEST → ACCOUNT

Khi guest quyết định đăng ký tài khoản:

```
1. Guest chơi 5 games với guest_token "abc123"
   ↓
2. Guest click "Đăng ký tài khoản"
   ↓
3. Tạo account mới với email/password
   ↓
4. Backend: UPDATE game_guests 
   SET linked_user_id = new_user_id 
   WHERE guest_token = "abc123"
   ↓
5. Backend: UPDATE game_players 
   SET user_id = new_user_id 
   WHERE player_id LIKE 'guest_%' 
   AND guest_token = "abc123"
   ↓
6. Tất cả game history của guest → chuyển vào account mới
```

---

## VII. TÓM TẮT V4

| Thay đổi | Mô tả |
|----------|-------|
| **Guest storage** | localStorage (primary) + Server (sync/backup) |
| **Guest identity** | guest_id + guest_token để verify |
| **Guest history** | Có thể xem lịch sử, migrate khi đăng ký |
| **Anonymous vs Guest** | Anonymous = có account, ẩn. Guest = không account |

---

*Version 4 - Làm rõ guest storage và migration path.*
