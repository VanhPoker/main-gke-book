# Plan Game Module - Version 7 (Pure WebSocket Architecture)

## Tổng quan

Sử dụng **WebSocket thuần** (gorilla/websocket) với Redis Pub/Sub để scale.
Bao gồm tất cả edge cases và best practices từ các version trước.

---

## I. KIẾN TRÚC TỔNG THỂ

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    GAME MODULE - WEBSOCKET ARCHITECTURE                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                        │
│   │   Browser   │   │   Browser   │   │   Mobile    │                        │
│   │   (Host)    │   │  (Player)   │   │   (Future)  │                        │
│   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘                        │
│          │                 │                 │                                │
│          │         WebSocket (wss://)        │                                │
│          │                 │                 │                                │
│          └────────────────┬────────────────┘                                 │
│                           ↓                                                   │
│   ┌───────────────────────────────────────────────────────────────────────┐  │
│   │                        GAME SERVER (Go)                                │  │
│   │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│   │  │                    WebSocket Hub                                 │  │  │
│   │  │  • Connection management                                        │  │  │
│   │  │  • Room routing                                                 │  │  │
│   │  │  • Broadcast (parallel goroutines)                              │  │  │
│   │  └─────────────────────────────────────────────────────────────────┘  │  │
│   │                               │                                        │  │
│   │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                  │  │
│   │  │   Answer    │   │    Game     │   │   Player    │                  │  │
│   │  │   Queue     │   │   Logic     │   │   Manager   │                  │  │
│   │  │ (Channel)   │   │  (Scoring)  │   │             │                  │  │
│   │  └─────────────┘   └─────────────┘   └─────────────┘                  │  │
│   └───────────────────────────────────────────────────────────────────────┘  │
│                               │                                               │
│   ┌───────────────────────────┼───────────────────────────────────────────┐  │
│   │                     DATA LAYER                                         │  │
│   │         ┌─────────────────┼─────────────────┐                          │  │
│   │         │                 │                 │                          │  │
│   │  ┌──────┴─────┐    ┌──────┴─────┐    ┌──────┴─────┐                   │  │
│   │  │ PostgreSQL │    │   Redis    │    │    S3      │                   │  │
│   │  │            │    │            │    │            │                   │  │
│   │  │• Users     │    │• Sessions  │    │• Quiz files│                   │  │
│   │  │• Results   │    │• Room state│    │• Media     │                   │  │
│   │  │• History   │    │• Pub/Sub   │    │            │                   │  │
│   │  └────────────┘    └────────────┘    └────────────┘                   │  │
│   └───────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## II. WEBSOCKET HUB DESIGN

### 2.1 Core Struct

```go
type GameHub struct {
    // Room management
    rooms       map[string]*Room          // roomCode -> Room
    roomsMu     sync.RWMutex
    
    // Connection management  
    clients     map[*Client]bool          // All connected clients
    clientsMu   sync.RWMutex
    
    // Channels for async processing
    register    chan *Client
    unregister  chan *Client
    broadcast   chan BroadcastMsg
    answerQueue chan AnswerSubmit         // Queue for concurrent answers
    
    // Workers
    workerCount int                       // Answer processing workers
}

type Room struct {
    Code        string
    HostId      string
    Players     map[string]*Player
    PlayersMu   sync.RWMutex
    
    // Game state
    Status      string                    // waiting, playing, ended
    Questions   []Question
    CurrentIdx  int
    StartTime   time.Time
    
    // Submitted answers tracking (prevent duplicate)
    Submitted   map[string]bool           // "playerId:questionId" -> true
    SubmittedMu sync.RWMutex
}

type Client struct {
    Conn       *websocket.Conn
    PlayerId   string
    PlayerType string                     // guest, account, anonymous
    RoomCode   string
    Send       chan []byte                // Outgoing messages
}
```

### 2.2 Message Types

```go
// Incoming messages
const (
    MSG_CREATE_ROOM    = "CREATE_ROOM"
    MSG_JOIN_ROOM      = "JOIN_ROOM"
    MSG_LEAVE_ROOM     = "LEAVE_ROOM"
    MSG_START_GAME     = "START_GAME"
    MSG_SUBMIT_ANSWER  = "SUBMIT_ANSWER"
    MSG_NEXT_QUESTION  = "NEXT_QUESTION"
    MSG_END_GAME       = "END_GAME"
    MSG_HEARTBEAT      = "HEARTBEAT"
)

// Outgoing messages
const (
    MSG_ROOM_CREATED       = "ROOM_CREATED"
    MSG_PLAYER_JOINED      = "PLAYER_JOINED"
    MSG_PLAYER_LEFT        = "PLAYER_LEFT"
    MSG_GAME_STARTED       = "GAME_STARTED"
    MSG_QUESTION           = "QUESTION"
    MSG_ANSWER_RESULT      = "ANSWER_RESULT"
    MSG_LEADERBOARD_UPDATE = "LEADERBOARD_UPDATE"
    MSG_GAME_ENDED         = "GAME_ENDED"
    MSG_ERROR              = "ERROR"
)
```

---

## III. XỬ LÝ CONCURRENT SUBMIT

### 3.1 Answer Queue với Worker Pool

```go
func (h *GameHub) StartAnswerWorkers(count int) {
    for i := 0; i < count; i++ {
        go h.answerWorker(i)
    }
}

func (h *GameHub) answerWorker(id int) {
    for answer := range h.answerQueue {
        h.processAnswer(answer)
    }
}

func (h *GameHub) handleSubmitAnswer(client *Client, data []byte) {
    var req AnswerSubmit
    json.Unmarshal(data, &req)
    
    // Ghi ServerTime NGAY khi nhận được
    req.ServerTime = time.Now().UnixNano()
    req.PlayerId = client.PlayerId
    req.RoomCode = client.RoomCode
    
    // Đẩy vào queue, không block
    select {
    case h.answerQueue <- req:
        // OK
    default:
        // Queue full - xử lý sync (fallback)
        h.processAnswer(req)
    }
}
```

### 3.2 Process Answer với Duplicate Check

```go
func (h *GameHub) processAnswer(answer AnswerSubmit) {
    room := h.getRoom(answer.RoomCode)
    if room == nil {
        return
    }
    
    // Check duplicate
    key := fmt.Sprintf("%s:%s", answer.PlayerId, answer.QuestionId)
    room.SubmittedMu.Lock()
    if room.Submitted[key] {
        room.SubmittedMu.Unlock()
        return // Đã submit rồi, bỏ qua
    }
    room.Submitted[key] = true
    room.SubmittedMu.Unlock()
    
    // Check timeout
    question := room.Questions[room.CurrentIdx]
    responseTime := answer.ServerTime - question.StartTime
    if responseTime > question.TimeLimit.Nanoseconds() {
        // Quá giờ
        h.sendToClient(answer.PlayerId, MSG_ANSWER_RESULT, AnswerResult{
            IsCorrect: false,
            Points:    0,
            Message:   "Hết giờ!",
        })
        return
    }
    
    // Validate answer
    isCorrect := question.CorrectChoiceId == answer.ChoiceId
    
    // Calculate points (nhanh hơn = điểm cao hơn)
    points := 0.0
    if isCorrect {
        timeRatio := float64(responseTime) / float64(question.TimeLimit.Nanoseconds())
        points = float64(question.MaxPoints) * (1 - timeRatio*0.5) // Min 50% points
    }
    
    // Update player score
    room.updatePlayerScore(answer.PlayerId, points, isCorrect)
    
    // Send result to player
    h.sendToClient(answer.PlayerId, MSG_ANSWER_RESULT, AnswerResult{
        IsCorrect:    isCorrect,
        Points:       points,
        ResponseTime: responseTime / 1e6, // ms
    })
    
    // Broadcast leaderboard update
    h.broadcastLeaderboard(room)
}
```

---

## IV. PARALLEL BROADCAST

### 4.1 Broadcast với Goroutines

```go
func (h *GameHub) broadcastToRoom(roomCode string, msgType string, data interface{}) {
    room := h.getRoom(roomCode)
    if room == nil {
        return
    }
    
    payload, _ := json.Marshal(Message{Type: msgType, Data: data})
    
    room.PlayersMu.RLock()
    players := make([]*Player, 0, len(room.Players))
    for _, p := range room.Players {
        players = append(players, p)
    }
    room.PlayersMu.RUnlock()
    
    // Parallel send với semaphore
    sem := make(chan struct{}, 100)  // Max 100 concurrent writes
    var wg sync.WaitGroup
    
    for _, player := range players {
        if player.Client == nil {
            continue
        }
        
        wg.Add(1)
        sem <- struct{}{}
        
        go func(c *Client) {
            defer wg.Done()
            defer func() { <-sem }()
            
            select {
            case c.Send <- payload:
                // OK
            default:
                // Client buffer full, close connection
                close(c.Send)
            }
        }(player.Client)
    }
    
    wg.Wait()
}
```

### 4.2 Client Write Pump

```go
func (c *Client) writePump() {
    ticker := time.NewTicker(30 * time.Second)  // Heartbeat
    defer ticker.Stop()
    
    for {
        select {
        case msg, ok := <-c.Send:
            if !ok {
                c.Conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            
            c.Conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if err := c.Conn.WriteMessage(websocket.TextMessage, msg); err != nil {
                return
            }
            
        case <-ticker.C:
            // Send ping
            c.Conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if err := c.Conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

---

## V. EDGE CASES HANDLING

### 5.1 Duplicate Submit

```go
// Trong Room struct
Submitted map[string]bool  // "playerId:questionId" -> true

// Khi process answer
key := playerId + ":" + questionId
if room.Submitted[key] {
    return  // Bỏ qua
}
room.Submitted[key] = true
```

### 5.2 Late Submit (Quá giờ)

```go
responseTime := answer.ServerTime - question.StartTime
if responseTime > question.TimeLimit.Nanoseconds() {
    // Từ chối, không tính điểm
    sendError(client, "TIME_EXPIRED", "Hết giờ rồi!")
    return
}
```

### 5.3 Disconnect & Reconnect

```go
// Khi client disconnect
func (h *GameHub) handleDisconnect(client *Client) {
    room := h.getRoom(client.RoomCode)
    if room != nil {
        // Đánh dấu offline, KHÔNG xóa khỏi room
        player := room.Players[client.PlayerId]
        if player != nil {
            player.Status = "offline"
            player.DisconnectTime = time.Now()
        }
        
        // Broadcast player offline
        h.broadcastToRoom(room.Code, MSG_PLAYER_LEFT, PlayerEvent{
            PlayerId: client.PlayerId,
            Name:     player.Name,
            Status:   "offline",
        })
    }
}

// Khi client reconnect
func (h *GameHub) handleReconnect(client *Client, roomCode, playerId string) {
    room := h.getRoom(roomCode)
    if room == nil {
        sendError(client, "ROOM_NOT_FOUND", "Phòng không tồn tại")
        return
    }
    
    player := room.Players[playerId]
    if player == nil {
        sendError(client, "PLAYER_NOT_FOUND", "Không tìm thấy player")
        return
    }
    
    // Restore connection
    player.Client = client
    player.Status = "online"
    client.RoomCode = roomCode
    client.PlayerId = playerId
    
    // Send current game state
    h.sendToClient(playerId, "GAME_STATE", GameState{
        RoomCode:        room.Code,
        Status:          room.Status,
        CurrentQuestion: room.CurrentIdx,
        YourScore:       player.Score,
        Leaderboard:     room.getLeaderboard(),
    })
    
    // Broadcast player online
    h.broadcastToRoom(room.Code, MSG_PLAYER_JOINED, PlayerEvent{
        PlayerId: playerId,
        Name:     player.Name,
        Status:   "online",
    })
}
```

### 5.4 Room Full

```go
func (h *GameHub) handleJoinRoom(client *Client, data []byte) {
    var req JoinRoomRequest
    json.Unmarshal(data, &req)
    
    room := h.getRoom(req.RoomCode)
    if room == nil {
        sendError(client, "ROOM_NOT_FOUND", "Phòng không tồn tại")
        return
    }
    
    if len(room.Players) >= room.MaxPlayers {
        sendError(client, "ROOM_FULL", "Phòng đã đầy")
        return
    }
    
    // Add player...
}
```

### 5.5 Host Disconnect giữa game

```go
// Khi host disconnect
if player.IsHost {
    // Option 1: Pause game
    room.Status = "paused"
    h.broadcastToRoom(room.Code, "GAME_PAUSED", "Host đã mất kết nối")
    
    // Option 2: Auto-assign new host (player có điểm cao nhất)
    // newHost := room.getHighestScorePlayer()
    // room.HostId = newHost.Id
    
    // Set timeout - nếu host không reconnect trong 2 phút, end game
    go func() {
        time.Sleep(2 * time.Minute)
        if room.Status == "paused" {
            h.endGame(room.Code)
        }
    }()
}
```

### 5.6 Server Restart Recovery

```go
// Lưu game state vào Redis định kỳ
func (h *GameHub) saveRoomState(room *Room) {
    state := RoomState{
        Code:        room.Code,
        HostId:      room.HostId,
        Players:     room.Players,
        Status:      room.Status,
        CurrentIdx:  room.CurrentIdx,
        Submitted:   room.Submitted,
    }
    
    data, _ := json.Marshal(state)
    redis.Set("room:"+room.Code, data, 1*time.Hour)
}

// Khi server restart, load từ Redis
func (h *GameHub) recoverRooms() {
    keys := redis.Keys("room:*")
    for _, key := range keys {
        data := redis.Get(key)
        var state RoomState
        json.Unmarshal(data, &state)
        
        room := h.createRoomFromState(state)
        h.rooms[room.Code] = room
    }
}
```

---

## VI. SCALING VỚI REDIS PUB/SUB

Khi cần nhiều server:

```go
// Subscribe to Redis channel
func (h *GameHub) subscribeRedis() {
    pubsub := redisClient.Subscribe("game:broadcast")
    
    for msg := range pubsub.Channel() {
        var broadcast BroadcastMsg
        json.Unmarshal([]byte(msg.Payload), &broadcast)
        
        // Broadcast to local clients in this room
        h.broadcastToLocalRoom(broadcast.RoomCode, broadcast.Type, broadcast.Data)
    }
}

// Publish qua Redis (khi có nhiều server)
func (h *GameHub) publishToRedis(roomCode, msgType string, data interface{}) {
    msg := BroadcastMsg{
        RoomCode: roomCode,
        Type:     msgType,
        Data:     data,
    }
    payload, _ := json.Marshal(msg)
    redisClient.Publish("game:broadcast", payload)
}
```

---

## VII. DATABASE SCHEMA

```sql
-- Guest identity
CREATE TABLE game_guests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_token VARCHAR(64) UNIQUE NOT NULL,
    guest_name VARCHAR(100) NOT NULL,
    guest_avatar VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    last_played_at TIMESTAMP DEFAULT NOW()
);

