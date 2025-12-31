# Plan Game Module - Version 5 (Access Control & Concurrent Submit)

## Bổ sung so với Version 4

1. **Quyền truy cập bộ đề** - Ai được tạo game với bộ đề nào
2. **Xử lý concurrent submit** - Goroutine queue, ordering chính xác

---

## I. QUYỀN TRUY CẬP BỘ ĐỀ (QUIZ ACCESS CONTROL)

### 1.1 Các loại bộ đề

| Loại | Mô tả | Ai xem được? |
|------|-------|--------------|
| **Personal** | Giáo viên tự tạo | Chỉ người tạo |
| **Class** | Bộ đề của lớp học | Thành viên trong lớp |
| **School** | Bộ đề của trường | Thành viên trường |
| **Public** | Bộ đề công khai | Tất cả mọi người |

### 1.2 Logic kiểm tra quyền khi tạo game

```go
// Khi user chọn bộ đề để tạo game
func CanCreateGameWithQuiz(userId string, quizId string) (bool, error) {
    quiz := GetQuiz(quizId)
    user := GetUser(userId)
    
    switch quiz.Visibility {
    case "public":
        // Ai cũng dùng được
        return true, nil
        
    case "personal":
        // Chỉ người tạo
        return quiz.CreatedBy == userId, nil
        
    case "class":
        // Thành viên của lớp đó
        return IsClassMember(userId, quiz.ClassId), nil
        
    case "school":
        // Thành viên của trường đó
        return user.SchoolId == quiz.SchoolId, nil
        
    default:
        return false, errors.New("UNKNOWN_VISIBILITY")
    }
}
```

### 1.3 API lấy danh sách Quiz có thể dùng

```go
// GET /api/game/available-quizzes
// Trả về tất cả quiz mà user có quyền tạo game

func GetAvailableQuizzes(userId string) []Quiz {
    user := GetUser(userId)
    
    quizzes := []Quiz{}
    
    // 1. Public quizzes
    quizzes = append(quizzes, GetPublicQuizzes()...)
    
    // 2. Personal quizzes (do user tạo)
    quizzes = append(quizzes, GetQuizzesByCreator(userId)...)
    
    // 3. Class quizzes (các lớp user tham gia)
    for _, classId := range user.ClassIds {
        quizzes = append(quizzes, GetClassQuizzes(classId)...)
    }
    
    // 4. School quizzes
    if user.SchoolId != "" {
        quizzes = append(quizzes, GetSchoolQuizzes(user.SchoolId)...)
    }
    
    return DedupAndSort(quizzes)
}
```

### 1.4 UI cho Host chọn bộ đề

```
┌─────────────────────────────────────────────────────────────┐
│  CHỌN BỘ ĐỀ ĐỂ TẠO GAME                                     │
│                                                              │
│  🔍 Tìm kiếm: [___________________]                         │
│                                                              │
│  ─── BỘ ĐỀ CỦA BẠN ───────────────────────────────────────  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 📝 Quiz Toán Lớp 5 - Chương 1        (15 câu)    [CHỌN] ││
│  │ 📝 Quiz Tiếng Việt - Từ vựng         (20 câu)    [CHỌN] ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  ─── BỘ ĐỀ LỚP 5A ────────────────────────────────────────  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 📚 Kiểm tra giữa kỳ Toán             (30 câu)    [CHỌN] ││
│  │ 📚 Ôn tập cuối kỳ                    (40 câu)    [CHỌN] ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  ─── BỘ ĐỀ TRƯỜNG THCS ABC ───────────────────────────────  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 🏫 Thi HSG Toán Cấp Trường           (50 câu)    [CHỌN] ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  ─── BỘ ĐỀ CÔNG KHAI ─────────────────────────────────────  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 🌐 Quiz IQ Test                      (25 câu)    [CHỌN] ││
│  │ 🌐 Kiến thức chung Việt Nam          (30 câu)    [CHỌN] ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### 1.5 Guest tạo game?

| Loại user | Được tạo game với quiz nào? |
|-----------|----------------------------|
| **Guest** | Chỉ Public quizzes |
| **Account** | Public + Personal + Class + School |

```go
func GetAvailableQuizzesForHost(hostType string, userId string) []Quiz {
    if hostType == "guest" {
        // Guest chỉ dùng được public
        return GetPublicQuizzes()
    }
    
    // Logged-in user có full access theo quyền
    return GetAvailableQuizzes(userId)
}
```

---

## II. XỬ LÝ CONCURRENT SUBMIT (Nhiều người nộp cùng lúc)

### 2.1 Vấn đề

```
Thời điểm T = 10.000ms (10 giây sau khi câu hỏi bắt đầu)

