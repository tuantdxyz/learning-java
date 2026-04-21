# 🛡️ Tổng hợp cơ chế JWT

## 1. Access Token & Refresh Token
- **Access Token**: sống ngắn (15–30 phút), dùng để gọi API.
- **Refresh Token**: sống dài (vài ngày–tuần), dùng để xin Access Token mới.
- **JTI (JWT ID)**: định danh duy nhất cho mỗi token, phục vụ blacklist.

**Tóm tắt:** Access Token cho request nhanh, Refresh Token để duy trì phiên lâu dài.

---

## 2. Redis Blacklist + TTL tối ưu
- Khi revoke token, lưu `jti` vào Redis.
- TTL của key = thời gian còn lại của token (`exp - now`).
- Redis tự xóa key khi token hết hạn → tránh phình to bộ nhớ.

**Ví dụ code:**
const ttl = decoded.exp - Math.floor(Date.now() / 1000);
await redis.set(`bl_${jti}`, true, 'EX', ttl);

---

## 3. Refresh Token Rotation
Mỗi lần dùng Refresh Token để xin Access Token mới → sinh Refresh Token mới.

Refresh Token cũ được đánh dấu used.

Tóm tắt: Rotation đảm bảo token cũ không thể dùng lại.

---

## 4. Reuse Detection
Nếu Refresh Token đã used mà lại xuất hiện lần nữa → dấu hiệu bị đánh cắp.

Hệ thống phát hiện reuse → kích hoạt Nuclear Revoke.

Tóm tắt: Phát hiện hacker dùng lại token cũ.

---

## 5. Nuclear Revoke
Khi phát hiện reuse → xóa toàn bộ session của user.

Các Refresh Token còn lại bị đánh dấu revoked.

Tóm tắt: Chặn hacker bằng cách hủy toàn bộ phiên.

---

## 6. Trạng thái token trong DB/Redis
| Trạng thái | Ý nghĩa |
| --- | --- |
| ``valid`` | Token còn hợp lệ |
| ``used`` | Token đã được dùng để cấp token mới |
| ``revoked`` | Token đã bị hủy (logout hoặc Nuclear Revoke) |

Ví dụ dữ liệu:

| id | user_id | token | status | issued_at | expires_at |
| --- | --- | --- | --- | --- | --- |
| 1 | U123 | RT1 | used | 2026-04-20 09:00:00 | 2026-04-27 09:00:00 |
| 2 | U123 | RT2 | valid | 2026-04-20 09:30:00 | 2026-04-27 09:30:00 |
| 3 | U123 | RT3 | revoked | 2026-04-19 08:00:00 | 2026-04-26 08:00:00 |


Tóm tắt: DB/Redis lưu trạng thái để quản lý vòng đời token.

---

## 7. Luồng hoạt động tổng quát
User login → cấp Access Token + Refresh Token (RT1).

Access Token hết hạn → client gọi /refresh với RT1.

Server cấp Access Token mới + Refresh Token mới (RT2), đánh dấu RT1 = used.

Hacker gửi lại RT1 → server phát hiện reuse → Nuclear Revoke → toàn bộ session bị xóa, RT2 = revoked.

Tóm tắt: Luồng chuẩn giúp vừa duy trì phiên cho user, vừa chặn hacker.

---

## 8. Frontend Interceptor để gọi Refresh Token
Cơ chế: Client-side (FE) không tự check hết hạn, mà dùng interceptor để bắt lỗi 401 từ API.

Khi gặp 401 → tự động gọi /refresh với Refresh Token → nhận Access Token mới → retry request ban đầu.

Ví dụ với Axios (React/JS):

import axios from "axios";

const api = axios.create({ baseURL: "http://localhost:8080/api" });

api.interceptors.response.use(
  response => response,
  async error => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const refreshToken = localStorage.getItem("refreshToken");
        const res = await axios.post("http://localhost:8080/auth/refresh", { refreshToken });

        localStorage.setItem("accessToken", res.data.accessToken);
        localStorage.setItem("refreshToken", res.data.refreshToken);

        api.defaults.headers.common["Authorization"] = `Bearer ${res.data.accessToken}`;
        originalRequest.headers["Authorization"] = `Bearer ${res.data.accessToken}`;

        return api(originalRequest);
      } catch (refreshError) {
        window.location.href = "/login"; // Nếu refresh fail → logout
      }
    }

    return Promise.reject(error);
  }
);

---

📌 Tổng kết
Redis Blacklist + TTL: thu hồi token stateless, tự động dọn dẹp.

Refresh Token Rotation: mỗi lần refresh sinh token mới.

Reuse Detection: phát hiện hacker dùng lại token cũ.

Nuclear Revoke: xóa toàn bộ session khi phát hiện reuse.

DB/Redis trạng thái: valid | used | revoked để quản lý vòng đời token.
