# 🔑 RBAC vs ABAC trong phân quyền hệ thống

## 1. RBAC (Role-Based Access Control)
- **Khái niệm:** Quyền được gán theo vai trò (role). Người dùng được gán role, role có permission.
- **Ưu điểm:** Dễ hiểu, dễ triển khai, phù hợp hệ thống nhỏ/vừa.
- **Nhược điểm:** Thiếu linh hoạt, khó mở rộng khi nhiều điều kiện phức tạp.

**Case thực tế: CMS (Content Management System)**
- Admin: tạo/sửa/xóa bài viết.
- Editor: sửa bài nhưng không xóa.
- Viewer: chỉ xem.
👉 Phù hợp vì vai trò rõ ràng, ít điều kiện phức tạp.

---

## 2. ABAC (Attribute-Based Access Control)
- **Khái niệm:** Quyền dựa trên thuộc tính (user, resource, action, context).
- **Ưu điểm:** Linh hoạt, kiểm soát chi tiết, phù hợp hệ thống lớn.
- **Nhược điểm:** Triển khai phức tạp, cần policy engine để đánh giá.

**Case thực tế: Ngân hàng số (Fintech)**
- Người dùng chỉ xem giao dịch của chính mình.
- Nhân viên chỉ xem dữ liệu khách hàng trong chi nhánh.
- Truy cập bị hạn chế theo thời gian (ví dụ ngoài giờ hành chính).
👉 Phù hợp vì cần nhiều thuộc tính (user, branch, time, resource type).

---

## 3. Hệ thống y tế điện tử (EHR)
- **Áp dụng ABAC:**  
  - Bác sĩ chỉ xem hồ sơ bệnh nhân khi đang điều trị.  
  - Y tá chỉ xem thông tin cơ bản, không xem dữ liệu nhạy cảm.  
  - Quyền truy cập phụ thuộc vào khoa, ca trực, loại dữ liệu.  
👉 Đảm bảo tuân thủ quy định bảo mật (HIPAA, GDPR).

---

## 4. Lưu ý triển khai
- **RBAC:** tránh hard-code role trong code (`if(user.role == 2)`), nên quản lý role/permission trong DB hoặc IAM service.
- **ABAC:** cần policy engine (ví dụ: XACML, Open Policy Agent) để đánh giá thuộc tính.
- **Best practice:**  
  - Hệ thống nhỏ/vừa → RBAC.  
  - Hệ thống lớn, nhiều điều kiện → ABAC.  
  - Có thể kết hợp: RBAC để quản lý vai trò chính, ABAC để bổ sung điều kiện chi tiết.

---

## 📌 Tổng kết
- **RBAC:** dễ triển khai, phù hợp hệ thống có vai trò rõ ràng.  
- **ABAC:** linh hoạt, phù hợp hệ thống phức tạp, nhiều điều kiện ngữ cảnh.  
- **Thực tế:** nhiều doanh nghiệp hiện nay kết hợp cả hai – RBAC để quản lý vai trò cơ bản, ABAC để kiểm soát chi tiết theo thuộc tính.
- - **Case điển hình:**  
  - CMS → RBAC.  
  - Fintech → ABAC.  
  - Y tế → ABAC.  
  - E-Commerce → Hybrid RBAC + ABAC.