Player A submit lúc client time: 10.001ms
Player B submit lúc client time: 10.002ms
Player C submit lúc client time: 10.001ms  ← Cùng thời gian với A!

Network latency khác nhau:
- A: latency 50ms  → Server nhận lúc 10.051ms
- B: latency 20ms  → Server nhận lúc 10.022ms  ← Server nhận B TRƯỚC A!
- C: latency 100ms → Server nhận lúc 10.101ms

Vấn đề: 
- Theo client time: A = C < B
- Theo server receive: B < A < C
- Ai "nhanh" hơn thực sự???
```

### 2.2 Giải pháp: Server Timestamp + Queue

**Nguyên tắc:** 
- Không tin client time (có thể gian lận)
- Dùng server receive time làm chuẩn
- Xếp thứ tự theo server time

```go
// Submit được gom vào channel
type SubmitRequest struct {
    PlayerId    string
    QuestionId  string
    ChoiceId    string
    ClientTime  int64     // Client gửi (không tin)
    ServerTime  int64     // Server ghi nhận (TIN)
}

// Buffered channel để gom submits
submitQueue := make(chan SubmitRequest, 10000)

// Khi nhận submit từ WebSocket
func handleSubmit(ws *websocket.Conn, msg []byte) {
    req := parseSubmit(msg)
    req.ServerTime = time.Now().UnixNano()  // ← Ghi server time NGAY
    
    submitQueue <- req  // Đẩy vào queue, không block
}
```

### 2.3 Worker Pool xử lý Queue

```go
// Khởi động N workers
func startSubmitWorkers(n int) {
    for i := 0; i < n; i++ {
        go submitWorker(i)
    }
}

func submitWorker(id int) {
    for req := range submitQueue {
        processSubmit(req)
    }
}

