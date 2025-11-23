# HƯỚNG DẪN DEPLOY ỨNG DỤNG QUA CLOUDFLARE TUNNEL

## Tổng quan
Hướng dẫn này giúp bạn expose ứng dụng local (Frontend + Backend) ra Internet bằng Cloudflare Tunnel, cho phép người khác truy cập qua domain của bạn.

---

## YÊU CẦU

### 1. Tài khoản Cloudflare
- Đăng ký miễn phí tại: https://dash.cloudflare.com/sign-up
- Có domain đã add vào Cloudflare (hoặc mua domain từ Cloudflare)

### 2. Cài đặt Cloudflared
- Download tại: https://github.com/cloudflare/cloudflared/releases
- Hoặc dùng lệnh: `winget install --id Cloudflare.cloudflared`

### 3. Cấu trúc dự án
```
my-project/
├── backend/          # Backend API (FastAPI, Express, Django, etc.)
├── frontend/         # Frontend (React, Vue, Angular, etc.)
└── ...
```

---

## BƯỚC 1: CÀI ĐẶT CLOUDFLARED

### Windows
```powershell
# Cách 1: Dùng winget
winget install --id Cloudflare.cloudflared

# Cách 2: Download thủ công
# Tải file .exe từ GitHub releases
# Copy vào C:\Windows\System32\ hoặc thêm vào PATH
```

### Kiểm tra cài đặt
```powershell
cloudflared --version
```

---

## BƯỚC 2: ĐĂNG NHẬP CLOUDFLARE

```powershell
cloudflared tunnel login
```

- Trình duyệt sẽ mở ra
- Chọn domain bạn muốn dùng
- Authorize
- File cert.pem sẽ được tạo tại: `C:\Users\[USER]\.cloudflared\cert.pem`

---

## BƯỚC 3: TẠO TUNNEL

### Tạo tunnel mới
```powershell
cloudflared tunnel create my-app-tunnel
```

Kết quả:
```
Tunnel credentials written to C:\Users\[USER]\.cloudflared\[TUNNEL-ID].json
Created tunnel my-app-tunnel with id [TUNNEL-ID]
```

**LƯU Ý**: Giữ file JSON này cẩn thận, đây là credentials của tunnel!

### Lấy danh sách tunnels
```powershell
cloudflared tunnel list
```

### Lấy token của tunnel (quan trọng!)
```powershell
cloudflared tunnel token [TUNNEL-ID]
```

Hoặc dùng tên tunnel:
```powershell
cloudflared tunnel token my-app-tunnel
```

**LƯU TOKEN NÀY** - Sẽ dùng để chạy tunnel sau này!

---

## BƯỚC 4: CÀI ĐẶT DNS

### Cách 1: Qua Dashboard (Khuyến nghị)
1. Truy cập: https://one.dash.cloudflare.com/
2. Networks → Tunnels
3. Click vào tunnel vừa tạo
4. Tab "Public Hostname"
5. Click "Add a published application route"

#### Cấu hình Frontend
- **Subdomain**: `app` (hoặc tên bạn muốn)
- **Domain**: chọn domain của bạn (vd: `mydomain.com`)
- **Service Type**: `HTTP`
- **Service URL**: `http://localhost:3000` (port frontend của bạn)

→ Kết quả: `app.mydomain.com` trỏ đến `http://localhost:3000`

#### Cấu hình Backend
- **Subdomain**: `api` (hoặc tên bạn muốn)
- **Domain**: chọn domain của bạn
- **Service Type**: `HTTP`
- **Service URL**: `http://localhost:8000` (port backend của bạn)

→ Kết quả: `api.mydomain.com` trỏ đến `http://localhost:8000`

### Cách 2: Qua Command Line
```powershell
# Route cho frontend
cloudflared tunnel route dns my-app-tunnel app.mydomain.com

# Route cho backend
cloudflared tunnel route dns my-app-tunnel api.mydomain.com
```

---

## BƯỚC 5: CẤU HÌNH BACKEND

### 1. Cấu hình CORS

Backend cần cho phép truy cập từ domain của bạn.

#### FastAPI (Python)
```python
# backend/app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# CORS Configuration
origins = [
    "http://localhost:3000",           # Local development
    "https://app.mydomain.com",        # Production domain
    "https://api.mydomain.com",        # API domain
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

#### Express.js (Node.js)
```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

const corsOptions = {
  origin: [
    'http://localhost:3000',
    'https://app.mydomain.com',
    'https://api.mydomain.com'
  ],
  credentials: true
};