-- Game rooms (lưu kết quả sau khi game end)
CREATE TABLE game_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    room_code VARCHAR(10) NOT NULL,
    quiz_data JSONB NOT NULL,
    host_user_id UUID REFERENCES user_infos(id),
    host_name VARCHAR(100),
    status VARCHAR(20) DEFAULT 'waiting',
    total_players INT DEFAULT 0,
    started_at TIMESTAMP,
    ended_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Player results
CREATE TABLE game_player_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    game_session_id UUID REFERENCES game_sessions(id) ON DELETE CASCADE,
    player_id VARCHAR(50) NOT NULL,  -- guest_xxx or user_xxx
    user_id UUID REFERENCES user_infos(id),
    player_name VARCHAR(100) NOT NULL,
    player_type VARCHAR(20) NOT NULL,  -- guest, account, anonymous
    total_score DECIMAL(10,2) DEFAULT 0,
    correct_count INT DEFAULT 0,
    final_rank INT
);
```

---

## VIII. FRONTEND INTEGRATION

```typescript
// useGameWebSocket.ts
class GameWebSocket {
    private ws: WebSocket | null = null
    private reconnectAttempts = 0
    private maxReconnects = 5
    
    connect(roomCode: string) {
        this.ws = new WebSocket(`wss://api.gkebook.vn/ws/game/${roomCode}`)
        
        this.ws.onopen = () => {
            this.reconnectAttempts = 0
            console.log('Connected')
        }
        
        this.ws.onclose = () => {
            this.handleReconnect(roomCode)
        }
        
        this.ws.onmessage = (event) => {
            const msg = JSON.parse(event.data)
            this.handleMessage(msg)
        }
    }
    
