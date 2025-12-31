# Plan Game Module - Version 2 (Module Độc Lập)

## Tổng quan

Xây dựng **Game Module** hoàn toàn độc lập, có thể:
1. Chạy standalone (ai cũng chơi được, không cần đăng nhập)
2. Tích hợp vào gkebook-school-frontend (giáo viên tạo game cho học sinh)

---

## I. KIẾN TRÚC MODULE ĐỘC LẬP

### 1.1 Tách biệt hoàn toàn

```
prj-gkebook-go-backend/
├── modules/
│   ├── quizzes/           # Module Quiz cũ (giữ nguyên)
│   └── game/              # 🆕 Module Game MỚI (độc lập)
│       ├── application/   # Business logic
│       ├── contracts/     # DTOs, interfaces
│       ├── domain/        # Repository
│       └── presentations/ # API endpoints

prj-gkebook-game-frontend/ # 🆕 Frontend riêng cho Game
├── src/
│   ├── pages/
│   │   ├── create/        # Giao diện HOST
│   │   ├── play/          # Giao diện PLAYER
│   │   └── join/          # Màn nhập tên + chọn avatar
│   └── components/
```

### 1.2 Hai loại chạy

| Mode | Mô tả | Auth |
|------|-------|------|
| **Standalone** | Ai cũng chơi được, không cần account | Guest với tên + avatar |
| **Integrated** | Giáo viên tạo, học sinh chơi | Dùng JWT từ school-frontend |

---

## II. SO SÁNH GIAO THỨC REAL-TIME

### 2.1 Phân tích chi tiết

| Tiêu chí | WebSocket | MQTT | gRPC |
|----------|-----------|------|------|
| **Latency** | Thấp (~10ms) | Rất thấp (~5ms) | Thấp (~10ms) |
| **Concurrent users** | 10k-50k/server | **Hàng triệu**/cluster | 10k-50k/server |
| **Browser support** | ✅ Native | ❌ Cần bridge | ❌ Cần gRPC-web |
| **Complexity** | Thấp | Trung bình | Cao |
| **Phù hợp** | Real-time game | IoT, Chat, Broadcast | Microservices |

### 2.2 Khuyến nghị

**🎯 Cho Quiz Game (không phải twitch-game):**

| Quy mô | Khuyến nghị | Lý do |
|--------|-------------|-------|
| **< 1,000 players** | **WebSocket** | Đơn giản, browser native, đủ dùng |
| **1,000 - 10,000** | **WebSocket + Redis** | Scale horizontal với Pub/Sub |
| **> 10,000** | **MQTT** | Thiết kế cho massive concurrency |

**Trả lời sếp:**
> *"WebSocket không cần thiết và gây quá tải"*

**Phản biện:**
1. Quiz game **không phải** twitch-game (FPS, MOBA) cần UDP
2. WebSocket + optimization (batching, throttling) xử lý tốt 1000+ players
3. MQTT chủ yếu cho IoT, không có browser support native
4. gRPC cho backend services, không phù hợp client-server game

**Nếu sếp vẫn muốn thử MQTT:**
- Cần thêm MQTT-over-WebSocket bridge cho browser
- Thêm MQTT broker (EMQX, Mosquitto)
- Phức tạp hơn nhưng scale tốt hơn ở quy mô lớn

---

## III. THIẾT KẾ GIAO DIỆN

