# Plan Game Module - Version 6 (MQTT over WebSocket Architecture)

## Tổng quan kiến trúc

Sử dụng **MQTT over WebSocket** làm giao thức realtime chính, với EMQX làm broker.

---

## I. KIẾN TRÚC TỔNG THỂ

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           GAME MODULE ARCHITECTURE                            │
│                          (MQTT over WebSocket)                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                        │
│   │   Browser   │   │   Browser   │   │   Mobile    │                        │
│   │   (Host)    │   │  (Player)   │   │   (Future)  │                        │
│   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘                        │
│          │                 │                 │                                │
│          │    MQTT over WebSocket (wss://broker:8084)                        │
│          │                 │                 │                                │
│          └────────────────┼────────────────┘                                 │
│                           ↓                                                   │
│   ┌───────────────────────────────────────────────────────────────────────┐  │
│   │                     EMQX BROKER (Single Node)                          │  │
│   │                                                                        │  │
│   │                      ┌─────────────────┐                               │  │
│   │                      │     EMQX        │                               │  │
│   │                      │   (500k conn)   │                               │  │
│   │                      └────────┬────────┘                               │  │
│   │                               │                                        │  │
│   │  Features:                    │                                        │  │
│   │  • Pub/Sub routing            │                                        │  │
│   │  • QoS (0, 1, 2)              │                                        │  │
│   │  • Retained messages          │                                        │  │
│   │  • Will messages              │                                        │  │
│   │  • Dashboard monitoring       │                                        │  │
│   └───────────────────────────────┼───────────────────────────────────────┘  │
│                               │                                               │
│                    MQTT (tcp://broker:1883)                                  │
│                               │                                               │
│   ┌───────────────────────────┼───────────────────────────────────────────┐  │
│   │                     GAME SERVER CLUSTER                                │  │
│   │                               │                                        │  │
│   │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                  │  │
│   │  │  Server 1   │   │  Server 2   │   │  Server 3   │                  │  │
│   │  │             │   │             │   │             │                  │  │
│   │  │ • Game logic│   │ • Game logic│   │ • Game logic│                  │  │
│   │  │ • Scoring   │   │ • Scoring   │   │ • Scoring   │                  │  │
│   │  │ • Validation│   │ • Validation│   │ • Validation│                  │  │
│   │  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘                  │  │
│   │         └─────────────────┼─────────────────┘                          │  │
│   └───────────────────────────┼───────────────────────────────────────────┘  │
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
│   │  │• History   │    │• Leaderboard│   │            │                   │  │
│   │  └────────────┘    └────────────┘    └────────────┘                   │  │
│   └───────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## II. MQTT TOPIC STRUCTURE

### 2.1 Topic Hierarchy

```
game/
├── rooms/
│   └── {roomCode}/
│       ├── status          # Room status (waiting/playing/ended)
│       ├── config          # Room configuration
│       ├── players/
│       │   ├── join        # Player join events
│       │   ├── leave       # Player leave events
│       │   └── list        # Current player list (retained)
│       ├── questions/
│       │   ├── current     # Current question (retained)
│       │   ├── timer       # Timer updates
│       │   └── result      # Question result after timeout
│       ├── answers/
│       │   └── submit      # Player answer submissions
│       └── leaderboard     # Live leaderboard updates
│
├── system/
│   ├── health              # System health status
│   └── broadcast           # System-wide announcements
│
└── users/
    └── {playerId}/
        └── private         # Private messages to specific player
```

### 2.2 Message Examples

```json
// Topic: game/rooms/123456/questions/current
{
    "question_id": "q1",
    "content": "1 + 1 = ?",
    "choices": [
        {"id": "a", "content": "1"},
        {"id": "b", "content": "2"},
        {"id": "c", "content": "3"}
    ],
    "time_limit": 30,
    "question_number": 1,
    "total_questions": 10,
    "started_at": "2024-12-31T15:00:00Z"
}

// Topic: game/rooms/123456/answers/submit
{
    "player_id": "guest_xyz789",
    "question_id": "q1",
    "choice_id": "b",
    "submitted_at": "2024-12-31T15:00:05.123Z"
}

// Topic: game/rooms/123456/leaderboard
{
    "updated_at": "2024-12-31T15:00:06Z",
    "rankings": [
        {"rank": 1, "player_id": "guest_xyz", "name": "Nam", "avatar": "🐱", "score": 850},
        {"rank": 2, "player_id": "user_abc", "name": "An", "avatar": "📷", "score": 720}
    ]
}
```

### 2.3 QoS Levels

| Topic | QoS | Lý do |
|-------|-----|-------|
| `.../questions/current` | 1 | Đảm bảo nhận được câu hỏi |
| `.../answers/submit` | 1 | Đảm bảo answer được gửi |
| `.../leaderboard` | 0 | OK nếu miss 1-2 updates |
| `.../players/list` | 1 + Retained | Luôn có khi join room |
| `.../timer` | 0 | Realtime, miss ok |

---

## III. COMPONENT DETAILS

### 3.1 EMQX Broker Setup

```yaml
# docker-compose.yml
version: '3.8'
services:
  emqx:
    image: emqx/emqx:5.4.0
    container_name: emqx
    ports:
      - "1883:1883"     # MQTT TCP
      - "8083:8083"     # MQTT WebSocket
      - "8084:8084"     # MQTT WebSocket SSL
      - "8883:8883"     # MQTT TCP SSL
      - "18083:18083"   # Dashboard
    environment:
      - EMQX_NAME=emqx
      - EMQX_HOST=127.0.0.1
      - EMQX_DASHBOARD__DEFAULT_PASSWORD=admin123
    volumes:
      - emqx-data:/opt/emqx/data
      - emqx-log:/opt/emqx/log
      - ./emqx.conf:/opt/emqx/etc/emqx.conf
    
volumes:
  emqx-data:
  emqx-log:
```

### 3.2 Frontend (MQTT.js)

```typescript
// lib/mqtt.ts
import mqtt, { MqttClient } from 'mqtt'

class GameMQTT {
    private client: MqttClient | null = null
    private roomCode: string = ''
    
    connect(playerId: string) {
        this.client = mqtt.connect('wss://broker.gkebook.vn:8084/mqtt', {
            clientId: `game_${playerId}_${Date.now()}`,
            clean: true,
            reconnectPeriod: 1000,
            connectTimeout: 30000,
            // Will message - báo cho room khi disconnect
            will: {
                topic: `game/rooms/${this.roomCode}/players/leave`,
                payload: JSON.stringify({ player_id: playerId }),
                qos: 1,
                retain: false
            }
        })
        
        this.client.on('connect', () => {
            console.log('Connected to MQTT broker')
        })
        
        this.client.on('error', (err) => {
            console.error('MQTT error:', err)
        })
        
        return this.client
    }
    
    joinRoom(roomCode: string) {
        this.roomCode = roomCode
        
        // Subscribe to room topics
        this.client?.subscribe([
            `game/rooms/${roomCode}/status`,
            `game/rooms/${roomCode}/questions/#`,
            `game/rooms/${roomCode}/leaderboard`,
            `game/rooms/${roomCode}/players/#`
        ], { qos: 1 })
    }
    
    submitAnswer(questionId: string, choiceId: string, playerId: string) {
        this.client?.publish(
            `game/rooms/${this.roomCode}/answers/submit`,
            JSON.stringify({
                player_id: playerId,
                question_id: questionId,
                choice_id: choiceId,
                submitted_at: new Date().toISOString()
            }),
            { qos: 1 }
        )
    }
    
    onQuestion(callback: (question: Question) => void) {
        this.client?.on('message', (topic, message) => {
            if (topic.endsWith('/questions/current')) {
                callback(JSON.parse(message.toString()))
            }
        })
    }
    
    onLeaderboard(callback: (leaderboard: Leaderboard) => void) {
        this.client?.on('message', (topic, message) => {
            if (topic.endsWith('/leaderboard')) {
                callback(JSON.parse(message.toString()))
            }
        })
    }
    
    disconnect() {
        this.client?.end()
    }
}

