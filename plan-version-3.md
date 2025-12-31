# Plan Game Module - Version 3 (Mixed Player Modes)

## Thay đổi so với Version 2

**Cập nhật:** Người dùng đã đăng nhập (có JWT) có thể:
1. Chơi với account của mình (dùng thông tin đã đăng nhập)
2. Chơi như guest (nhập tên + chọn avatar mới)

Cả hai loại có thể chơi chung trong cùng một phòng.

---

## I. PLAYER MODES (Cập nhật)

### 1.1 Ba loại người chơi trong một phòng

| Loại | Mô tả | Identifier |
|------|-------|------------|
| **Guest** | Chưa đăng nhập, nhập tên + avatar | `guest_<random_id>` |
| **Logged-in (Account)** | Đã đăng nhập, dùng thông tin account | `user_<user_id>` |
| **Logged-in (As Guest)** | Đã đăng nhập nhưng chọn chơi ẩn danh | `anon_<user_id>` |

### 1.2 Luồng Join với tùy chọn

```
┌─────────────────────────────────────────────────────────────┐
│ 1. NHẬP MÃ PHÒNG (Giống nhau cho tất cả)                    │
│    Mã phòng: [ 123456 ]                                     │
│    [  VÀO PHÒNG  ]                                          │
└─────────────────────────────────────────────────────────────┘
                         ↓
                    ┌────────────┐
                    │ Check JWT  │
                    └────────────┘
                    ↓            ↓
           Có JWT (đã login)    Không có JWT
                    ↓                ↓
┌───────────────────────────┐  ┌───────────────────────────┐
│ 2a. CHỌN CÁCH CHƠI        │  │ 2b. NHẬP THÔNG TIN GUEST  │
│ ┌─────────────────────┐   │  │ ┌─────────────────────┐   │
│ │ 👤 Nguyễn Văn A     │   │  │ │ Tên: [ __________ ] │   │
│ │ ┌───┐               │   │  │ │                     │   │
│ │ │ 📷│ avatar user   │   │  │ │ Avatar:             │   │
│ │ └───┘               │   │  │ │ 🐱 🐶 🦊 🐼 🐨     │   │
│ │                     │   │  │ └─────────────────────┘   │
│ │ [CHƠI VỚI TÀI KHOẢN]│   │  │                           │
│ └─────────────────────┘   │  │ [  THAM GIA  ]            │
│                            │  └───────────────────────────┘
│ ────── HOẶC ──────        │
│                            │
│ ┌─────────────────────┐   │
│ │ 🎭 CHƠI ẨN DANH     │   │
│ │ Tên: [ __________ ] │   │
│ │ Avatar:             │   │
│ │ 🐱 🐶 🦊 🐼 🐨     │   │
│ │                     │   │
│ │ [CHƠI KHÔNG CẦN    │   │
│ │  HIỆN TÀI KHOẢN]   │   │
│ └─────────────────────┘   │
└───────────────────────────┘
```

### 1.3 Hiển thị trong Waiting Room

```
┌─────────────────────────────────────────────────────────────┐
│  PHÒNG 123456 - Người chơi (6/100):                         │
│                                                              │
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐                        │
│  │ 📷│ │ 📷│ │🐱│ │🦊│ │🐼│ │🐶│                             │
│  │    │ │    │ │    │ │    │ │    │ │    │                   │
│  │Nam │ │An  │ │Hùng│ │Linh│ │Khoa│ │Minh│                   │
│  │ ✓  │ │ ✓  │ │    │ │    │ │    │ │    │                   │
│  └───┘ └───┘ └───┘ └───┘ └───┘ └───┘                        │
│    ↑      ↑      ↑      ↑      ↑      ↑                     │
│  User   User  Guest  Guest  Guest  Anon                     │
│(account)(account)           (logged-in nhưng chọn ẩn danh)  │
│                                                              │
│  ✓ = Đã có tài khoản (hiện badge nhỏ)                       │
└─────────────────────────────────────────────────────────────┘
```

---

## II. IMPLEMENTATION DETAILS

### 2.1 Join Room API

```go
// POST /api/game/rooms/:code/join
type JoinRoomRequest struct {
    // Nếu có JWT trong header → có thể bỏ qua các field này
    
    // Option 1: Chơi với account (chỉ cần gửi flag)
    UseAccount bool `json:"use_account,omitempty"`
    
    // Option 2: Chơi như guest (bắt buộc nếu không có JWT hoặc chọn ẩn danh)
    PlayerName string `json:"player_name,omitempty"`
    Avatar     string `json:"avatar,omitempty"`
}

// Response
type JoinRoomResponse struct {
    PlayerId   string `json:"player_id"`     // guest_xxx, user_xxx, hoặc anon_xxx
    PlayerName string `json:"player_name"`   // Tên hiển thị
    Avatar     string `json:"avatar"`        // Avatar code hoặc URL
    PlayerType string `json:"player_type"`   // "guest", "account", "anonymous"
}
```

### 2.2 Backend Logic

```go
func JoinRoom(ctx context.Context, req JoinRoomRequest) (*JoinRoomResponse, error) {
    room := GetRoom(req.RoomCode)
    
    // Check room capacity
    if len(room.Players) >= room.Config.MaxPlayers {
        return nil, errors.New("ROOM_FULL")
    }
    
    // Determine player type
    jwt := GetJWTFromContext(ctx)
    
    var player GamePlayer
    
    if jwt != nil && req.UseAccount {
        // Option 1: Logged-in user using their account
        user := GetUserFromJWT(jwt)
        player = GamePlayer{
            ID:         "user_" + user.ID,
            UserID:     &user.ID,  // Link to user account
            Name:       user.FullName,
            Avatar:     user.AvatarURL,
            PlayerType: "account",
        }
    } else if jwt != nil && !req.UseAccount {
        // Option 2: Logged-in user playing anonymously
        player = GamePlayer{
            ID:         "anon_" + user.ID,
            UserID:     &user.ID,  // Still track for analytics (optional)
            Name:       req.PlayerName,
            Avatar:     req.Avatar,
            PlayerType: "anonymous",
        }
    } else {
        // Option 3: Guest (no JWT)
        player = GamePlayer{
            ID:         "guest_" + uuid.New().String()[:8],
            UserID:     nil,
            Name:       req.PlayerName,
            Avatar:     req.Avatar,
            PlayerType: "guest",
        }
    }
    
    room.AddPlayer(player)
    return &JoinRoomResponse{...}, nil
}
```