app.use(cors(corsOptions));
```

#### Django (Python)
```python
# backend/settings.py
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "https://app.mydomain.com",
    "https://api.mydomain.com",
]

CORS_ALLOW_CREDENTIALS = True
```

---

## BƯỚC 6: CẤU HÌNH FRONTEND

### 1. Tạo file Environment Variables

Tạo file `.env.production` trong thư mục frontend:

```env
# .env.production
VITE_API_BASE_URL=https://api.mydomain.com/api/v1
# Hoặc với React (Create React App)
REACT_APP_API_BASE_URL=https://api.mydomain.com/api/v1
# Hoặc với Next.js
NEXT_PUBLIC_API_BASE_URL=https://api.mydomain.com/api/v1
```

**QUAN TRỌNG**: Phải dùng `https://` không phải `http://`

### 2. Sử dụng trong code

#### React/Vite
```typescript
// src/lib/api.ts
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000/api/v1';

export const api = {
  login: async (username: string, password: string) => {
    const response = await fetch(`${API_BASE_URL}/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username, password })
    });
    return response.json();
  }
};
```

#### React (Create React App)
```javascript
// src/config.js
const API_BASE_URL = process.env.REACT_APP_API_BASE_URL || 'http://localhost:8000/api/v1';
```

#### Next.js
```javascript
// src/config.js
const API_BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL || 'http://localhost:8000/api/v1';
```

---

## BƯỚC 7: CHẠY TUNNEL

### Cách 1: Chạy bằng Token (Đơn giản nhất)

Tạo file `start_tunnel.bat`:
```batch
@echo off
echo Starting Cloudflare Tunnel...
cloudflared tunnel run --token [TOKEN-CỦA-BẠN]
pause
```

**Lấy token**: `cloudflared tunnel token my-app-tunnel`

### Cách 2: Chạy bằng Config File (Nâng cao)

Tạo file `C:\Users\[USER]\.cloudflared\config.yml`:
```yaml
tunnel: [TUNNEL-ID]
credentials-file: C:\Users\[USER]\.cloudflared\[TUNNEL-ID].json

ingress:
  # Frontend route
  - hostname: app.mydomain.com
    service: http://localhost:3000
  
  # Backend route
  - hostname: api.mydomain.com
    service: http://localhost:8000
  
  # Catch-all rule (bắt buộc)
  - service: http_status:404
```

Chạy tunnel:
```powershell
cloudflared tunnel --config C:\Users\[USER]\.cloudflared\config.yml run
```

---

## BƯỚC 8: KHỞI ĐỘNG ỨNG DỤNG

### Script khởi động tất cả (Windows)

Tạo file `start_all.bat`:
```batch
@echo off
echo ========================================
echo Starting All Services
echo ========================================

REM Start Backend
echo.
echo [1/3] Starting Backend...
start "Backend Server" cmd /k "cd backend && python -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000"

REM Wait a bit
timeout /t 3 /nobreak >nul

REM Start Cloudflare Tunnel
echo.
echo [2/3] Starting Cloudflare Tunnel...
start "Cloudflare Tunnel" cmd /k "cloudflared tunnel run --token [TOKEN-CỦA-BẠN]"

REM Wait a bit
timeout /t 3 /nobreak >nul

REM Start Frontend
echo.
echo [3/3] Starting Frontend...
start "Frontend Dev Server" cmd /k "cd frontend && npm run dev"

echo.
echo ========================================
echo All services started!
echo ========================================
echo.
echo Backend:  http://localhost:8000
echo Frontend: http://localhost:3000
echo.
echo Public URLs:
echo Frontend: https://app.mydomain.com
echo Backend:  https://api.mydomain.com
echo.
pause
```

### Chạy script
```powershell
.\start_all.bat
```

---

## BƯỚC 9: KIỂM TRA

### 1. Kiểm tra Backend
```powershell
# Local
curl http://localhost:8000/health

# Public
curl https://api.mydomain.com/health
```

### 2. Kiểm tra Frontend
Mở trình duyệt:
- Local: http://localhost:3000
- Public: https://app.mydomain.com

### 3. Kiểm tra từ máy khác
- Dùng điện thoại hoặc máy khác
- Truy cập: https://app.mydomain.com
- Thử đăng nhập

---

## TROUBLESHOOTING (XỬ LÝ LỖI)

### Lỗi 1: Mixed Content (HTTPS gọi HTTP)
**Triệu chứng**: Console báo "Mixed Content: The page at 'https://...' was loaded over HTTPS..."

**Giải pháp**:
- Đảm bảo file `.env.production` dùng `https://`
- Restart frontend dev server
- Hard refresh trình duyệt (Ctrl+Shift+R)