### 3.1 Luồng Host (Người tạo game)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. CHỌN QUIZ                                                 │
│    ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│    │  Quiz 1  │ │  Quiz 2  │ │  Quiz 3  │ ...               │
│    └──────────┘ └──────────┘ └──────────┘                   │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. CONFIG GAME                                               │
│    ┌─────────────────────────────────────────────────────┐  │
│    │ Số người chơi tối đa: [  100  ▼]                    │  │
│    │ Mode: [● Classic] [ ] Team (coming soon)            │  │
│    │ Thời gian mỗi câu: [  30s  ▼]                       │  │
│    │ Hiện đáp án đúng sau mỗi câu: [✓]                   │  │
│    └─────────────────────────────────────────────────────┘  │
│                                                              │
│    [  TẠO PHÒNG  ]                                          │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. WAITING ROOM                                              │
│    ┌───────────────────────────────────────────────────────┐│
│    │         MÃ PHÒNG: 123456                              ││
│    │         [COPY] [QR CODE]                              ││
│    │         game.gkebook.vn/join/123456                   ││
│    └───────────────────────────────────────────────────────┘│
│                                                              │
│    Người chơi (5/100):                                       │
│    ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐                            │
│    │🐱│ │🐶│ │🦊│ │🐼│ │🐨│                                 │
│    │Nam│ │Hùng││Linh││Khoa││Minh│                           │
│    └───┘ └───┘ └───┘ └───┘ └───┘                            │
│                                                              │
│    [  BẮT ĐẦU GAME  ]                                       │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Luồng Player (Người chơi)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. NHẬP MÃ PHÒNG                                             │
│    ┌─────────────────────────────────────────────────────┐  │
│    │                                                      │  │
│    │     Nhập mã phòng: [ 123456 ]                       │  │
│    │                                                      │  │
│    │     [  VÀO PHÒNG  ]                                 │  │
│    │                                                      │  │
│    └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. NHẬP TÊN + CHỌN AVATAR                                    │
│    ┌─────────────────────────────────────────────────────┐  │
│    │  Tên của bạn: [ Nguyễn Văn A ]                      │  │
│    │                                                      │  │
│    │  Chọn avatar:                                        │  │
│    │  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐   │  │
│    │  │🐱│ │🐶│ │🦊│ │🐼│ │🐨│ │🦁│ │🐯│ │🐸│          │  │
│    │  └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘   │  │
│    │         ↑                                            │  │
│    │      (Đang chọn)                                     │  │
│    │                                                      │  │
│    │  [  THAM GIA  ]                                     │  │
│    └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. HƯỚNG DẪN LUẬT CHƠI                                       │
│    ┌─────────────────────────────────────────────────────┐  │
│    │  📜 LUẬT CHƠI                                        │  │
│    │  ──────────────────────────────────────────────────  │  │
│    │  • Mỗi câu hỏi có thời gian giới hạn                 │  │
│    │  • Trả lời ĐÚNG và NHANH sẽ được điểm cao hơn        │  │
│    │  • Trả lời SAI hoặc HẾT GIỜ = 0 điểm                 │  │
│    │  • Người có tổng điểm cao nhất sẽ thắng              │  │
│    │                                                      │  │
│    │  [  ✓ ĐÃ HIỂU, SẴN SÀNG!  ]                         │  │
│    └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. WAITING (Đợi Host bắt đầu)                                │
│    ┌─────────────────────────────────────────────────────┐  │
│    │          ⏳ Đang đợi Host bắt đầu game...            │  │
│    │                                                      │  │
│    │  Người chơi trong phòng: 5                           │  │
│    │  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐                      │  │
│    │  │🐱│ │🐶│ │🦊│ │🐼│ │🐨│                            │  │
│    │  │Bạn│ │Hùng││Linh││Khoa││Minh│                      │  │
│    │  └───┘ └───┘ └───┘ └───┘ └───┘                      │  │
│    └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## IV. XỬ LÝ EDGE CASES

### 4.1 Nhiều người submit cùng lúc

**Vấn đề:** 1000 người nhấn nút cùng 1 giây, ai trước?

**Giải pháp: Server Timestamp**
```go
type SubmitAnswer struct {
    UserId      string
    QuestionId  string
    ChoiceId    string
    ClientTime  time.Time  // Thời gian client gửi (không tin)
    ServerTime  time.Time  // ✅ Server ghi nhận lúc nhận được
}

// Xếp hạng theo ServerTime
// Ai server nhận trước = submit trước
```

**Cách tính điểm công bằng:**
```
Điểm = ĐiểmCơBản × (1 - ThờiGianTrảLời/ThờiGianToiDa)

VD: Câu 100 điểm, max 30 giây
- Trả lời đúng sau 5 giây: 100 × (1 - 5/30) = 83 điểm
- Trả lời đúng sau 15 giây: 100 × (1 - 15/30) = 50 điểm
- Trả lời đúng sau 25 giây: 100 × (1 - 25/30) = 17 điểm
```

### 4.2 Quá nhiều người vào phòng

**Giải pháp:**

```go
type RoomConfig struct {
    MaxPlayers int  // Host config: 10, 50, 100, 500, 1000
}

func JoinRoom(roomCode, userId string) error {
    room := GetRoom(roomCode)
    
    if len(room.Players) >= room.Config.MaxPlayers {
        return errors.New("ROOM_FULL")  // Trả về lỗi rõ ràng
    }
    
    room.AddPlayer(userId)
    return nil
}
```

**UX khi phòng đầy:**
```
┌─────────────────────────────────┐
│         😅 PHÒNG ĐÃ ĐẦY         │
│                                  │
│  Phòng này đã đạt 100/100       │
│  người chơi tối đa.             │
│                                  │
│  [  THỬ PHÒNG KHÁC  ]           │
└─────────────────────────────────┘
```

---

## V. DATABASE SCHEMA

### 5.1 Bảng mới cho module độc lập

