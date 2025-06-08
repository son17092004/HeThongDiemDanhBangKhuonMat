# BUG FIX: Logic tính điểm chuyên cần không chính xác

## 🐛 Vấn đề phát hiện

### Biểu hiện lỗi

- API `/api/attendance/student/:studentId/scores` trả về kết quả sai:
  - Sinh viên có mặt đầy đủ 3/3 buổi nhưng hiển thị "Cấm thi (vắng 9/3 buổi)"
  - Điểm chuyên cần: 0/10 thay vì 10/10
  - Tỷ lệ tham gia: -200% thay vì 100%
  - `absent_sessions: 9`, `attended_sessions: -6` (giá trị âm!)

### Nguyên nhân gốc rễ

Logic tính điểm chuyên cần **đếm cả những buổi chưa học** như là "vắng mặt":

- Lớp có 12 buổi dự kiến, chỉ hoàn thành 3 buổi
- Hệ thống tính: 12 - 3 = 9 buổi vắng
- Thực tế: Sinh viên có mặt đủ 3/3 buổi đã hoàn thành

## ✅ Giải pháp đã triển khai

### 1. Sửa logic trong `updateAttendanceScores()`

**Trước khi sửa:**

```javascript
// Logic cũ: Đếm cả session chưa diễn ra như "vắng"
const totalSessions = teachingClass.total_sessions; // 12 buổi dự kiến
const absentCount = totalSessions - presentCount; // 12 - 3 = 9 vắng ❌
```

**Sau khi sửa:**

```javascript
// Logic mới: CHỈ đếm sessions đã hoàn thành (completed)
const completedSessions = await AttendanceSession.find({
  teaching_class_id: teachingClassId,
  status: "completed", // CHỈ những buổi đã hoàn thành
});

const totalCompletedSessions = completedSessions.length; // 3 buổi hoàn thành
const absentCount = totalCompletedSessions - presentCount; // 3 - 3 = 0 vắng ✅
```

### 2. Sử dụng AttendanceLog làm nguồn dữ liệu chính thống

```javascript
// Chỉ đếm những sinh viên có log "present" trong AttendanceLog
const attendanceLogs = await AttendanceLog.aggregate([
  {
    $match: {
      student_id: { $in: teachingClass.students },
      session_id: { $in: completedSessionIds },
      status: "present", // CHỈ đếm status "present"
    },
  },
  {
    $group: {
      _id: "$student_id",
      presentCount: { $sum: 1 },
    },
  },
]);
```

### 3. Công thức tính điểm được cập nhật

```javascript
// Tính điểm dựa trên số buổi vắng THỰC TẾ
const attendanceScore = Math.max(0, 10 - absent_count * 2);
const isFailedDueToAbsent = absent_count > maxAbsentAllowed;
```

## 📊 Kết quả sau khi sửa

### Lớp "Hệ Thống Thông Tin D22CQCN02-N"

| Chỉ số             | Trước khi sửa | Sau khi sửa     |
| ------------------ | ------------- | --------------- |
| Điểm chuyên cần    | 0/10          | 10/10 ✅        |
| Số buổi hoàn thành | 3             | 3               |
| Số buổi có mặt     | -6 ❌         | 3 ✅            |
| Số buổi vắng       | 9 ❌          | 0 ✅            |
| Tỷ lệ tham gia     | -200% ❌      | 100% ✅         |
| Trạng thái         | Cấm thi ❌    | Đủ điều kiện ✅ |

### Lớp "Xác suất thống kê"

| Chỉ số             | Kết quả                    |
| ------------------ | -------------------------- |
| Điểm chuyên cần    | 2/10                       |
| Số buổi hoàn thành | 12                         |
| Số buổi có mặt     | 8                          |
| Số buổi vắng       | 4                          |
| Tỷ lệ tham gia     | 67%                        |
| Trạng thái         | Cấm thi (vắng 4/3 buổi) ✅ |

## 🔧 Các file đã được sửa

1. **`backend/src/controllers/attendance.controller.js`**

   - Hàm `updateAttendanceScores()`: Sửa logic tính điểm
   - Hàm `getStudentScores()`: Đã hoạt động đúng (chỉ hiển thị dữ liệu)

2. **Frontend không cần sửa** - vấn đề nằm ở backend logic

## 🎯 Nguyên tắc mới được áp dụng

1. **Chỉ tính dựa trên sessions đã hoàn thành:** Không đếm sessions chưa diễn ra
2. **AttendanceLog là nguồn chính thống:** Dữ liệu điểm danh chính xác nhất
3. **Tính điểm thực tế:** Điểm số phản ánh đúng tỷ lệ tham gia thực tế
4. **Validation dữ liệu:** Không cho phép giá trị âm hoặc bất hợp lý

## ✅ Tính năng hoạt động chính xác

- ✅ Điểm chuyên cần phản ánh đúng thực tế
- ✅ Tỷ lệ tham gia tính chính xác (0-100%)
- ✅ Trạng thái cấm thi dựa trên số buổi vắng thực tế
- ✅ Giao diện ScoresPage hiển thị đúng dữ liệu
- ✅ Logic tương thích với các lớp đang học và đã hoàn thành

## 🚀 Kết luận

Bug nghiêm trọng về logic tính điểm chuyên cần đã được **khắc phục hoàn toàn**. Hệ thống giờ đây:

- Chỉ tính điểm dựa trên những buổi đã thực sự diễn ra
- Không penalize sinh viên vì những buổi chưa học
- Đảm bảo tính chính xác và công bằng trong đánh giá

**Nguy cơ:** Trước đây có thể có nhiều sinh viên bị đánh giá sai và cấm thi một cách bất công. Với logic mới, tất cả điểm số sẽ được tính lại chính xác.