### 2.3 Database Schema Update

```sql
-- Cập nhật bảng game_players
CREATE TABLE game_players (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id UUID NOT NULL REFERENCES game_rooms(id) ON DELETE CASCADE,
    
    -- Player identification
    player_id VARCHAR(50) NOT NULL,       -- "guest_xxx", "user_xxx", "anon_xxx"
    user_id UUID REFERENCES user_infos(id), -- NULL nếu pure guest
    
    -- Display info
    player_name VARCHAR(100) NOT NULL,
    avatar VARCHAR(255) NOT NULL,         -- Emoji code hoặc URL
    
    -- Type: "guest", "account", "anonymous"
    player_type VARCHAR(20) NOT NULL DEFAULT 'guest',
    
    -- Results
    total_score DECIMAL(10,2) DEFAULT 0,
    correct_count INT DEFAULT 0,
    wrong_count INT DEFAULT 0,
    final_rank INT,
    
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(room_id, player_id)
);

-- Index cho query theo user_id (để xem lịch sử)
CREATE INDEX idx_game_players_user ON game_players(user_id) WHERE user_id IS NOT NULL;
```

---

## III. FRONTEND FLOW

### 3.1 Check trạng thái đăng nhập

```typescript
// Khi vào trang Join
const handleJoinPage = async () => {
    const jwt = getJWTFromStorage();
    
    if (jwt) {
        // Đã đăng nhập → hiện 2 options
        setShowAccountOption(true);
        setUserInfo(await getUserInfo(jwt));
    } else {
        // Chưa đăng nhập → chỉ hiện form guest
        setShowAccountOption(false);
    }
};
```

### 3.2 UI Components

```tsx
// JoinRoomPage.tsx
function JoinRoomPage() {
    const { isLoggedIn, userInfo } = useAuth();
    
    return (
        <div>
            <h1>Tham gia phòng {roomCode}</h1>
            
            {isLoggedIn ? (
                <div className="join-options">
                    {/* Option 1: Dùng account */}
                    <div className="option account-option">
                        <img src={userInfo.avatar} alt="avatar" />
                        <span>{userInfo.fullName}</span>
                        <button onClick={() => joinWithAccount()}>
                            Chơi với tài khoản
                        </button>
                    </div>
                    
                    <div className="divider">HOẶC</div>
                    
                    {/* Option 2: Ẩn danh */}
                    <div className="option anonymous-option">
                        <h3>🎭 Chơi ẩn danh</h3>
                        <input 
                            placeholder="Nhập tên" 
                            value={guestName}
                            onChange={e => setGuestName(e.target.value)}
                        />
                        <AvatarPicker 
                            selected={avatar}
                            onSelect={setAvatar}
                        />
                        <button onClick={() => joinAsAnonymous()}>
                            Tham gia
                        </button>
                    </div>
                </div>
            ) : (
                /* Guest chỉ thấy form nhập tên + avatar */
                <GuestJoinForm onJoin={joinAsGuest} />
            )}
        </div>
    );
}
```

---

## IV. LƯU KẾT QUẢ & HISTORY

### 4.1 Quản lý lịch sử theo loại người chơi

| Player Type | Lưu vào account? | Hiện trong History? |
|-------------|------------------|---------------------|
| **account** | ✅ Có | ✅ Có (xem được trong profile) |
| **anonymous** | ❌ Không lưu vào profile | ❌ Không hiện |
| **guest** | ❌ Không có account | ❌ Không thể |

### 4.2 Thống kê (cho giáo viên trong integrated mode)

```sql
-- Lấy kết quả chỉ từ account players
SELECT p.*, u.fullname, u.student_id
FROM game_players p
LEFT JOIN user_infos u ON p.user_id = u.id
WHERE p.room_id = 'xxx'
  AND p.player_type = 'account'  -- Chỉ lấy ai chơi bằng account
ORDER BY p.final_rank;
```

---

## V. TÓM TẮT THAY ĐỔI V3

| Aspect | V2 | V3 |
|--------|----|----|
| Logged-in user options | Chỉ chơi với account | Có thể chọn account HOẶC ẩn danh |
| Mixed room | Không đề cập | ✅ Guest + Account + Anonymous chơi chung |
| Player tracking | user_id hoặc null | player_id prefix (guest_, user_, anon_) |
| History | Saved nếu có account | Saved CHỈ nếu chọn "account" mode |

---

## VI. CÂU HỎI TIẾP THEO

1. **Khi chơi ẩn danh, có lưu `user_id` để tracking không?**
   - [ ] Có, để biết ai chơi (nhưng không hiện public)
   - [ ] Không, hoàn toàn ẩn danh như guest

2. **Badge verified cho account player?**
   - [ ] Hiện ✓ badge nhỏ
   - [ ] Không phân biệt visually

3. **Cho phép switch giữa game?**
   - [ ] Không, chọn 1 lần khi join
   - [ ] Có thể đổi ở màn waiting

---

*Version 3 - Cập nhật mixed player modes theo yêu cầu.*