```sql
-- Bảng cấu hình phòng game
CREATE TABLE game_rooms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    room_code VARCHAR(10) UNIQUE NOT NULL,
    
    -- Quiz data (copy từ quiz hoặc standalone)
    quiz_data JSONB NOT NULL,  -- Lưu questions+choices để độc lập
    
    -- Host info (có thể null nếu guest host)
    host_user_id UUID REFERENCES user_infos(id),
    host_name VARCHAR(100),    -- Tên nếu là guest
    
    -- Config
    max_players INT DEFAULT 100,
    time_per_question INT DEFAULT 30,  -- seconds
    game_mode VARCHAR(20) DEFAULT 'classic',
    show_answers BOOLEAN DEFAULT true,
    
    -- Status
    status VARCHAR(20) DEFAULT 'waiting',  -- waiting, playing, finished
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    started_at TIMESTAMP WITH TIME ZONE,
    ended_at TIMESTAMP WITH TIME ZONE
);

-- Bảng người chơi (có thể là guest)
CREATE TABLE game_players (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id UUID NOT NULL REFERENCES game_rooms(id) ON DELETE CASCADE,
    
    -- Player info
    user_id UUID REFERENCES user_infos(id),  -- NULL nếu guest
    player_name VARCHAR(100) NOT NULL,        -- Tên hiển thị
    avatar VARCHAR(50) NOT NULL,              -- Avatar code (🐱, 🐶, etc.)
    
    -- Result
    total_score DECIMAL(10,2) DEFAULT 0,
    correct_count INT DEFAULT 0,
    wrong_count INT DEFAULT 0,
    final_rank INT,
    
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(room_id, player_name)  -- Không trùng tên trong 1 phòng
);

-- Danh sách avatars
CREATE TABLE game_avatars (
    id SERIAL PRIMARY KEY,
    code VARCHAR(10) NOT NULL,      -- 🐱
    name VARCHAR(50) NOT NULL,      -- "Mèo"
    category VARCHAR(30),           -- "animal", "emoji", "custom"
    is_active BOOLEAN DEFAULT true
);

-- Insert default avatars
INSERT INTO game_avatars (code, name, category) VALUES
('🐱', 'Mèo', 'animal'),
('🐶', 'Chó', 'animal'),
('🦊', 'Cáo', 'animal'),
('🐼', 'Gấu trúc', 'animal'),
('🐨', 'Koala', 'animal'),
('🦁', 'Sư tử', 'animal'),
('🐯', 'Hổ', 'animal'),
('🐸', 'Ếch', 'animal'),
('🐵', 'Khỉ', 'animal'),
('🐰', 'Thỏ', 'animal'),
('🦄', 'Kỳ lân', 'fantasy'),
('🐲', 'Rồng', 'fantasy');
```

---

## VI. TÍCH HỢP VỚI GKEBOOK-SCHOOL

### 6.1 Hai cách sử dụng

```
┌──────────────────────────────────────────────────────────────────────┐
│                        GAME MODULE                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   Standalone Mode                    │   Integrated Mode             │
│   (game.gkebook.vn)                  │   (school.gkebook.vn/game)    │
│                                      │                               │
│   • Ai cũng tạo được                 │   • Chỉ giáo viên tạo        │
│   • Guest players (tên + avatar)     │   • Học sinh đăng nhập       │
│   • Không lưu history                │   • Lưu vào mission results  │
│   • Không tính điểm chính thức       │   • Tính điểm vào học        │
│                                      │                               │
└──────────────────────────────────────────────────────────────────────┘
```

### 6.2 API modes

```go
// Standalone: Không cần auth
POST /api/game/rooms          // Tạo phòng (guest host)
POST /api/game/rooms/:code/join  // Vào phòng (guest player)

// Integrated: Cần auth JWT
POST /api/game/rooms?integrated=true  // Tạo phòng (teacher only)
POST /api/game/rooms/:code/join?user_id=xxx  // Vào với account
```

---

## VII. TIMELINE PROPOSED

| Tuần | Công việc | Deliverable |
|------|-----------|-------------|
| **Tuần 1** | Setup module structure + DB | `/modules/game/`, migrations |
| **Tuần 2** | Core WebSocket + Room management | Tạo/Join phòng hoạt động |
| **Tuần 3** | Game loop + Scoring | Chơi được, tính điểm |
| **Tuần 4** | Host UI + Player UI | Giao diện hoàn chỉnh |
| **Tuần 5** | Testing + Edge cases | Load test 500 players |
| **Tuần 6** | Integration với school-frontend | Giáo viên tạo game |

---

## VIII. CÂU HỎI CẦN QUYẾT ĐỊNH

1. **Frontend riêng hay chung?**
   - [ ] Tạo `prj-gkebook-game-frontend` mới
   - [ ] Thêm vào `prj-gkebook-shadcn-frontend1`

2. **Domain cho game?**
   - [ ] game.gkebook.vn (subdomain riêng)
   - [ ] gkebook.vn/game (cùng domain)

3. **Giao thức chính?**
   - [ ] WebSocket (đơn giản, browser native)
   - [ ] MQTT over WebSocket (scale tốt hơn, phức tạp hơn)

4. **Lưu quiz data thế nào?**
   - [ ] Copy vào `quiz_data JSONB` (độc lập hoàn toàn)
   - [ ] Reference sang bảng `exams` (phụ thuộc module quiz)

---

*Đây là Version 2 - cần review và quyết định các câu hỏi trước khi implement.*