export const gameMQTT = new GameMQTT()
```

### 3.3 Backend (Go MQTT Client)

```go
// internal/mqtt/client.go
package mqtt

import (
    "encoding/json"
    "fmt"
    mqtt "github.com/eclipse/paho.mqtt.golang"
)

type GameMQTTClient struct {
    client mqtt.Client
}

func NewGameMQTTClient(brokerURL string) *GameMQTTClient {
    opts := mqtt.NewClientOptions()
    opts.AddBroker(brokerURL)
    opts.SetClientID(fmt.Sprintf("game_server_%d", time.Now().UnixNano()))
    opts.SetAutoReconnect(true)
    opts.SetCleanSession(true)
    
    client := mqtt.NewClient(opts)
    if token := client.Connect(); token.Wait() && token.Error() != nil {
        panic(token.Error())
    }
    
    return &GameMQTTClient{client: client}
}

// Subscribe to all answer submissions
func (m *GameMQTTClient) SubscribeAnswers(handler func(roomCode string, answer AnswerSubmit)) {
    topic := "game/rooms/+/answers/submit"
    
    m.client.Subscribe(topic, 1, func(c mqtt.Client, msg mqtt.Message) {
        // Extract roomCode from topic: game/rooms/{roomCode}/answers/submit
        parts := strings.Split(msg.Topic(), "/")
        roomCode := parts[2]
        
        var answer AnswerSubmit
        json.Unmarshal(msg.Payload(), &answer)
        
        handler(roomCode, answer)
    })
}