### Lỗi 2: CORS Error
**Triệu chứng**: Console báo "Access to fetch at '...' has been blocked by CORS policy"

**Giải pháp**:
- Kiểm tra CORS config trong backend có domain của bạn
- Restart backend server
- Xóa cache trình duyệt

### Lỗi 3: 502 Bad Gateway
**Triệu chứng**: Truy cập domain báo lỗi 502

**Giải pháp**:
- Kiểm tra backend/frontend có đang chạy không
- Kiểm tra port mapping đúng chưa
- Kiểm tra tunnel đang chạy

### Lỗi 4: 404 Not Found trên API
**Triệu chứng**: API endpoints trả về 404

**Giải pháp**:
- Kiểm tra route trong Cloudflare Dashboard
- Đảm bảo service URL trỏ đúng port
- Kiểm tra path trong ingress config

### Lỗi 5: Tunnel không kết nối
**Triệu chứng**: `cloudflared tunnel run` báo lỗi

**Giải pháp**:
```powershell
# Kiểm tra tunnel có tồn tại
cloudflared tunnel list

# Kiểm tra token còn hợp lệ
cloudflared tunnel token [TUNNEL-ID]

# Tạo lại tunnel nếu cần
cloudflared tunnel delete [TUNNEL-ID]
cloudflared tunnel create my-new-tunnel
```

---

## TIPS & BEST PRACTICES

### 1. Quản lý Token an toàn
- **KHÔNG** commit token vào Git
- Lưu token vào file riêng: `.tunnel_token` và add vào `.gitignore`
- Hoặc dùng biến môi trường:
  ```batch
  set TUNNEL_TOKEN=eyJhIjoiN2Q2MTllNGQ...
  cloudflared tunnel run --token %TUNNEL_TOKEN%
  ```

### 2. Production vs Development
Tạo 2 file env riêng:

**`.env.development`** (cho local):
```env
VITE_API_BASE_URL=http://localhost:8000/api/v1
```

**`.env.production`** (cho public):
```env
VITE_API_BASE_URL=https://api.mydomain.com/api/v1
```

### 3. Tự động khởi động Tunnel
Tạo Windows Service hoặc dùng Task Scheduler để tunnel tự chạy khi khởi động máy.

### 4. Monitoring
Kiểm tra tunnel status:
```powershell
cloudflared tunnel info [TUNNEL-ID]
```

Xem logs:
```powershell
# Tunnel logs
cloudflared tunnel run --token [TOKEN] --loglevel debug

# Hoặc trong config
cloudflared tunnel --config config.yml run --loglevel debug
```

### 5. Multiple Environments
Tạo nhiều tunnels cho các môi trường khác nhau:
```powershell
cloudflared tunnel create my-app-dev
cloudflared tunnel create my-app-staging
cloudflared tunnel create my-app-prod
```

---

## TÓM TẮT NHANH

```powershell
# 1. Cài đặt
winget install --id Cloudflare.cloudflared

# 2. Đăng nhập
cloudflared tunnel login

# 3. Tạo tunnel
cloudflared tunnel create my-app

# 4. Lấy token
cloudflared tunnel token my-app

# 5. Cấu hình DNS qua dashboard
# https://one.dash.cloudflare.com/ → Networks → Tunnels

# 6. Sửa .env.production
VITE_API_BASE_URL=https://api.mydomain.com/api/v1

# 7. Sửa CORS trong backend
# Thêm domain của bạn vào allowed origins

# 8. Chạy tất cả
# - Backend: python -m uvicorn app.main:app --reload
# - Tunnel: cloudflared tunnel run --token [TOKEN]
# - Frontend: npm run dev

# 9. Truy cập
# https://app.mydomain.com
```

---

## KẾT LUẬN

Sau khi setup xong, bạn có:
- ✅ Backend chạy local nhưng access qua `https://api.mydomain.com`
- ✅ Frontend chạy local nhưng access qua `https://app.mydomain.com`
- ✅ Không cần mở port router
- ✅ Không cần VPS/hosting
- ✅ HTTPS miễn phí
- ✅ Máy nào cũng truy cập được

**Lưu ý**: Đây là giải pháp development/demo. Với production thực sự, nên deploy lên server/cloud.

---

## TÀI LIỆU THAM KHẢO

- Cloudflare Tunnel Docs: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/
- Cloudflared GitHub: https://github.com/cloudflare/cloudflared
- Cloudflare Dashboard: https://dash.cloudflare.com/
- Zero Trust Dashboard: https://one.dash.cloudflare.com/
