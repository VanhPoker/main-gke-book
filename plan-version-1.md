# Plan Game Module - Version 1 (Scale-focused)

Đây là bản plan tập trung vào scalability cho module Game đã có.
Đã archived vì đang chuyển hướng sang thiết kế module độc lập mới.

---

## Nội dung chính của Version 1:

### I. Xây dựng MVP và Tối ưu
- Giai đoạn 1: MVP (~100 players)
- Giai đoạn 2: Optimize (100 → 500)
- Giai đoạn 3: External Storage (1,000 → 5,000)
- Giai đoạn 4: Horizontal Scale (5,000 → 10,000+)

### II. Kỹ thuật scaling
- Parallel Broadcast (Goroutines)
- Non-blocking Submit Handler
- Per-Room Locks
- Message Batching
- Leaderboard Throttling
- Redis Game State
- Redis Pub/Sub
- Load Balancer

### III. Edge Cases
- Duplicate submit
- Late submit
- WebSocket disconnect/reconnect
- Server restart recovery

### IV. Database Schema
- game_sessions
- game_player_results
- game_answer_details

### V. Monitoring & Testing
- Metrics: P99 latency, queue length, error rate
- Load testing, Chaos testing

### VI. Development Rules
- Module isolation
- Backward compatibility
- Database migration rules
- Code review process

---

**Lý do archive**: 
Chuyển sang phương hướng mới - module game độc lập với:
- Guest players (không cần đăng nhập)
- Avatar system
- Nghiên cứu giao thức thay thế WebSocket (gRPC, MQTT)