// Publish question to room
func (m *GameMQTTClient) PublishQuestion(roomCode string, question Question) {
    topic := fmt.Sprintf("game/rooms/%s/questions/current", roomCode)
    payload, _ := json.Marshal(question)
    
    m.client.Publish(topic, 1, true, payload)  // retained = true
}

// Publish leaderboard update
func (m *GameMQTTClient) PublishLeaderboard(roomCode string, leaderboard Leaderboard) {
    topic := fmt.Sprintf("game/rooms/%s/leaderboard", roomCode)
    payload, _ := json.Marshal(leaderboard)
    
    m.client.Publish(topic, 0, false, payload)  // QoS 0, not retained
}

// Publish room status
func (m *GameMQTTClient) PublishRoomStatus(roomCode string, status RoomStatus) {
    topic := fmt.Sprintf("game/rooms/%s/status", roomCode)
    payload, _ := json.Marshal(status)
    
    m.client.Publish(topic, 1, true, payload)  // retained
}
```

---

## IV. GAME FLOW VỚI MQTT

### 4.1 Sequence Diagram

```
Host              Broker              Server             Players
  │                  │                   │                   │
  │──Create Room────►│                   │                   │
  │                  │◄──Subscribe───────│                   │
  │◄─Room Created────│                   │                   │
  │                  │                   │                   │
  │                  │◄─────────────Join Room───────────────│
  │                  │──Player Joined───►│                   │
  │                  │───────────────────┼──Player Joined──►│
  │                  │                   │                   │
  │──Start Game─────►│                   │                   │
  │                  │──Game Started────►│                   │
  │                  │───────────────────┼───Game Started──►│
  │                  │                   │                   │
  │                  │◄──Pub Question────│                   │
  │◄─Question────────│───────────────────┼────Question─────►│
  │                  │                   │                   │
  │                  │◄──────────────Submit Answer──────────│
  │                  │───Answer─────────►│                   │
  │                  │                   │──Process──┐       │
  │                  │                   │◄──────────┘       │
  │                  │◄──Pub Leaderboard─│                   │
  │◄─Leaderboard─────│───────────────────┼───Leaderboard───►│
  │                  │                   │                   │