func processSubmit(req SubmitRequest) {
    room := GetRoom(req.RoomId)
    
    // Check duplicate (đã submit câu này chưa)
    key := req.PlayerId + ":" + req.QuestionId
    if room.HasSubmitted(key) {
        return  // Bỏ qua duplicate
    }
    room.MarkSubmitted(key)
    
    // Validate answer
    question := room.GetQuestion(req.QuestionId)
    isCorrect := question.IsCorrect(req.ChoiceId)
    
    // Tính điểm dựa trên thời gian
    responseTime := req.ServerTime - question.StartTime
    points := calculatePoints(isCorrect, responseTime, question.TimeLimit)
    
    // Lưu kết quả với ServerTime để xếp hạng
    room.SaveAnswer(Answer{
        PlayerId:     req.PlayerId,
        QuestionId:   req.QuestionId,
        ChoiceId:     req.ChoiceId,
        IsCorrect:    isCorrect,
        Points:       points,
        ResponseTime: responseTime,
        ServerTime:   req.ServerTime,  // ← Dùng để xếp hạng ai nhanh hơn
    })
    
    // Broadcast result
    broadcastAnswerResult(room.Id, req.PlayerId, isCorrect, points)
}
```

### 2.4 Xếp hạng khi cùng điểm

Khi 2 người cùng điểm, ai nhanh hơn?

```go
func calculateFinalRanking(room *Room) []PlayerRank {
    players := room.GetPlayers()
    
    // Tính tổng điểm và tổng thời gian
    for _, p := range players {
        p.TotalScore = sumScore(p.Answers)
        p.TotalResponseTime = sumResponseTime(p.Answers)
    }
    
    // Sort: Điểm cao trước, nếu bằng điểm thì thời gian ít hơn trước
    sort.Slice(players, func(i, j int) bool {
        if players[i].TotalScore != players[j].TotalScore {
            return players[i].TotalScore > players[j].TotalScore  // Điểm cao hơn
        }
        return players[i].TotalResponseTime < players[j].TotalResponseTime  // Nhanh hơn
    })
    
    return players
}
```

### 2.5 Đảm bảo ordering trong cùng 1 câu hỏi

Nếu muốn biết AI trả lời câu X TRƯỚC nhất:

```go
// Lấy người trả lời đúng đầu tiên của câu hỏi
func GetFirstCorrectAnswer(roomId, questionId string) *Answer {
    answers := GetAnswersByQuestion(roomId, questionId)
    
    // Filter đúng
    correctAnswers := filter(answers, func(a Answer) bool {
        return a.IsCorrect
    })
    
    if len(correctAnswers) == 0 {
        return nil
    }
    
    // Sort theo ServerTime, lấy đầu tiên
    sort.Slice(correctAnswers, func(i, j int) bool {
        return correctAnswers[i].ServerTime < correctAnswers[j].ServerTime
    })
    
    return &correctAnswers[0]
}
```

### 2.6 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     CONCURRENT SUBMIT HANDLING                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Player A ─┐                                                        │
│   Player B ─┼─► WebSocket ─► Main Handler ─► submitQueue (buffered) │
│   Player C ─┘      │              │              │                   │
│                    │              ↓              │                   │
│                    │    Ghi ServerTime ngay      │                   │
│                    │    Đẩy vào queue            │                   │
│                    │    Return ngay (non-block)  │                   │
│                    │                              ↓                   │
│                    │              ┌──────────────────────────┐       │
│                    │              │     Worker Pool (20)     │       │
│                    │              │  ┌────┐ ┌────┐ ┌────┐   │       │
│                    │              │  │ W1 │ │ W2 │ │... │   │       │
│                    │              │  └────┘ └────┘ └────┘   │       │
│                    │              │         ↓                │       │
│                    │              │  - Validate answer       │       │
│                    │              │  - Calculate points      │       │
│                    │              │  - Save to memory/Redis  │       │
│                    │              │  - Broadcast result      │       │
│                    │              └──────────────────────────┘       │
│                    │                              ↓                   │
│                    └─────────────────────────────────────────────────┘
│                               Results broadcast to all                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## III. TỔNG HỢP V5

| Vấn đề | Giải pháp |
|--------|-----------|
| **Quiz access control** | Check visibility: public/personal/class/school |
| **Guest tạo game** | Chỉ được dùng public quizzes |
| **Concurrent submit** | Channel queue + Worker pool |
| **Ai nộp trước?** | ServerTime làm chuẩn (không tin client) |
| **Cùng điểm?** | Xếp theo tổng thời gian response |

---

## IV. CÂU HỎI TIẾP THEO

1. **Có cần hiện "Người trả lời đầu tiên" mỗi câu không?**
   - [ ] Có, như feature "First to answer" của Kahoot
   - [ ] Không cần

2. **Số workers bao nhiêu?**
   - [ ] 10 (nhẹ)
   - [ ] 20 (cân bằng)
   - [ ] 50 (nặng nhưng nhanh)

3. **Queue size bao nhiêu?**
   - [ ] 1,000 (100 players × 10 questions)
   - [ ] 10,000 (1000 players × 10 questions)
   - [ ] 100,000 (buffer lớn cho burst)

---

*Version 5 - Bổ sung access control và concurrent submit handling.*