    private handleReconnect(roomCode: string) {
        if (this.reconnectAttempts < this.maxReconnects) {
            const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000)
            setTimeout(() => {
                this.reconnectAttempts++
                this.connect(roomCode)
            }, delay)
        }
    }
    
    send(type: string, data: any) {
        if (this.ws?.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify({ type, data }))
        }
    }
    
    submitAnswer(questionId: string, choiceId: string) {
        this.send('SUBMIT_ANSWER', { question_id: questionId, choice_id: choiceId })
    }
}
```

---

## IX. CAPACITY & TIMELINE

### Capacity

| Component | Single Server | With Redis Scaling |
|-----------|---------------|-------------------|
| WebSocket connections | ~50,000 | ~50k × N servers |
| Rooms concurrent | ~500 | ~500 × N |
| Answer processing | ~10,000/s | Scales with workers |

### Timeline

| Tuần | Task |
|------|------|
| 1 | WebSocket Hub + Room management |
| 2 | Answer Queue + Scoring |
| 3 | Edge cases (duplicate, timeout, reconnect) |
| 4 | Frontend integration |
| 5 | Redis Pub/Sub (optional for scaling) |
| 6 | Testing + Deploy |

---

## X. TÓM TẮT

| Feature | Implementation |
|---------|---------------|
| **Protocol** | WebSocket thuần (gorilla/websocket) |
| **Concurrent submit** | Channel queue + Worker pool |
| **Duplicate prevention** | Map[playerId:questionId] |
| **Late submit** | ServerTime validation |
| **Reconnect** | Session recovery từ Redis |
| **Parallel broadcast** | Goroutines + Semaphore |
| **Scaling** | Redis Pub/Sub khi cần |

---

*Version 7 - Pure WebSocket Architecture với đầy đủ edge cases*