```

### 4.2 Room Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ROOM LIFECYCLE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐          │
│  │ CREATED │───►│ WAITING │───►│ PLAYING │───►│  ENDED  │          │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘          │
│       │              │              │              │                 │
│  Topics:        Topics:        Topics:        Topics:               │
│  • status       • status       • status       • status              │
│  • config       • players/*    • questions/*  • leaderboard         │
│                               • answers/*                            │
│                               • leaderboard                          │
│                               • timer                                │
│                                                                      │
│  Retained:      Retained:      Retained:      Retained:             │
│  • config       • players/list • questions/   • final results       │
│                               current                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## V. SCALING STRATEGY (Theo giai đoạn)

### 5.1 Phase 1: Single Node (Đủ cho 99% use cases)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   PHASE 1: SIMPLE SETUP                              │
│                   (< 50,000 connections)                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Browsers ──────► EMQX (single) ──────► Game Server                │
│                       │                       │                      │
│                       │                       ↓                      │
│                       │               Redis + PostgreSQL             │
│                       │                                              │
│   Capacity: 500,000 connections                                      │
│   Cost: $0 (EMQX Community Edition)                                  │
│   Ops: Đơn giản, 1 container                                         │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Phase 2: High Availability (Khi cần uptime 99.9%+)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   PHASE 2: HA SETUP                                  │
│                   (Production critical, > 50k)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│              Load Balancer (Nginx/HAProxy)                          │
│                         │                                            │
│                ┌────────┴────────┐                                  │
│            ┌───┴───┐         ┌───┴───┐                              │
│            │ EMQX  │ ◄─────► │ EMQX  │                              │
│            │ Node1 │         │ Node2 │                              │
│            └───────┘         └───────┘                              │
│                                                                      │
│   Capacity: 1M+ connections                                          │
│   HA: 1 node die → vẫn chạy                                         │
│   Zero-downtime upgrade                                              │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.3 Khi nào upgrade?

| Điều kiện | Action |
|-----------|--------|
| < 50k connections | Giữ Phase 1 |
| Game down = mất tiền | Chuyển Phase 2 |
| > 100k connections | Chuyển Phase 2 |
| Cần zero-downtime deploy | Chuyển Phase 2 |

### 5.4 Capacity Reference

| Component | Capacity | Notes |
|-----------|----------|-------|
| **EMQX (1 node)** | ~500k connections | Đủ cho hầu hết cases |
| **EMQX (2 nodes)** | ~1M connections | HA + scale |
| **Game Server** | Stateless | Scale khi cần |
| **Redis** | Cache only | Room state, sessions |
| **PostgreSQL** | Persistent | Results, history |

---

## VI. SECURITY

### 6.1 Authentication

```yaml
# emqx.conf
authentication = [
  {
    mechanism = password_based
    backend = http
    enable = true
    
    method = post
    url = "http://game-server:8080/api/mqtt/auth"
    body {
      username = "${username}"
      password = "${password}"
    }
    headers {
      content-type = "application/json"
    }
  }
]
```

### 6.2 Authorization (ACL)

```yaml
# emqx.conf
authorization {
  sources = [
    {
      type = http
      enable = true
      
      method = post
      url = "http://game-server:8080/api/mqtt/acl"
      body {
        username = "${username}"
        topic = "${topic}"
        action = "${action}"
      }
    }
  ]
}
```

```go
// ACL endpoint
func HandleMQTTACL(c echo.Context) error {
    req := struct {
        Username string `json:"username"`
        Topic    string `json:"topic"`
        Action   string `json:"action"`  // publish/subscribe
    }{}
    c.Bind(&req)
    
    // Parse: guest_xyz789 hoặc user_abc123
    playerType, playerId := parseUsername(req.Username)
    
    // Extract roomCode from topic
    roomCode := extractRoomCode(req.Topic)
    
    // Check if player is in room
    if !isPlayerInRoom(playerId, roomCode) {
        return c.JSON(200, map[string]string{"result": "deny"})
    }
    
    // Host can publish to questions, others cannot
    if strings.Contains(req.Topic, "/questions/") && req.Action == "publish" {
        if !isHost(playerId, roomCode) {
            return c.JSON(200, map[string]string{"result": "deny"})
        }
    }
    
    return c.JSON(200, map[string]string{"result": "allow"})
}
```

---

## VII. DEPLOYMENT

### 7.1 Docker Compose (Development)

```yaml
version: '3.8'

services:
  emqx:
    image: emqx/emqx:5.4.0
    ports:
      - "1883:1883"
      - "8084:8084"
      - "18083:18083"
    volumes:
      - ./config/emqx.conf:/opt/emqx/etc/emqx.conf

  game-server:
    build: ./game-server
    ports:
      - "8080:8080"
    environment:
      - MQTT_BROKER=tcp://emqx:1883
      - DATABASE_URL=postgres://...
      - REDIS_URL=redis://redis:6379
    depends_on:
      - emqx
      - postgres
      - redis

  postgres:
    image: postgres:15
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

volumes:
  postgres-data:
  redis-data:
```

### 7.2 Kubernetes (Production)

```yaml
# emqx-cluster.yaml
apiVersion: apps.emqx.io/v2beta1
kind: EMQX
metadata:
  name: emqx
spec:
  image: emqx/emqx:5.4.0
  coreTemplate:
    spec:
      replicas: 3
  replicantTemplate:
    spec:
      replicas: 3
```

---

## VIII. MONITORING

### 8.1 EMQX Dashboard

```
http://localhost:18083
- Connections
- Subscriptions  
- Message rate
- Cluster status
```

### 8.2 Prometheus Metrics

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'emqx'
    static_configs:
      - targets: ['emqx:18083']
    metrics_path: /api/v5/prometheus/stats
```

### 8.3 Key Metrics

| Metric | Alert threshold |
|--------|-----------------|
| `emqx_connections_count` | > 80% capacity |
| `emqx_messages_sent_rate` | > 10k/s |
| `emqx_messages_dropped_rate` | > 0 |
| `emqx_subscriptions_count` | monitoring |

---

## IX. TIMELINE

| Tuần | Task |
|------|------|
| 1 | Setup EMQX, basic pub/sub working |
| 2 | Implement room lifecycle với MQTT |
| 3 | Frontend MQTT integration |
| 4 | Game flow complete (question, answer, score) |
| 5 | Testing, security, edge cases |
| 6 | Deployment, monitoring |

---

*Version 6 - Complete MQTT over WebSocket Architecture*
