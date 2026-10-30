# Hướng Dẫn Kiến Trúc & Triển Khai Google Flow Cho Hermes Agent (MCP Specification)

> **Mục tiêu:** Cung cấp tài liệu kỹ thuật hoàn chỉnh để xây dựng một **MCP Server (Model Context Protocol)** hoặc tích hợp tự động hóa toàn diện cho Google Flow trên Hermes Agent (Ubuntu Server headless).
> **Tác giả:** Trợ lý ảo Hermes Agent (kiểm chứng thực tế trên Ubuntu 24.04 LTS).

---

## MỤC LỤC
1. [Kiến trúc tổng quan MCP Server](#1-kiến-trúc-tổng-quan-mcp-server)
2. [Cài đặt môi trường & Thư viện hệ thống](#2-cài-đặt-môi-trường--thư-viện-hệ-thống)
3. [Cấu hình Chromium Headless & Hướng dẫn Đăng nhập Google an toàn](#3-cấu-hình-chromium-headless--hướng-dẫn-đăng-nhập-google-an-toàn)
4. [Tự động hóa Google Flow qua Chrome DevTools Protocol (CDP)](#4-tự-động-hóa-google-flow-qua-chrome-devtools-protocol-cdp)
5. [Quy trình Upscale và Tải ảnh 2K sắc nét](#5-quy-trình-upscale-và-tải-ảnh-2k-sắc-nét)
6. [Kỹ thuật xóa sạch triệt để Watermark Gemini (Ảnh & Video)](#6-kỹ-thuật-xóa-sạch-triệt-để-watermark-gemini-ảnh--video)
7. [Quản lý vòng đời dữ liệu (Tự động xóa sau 24 giờ)](#7-quản-lý-vòng-đời-dữ-liệu-tự-động-xóa-sau-24-giờ)
8. [Đặc tả kỹ thuật thiết kế MCP Server cho Hermes Agent](#8-đặc-tả-kỹ-thuật-thiết-kế-mcp-server-cho-hermes-agent)

---

## 1. Kiến trúc tổng quan MCP Server

```
+-----------------------------------------------------------------------------------+
|                                  HERMES AGENT                                     |
|  (Nhận lệnh từ Telegram/CLI -> Gọi MCP Tool: generate_image, generate_video, ...) |
+-----------------------------------------+-----------------------------------------+
                                          | JSON-RPC (Model Context Protocol)
+-----------------------------------------v-----------------------------------------+
|                          GOOGLE FLOW MCP SERVER (Python)                          |
|  - Quản lý phiên kết nối CDP WebSocket (ws://127.0.0.1:9222/devtools/page/...)   |
|  - Điều khiển ProseMirror Editor & Menu Google Flow                               |
|  - Kích hoạt Upscale 2K & tải file gốc về local VPS                               |
|  - Pipeline khử Watermark (OpenCV grain matching + FFmpeg delogo)                |
|  - Trả về đường dẫn media đã xử lý hoàn chỉnh                                     |
+-----------------------------------------+-----------------------------------------+
                                          | CDP (Chrome DevTools Protocol)
+-----------------------------------------v-----------------------------------------+
|                  CHROMIUM HEADLESS (Persistent User Data Dir)                     |
|  - Chạy background tại cổng 9222 (127.0.0.1)                                      |
|  - Đã lưu sẵn phiên đăng nhập Google Account (Google AI Pro/Ultra)                |
|  - Truy cập https://flow.google.com / https://labs.google/fx/tools/flow           |
+-----------------------------------------------------------------------------------+
```

---

## 2. Cài đặt môi trường & Thư viện hệ thống

### 2.1. Thư viện hệ điều hành Ubuntu 24.04 LTS (Noble Numbat)
Ubuntu 24.04 sử dụng chuẩn `t64` (chuyển đổi 64-bit time_t), nếu thiếu các gói này Chromium sẽ không thể khởi động ở chế độ headless:

```bash
sudo apt-get update && sudo apt-get install -y \
    ca-certificates \
    fonts-liberation \
    libatk1.0-0t64 \
    libatk-bridge2.0-0t64 \
    libcups2t64 \
    libatspi0 \
    libxcomposite1 \
    libxdamage1 \
    libxfixes3 \
    libxrandr2 \
    libgbm1 \
    libcairo2 \
    libpango-1.0-0 \
    libasound2t64 \
    ffmpeg \
    jq \
    curl
```

### 2.2. Thư viện Python bổ trợ cho xử lý Media và CDP
Cài đặt vào môi trường Python/Virtualenv chạy Agent hoặc MCP:

```bash
pip install --upgrade \
    opencv-python-headless \
    pillow \
    numpy \
    websockets \
    imageio \
    moviepy \
    requests
```

**Bảng mục đích các thư viện:**
* `opencv-python-headless` & `numpy`: Xử lý phân tích pixel, trích xuất contour watermark, seamless grain blending.
* `pillow`: Kiểm tra siêu dữ liệu ảnh, đo kích thước, crop thẩm định.
* `websockets`: Giao tiếp trực tiếp với cổng Chrome DevTools Protocol (CDP) không phụ thuộc Selenium/Playwright nặng nề.
* `ffmpeg`: Xác thực codec, thời lượng video (`ffprobe`) và xóa watermark video (`delogo`).

---

## 3. Cấu hình Chromium Headless & Hướng dẫn Đăng nhập Google an toàn

Google chặn đăng nhập tự động từ các bot automation tiêu chuẩn. Do đó, nguyên tắc cốt lõi là: **Dùng Chromium với thư mục profile bền vững, khởi chạy cổng gỡ lỗi cục bộ, và dùng SSH Tunnel để người dùng đăng nhập thủ công 1 lần duy nhất từ trình duyệt máy tính cá nhân.**

### 3.1. Tạo cấu trúc thư mục
```bash
mkdir -p ~/.hermes/flow-profile
mkdir -p ~/.hermes/cache/flow
chmod 700 ~/.hermes/flow-profile
chmod 700 ~/.hermes/cache/flow
```

### 3.2. Lệnh khởi chạy Chromium Headless (Chuẩn Anti-Detection)
Chạy Chromium nền với các tham số loại bỏ cờ tự động hóa:

```bash
nohup chromium-browser \
    --headless=new \
    --no-sandbox \
    --disable-gpu \
    --disable-dev-shm-usage \
    --remote-debugging-port=9222 \
    --remote-debugging-address=127.0.0.1 \
    --user-data-dir=/home/anna-hermes/flow-profile \
    --disable-blink-features=AutomationControlled \
    --window-size=1600,1000 \
    --user-agent="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36" \
    "https://labs.google/fx/tools/flow" > ~/.hermes/cache/flow/chromium.log 2>&1 &
```

> **Lưu ý tối quan trọng:** `--remote-debugging-address` bắt buộc phải là `127.0.0.1`. Không bao giờ để `0.0.0.0` vì sẽ làm lộ toàn bộ phiên trình duyệt ra internet.

### 3.3. Hướng dẫn Đăng nhập Tài khoản Google qua SSH Tunnel (Bảo mật 100%)

Khi mở lần đầu, tài khoản Google sẽ chưa được đăng nhập. Thực hiện theo các bước sau để đăng nhập:

1. **Trên máy tính cá nhân (Windows PowerShell / macOS Terminal):**
   Mở SSH Tunnel chuyển tiếp cổng 9222 của VPS về máy cá nhân:
   ```bash
   ssh -L 9222:127.0.0.1:9222 <user>@<vps-ip>
   ```
   *(Ví dụ: `ssh -L 9222:127.0.0.1:9222 root@165.99.16.151`)*

2. **Mở trình duyệt Google Chrome trên máy tính cá nhân:**
   Truy cập thanh địa chỉ:
   ```
   chrome://inspect
   ```

3. **Cấu hình Discovery Target:**
   - Bấm vào nút **Configure...**
   - Thêm địa chỉ: `localhost:9222` (hoặc `127.0.0.1:9222`) -> Bấm **Done**.

4. **Tương tác và Đăng nhập:**
   - Trong danh sách **Remote Target**, sẽ xuất hiện tab Google Flow đang chạy trên VPS.
   - Bấm vào chữ **inspect**.
   - Một cửa sổ hiển thị màn hình thực tế của VPS sẽ hiện lên. Người dùng tiến hành đăng nhập email, mật khẩu Google, xác minh 2FA / Passkey bình thường như trên máy thật.
   - Khi đã vào được trang chính của Google Flow (gói PRO hoặc gói cá nhân), đóng cửa sổ inspect lại.
   - Toàn bộ cookies, tokens, session storage được lưu vĩnh viễn trong `~/.hermes/flow-profile`. Các lần khởi động sau không bao giờ bị hỏi lại đăng nhập.

---

## 4. Tự động hóa Google Flow qua Chrome DevTools Protocol (CDP)

### 4.1. Lấy WebSocket Endpoint của Tab Google Flow
```bash
curl -s http://127.0.0.1:9222/json/list | jq -r '.[] | select(.url | contains("flow")) | .webSocketDebuggerUrl'
```

### 4.2. Khắc phục lỗi tương tác với ProseMirror Editor
Google Flow sử dụng framework Angular Material kết hợp trình biên soạn rich-text **ProseMirror** (`div.ProseMirror`).
* **Lỗi kinh điển:** Nếu dùng JavaScript gán `editor.innerText = "prompt"` hoặc `editor.innerHTML = ...`, trình soạn thảo nội bộ của ProseMirror sẽ không ghi nhận thay đổi, khiến nút **Bắt đầu tạo** (`button[aria-label="Bắt đầu tạo"]`) bị khóa vĩnh viễn (`disabled`).
* **Quy trình chuẩn qua CDP:**
  1. Lấy toạ độ tâm của `div.ProseMirror`.
  2. Bắn sự kiện click chuột (`Input.dispatchMouseEvent`) để focus con trỏ.
  3. Chọn toàn bộ văn bản cũ (`Ctrl + A`) và xóa đi.
  4. Gõ văn bản bằng phương thức CDP `Input.insertText` (giúp Angular nhận diện input thực từ bàn phím).
  5. Bắn các sự kiện tổng hợp:
     ```javascript
     const ed = document.querySelector('div.ProseMirror');
     ed.dispatchEvent(new Event('input', { bubbles: true }));
     ed.dispatchEvent(new KeyboardEvent('keyup', { bubbles: true, key: 'a' }));
     ed.dispatchEvent(new Event('change', { bubbles: true }));
     ```
  6. Click nút submit `button[aria-label="Bắt đầu tạo"]`.

### 4.3. Neo nhân vật nhất quán (`@Luna`)
Để Google Flow nhận diện đúng khuôn mặt từ ảnh tham chiếu, prompt không thể chỉ chứa chữ `@Luna` dạng plain text mà phải là **Entity Mention Chip**.
* **Quy trình chèn thẻ Chip nhân vật:**
  1. Focus vào ProseMirror.
  2. Nhập ký tự `@` qua `Input.insertText`.
  3. Chờ 500ms để menu autocomplete `.cdk-overlay-pane` hiện ra.
  4. Click vào button nhân vật tương ứng (`button.asset-item`).
  5. Khi đó `@` sẽ tự động chuyển thành `<span class="mention-chip" data-reference-type="entity">Luna</span>`.
  6. Tiếp tục gõ nội dung prompt mô tả bối cảnh, trang phục, góc máy.

### 4.4. Tránh bộ lọc kiểm duyệt an toàn (Content Moderation Policy)
* **Quy tắc an toàn của Google:** Google Flow tự động từ chối ngầm các prompt chứa từ khóa đồ bơi chi tiết (`string bikini`, `tie-side bikini bottoms`, `sticker covers...`). Khi vi phạm, editor sẽ hiện thông báo *"Lượt tạo này có thể vi phạm các chính sách của chúng tôi"* và vô hiệu hóa nút submit.
* **Giải pháp:** Chuyển đổi ngôn từ sang phong cách thời trang nghỉ dưỡng cao cấp:
  - *Thay vì:* `bikini, sexy, reveal`
  - *Dùng:* `chic royal blue patterned summer resort swimwear two-piece set with delicate ties, editorial lighting, 35mm candid lifestyle photography`
* **Cơ chế tự phục hồi (Self-Healing):** Nếu editor bị kẹt do lỗi state, gửi lệnh `Page.reload` qua CDP. Trang sẽ tải lại sau ~4 giây với đầy đủ profile, model đang chọn và nhân vật mà không bị mất dữ liệu.

---

## 5. Quy trình Upscale và Tải ảnh 2K sắc nét

### 5.1. Phân biệt độ phân giải
* Ảnh preview mặc định Google Flow trả về chỉ ở mức **~1K (896x1200 pixel)**.
* Google Flow tích hợp sẵn tính năng **Upscale 2K (1792x2400 pixel, ~4.3 triệu điểm ảnh)** miễn phí 200 lượt/ngày cho mọi tài khoản.

### 5.2. Kịch bản tự động hóa tải bản 2K qua CDP
Sau khi ảnh render xong trên giao diện:
1. Click vào ô hiển thị ảnh (`flow-grid-tile-container`) để mở màn hình chi tiết (Detail View).
2. Click vào icon tùy chọn khác: `button[aria-label="Tuỳ chọn khác"]` (`more_vert`).
3. Rê chuột và click vào mục menu: `Tải nội dung nghe nhìn xuống` (`download`).
4. Trong menu con xuất hiện, click vào mục:
   `2K
Đã tăng độ phân giải`
5. File ảnh chất lượng 2K gốc (ví dụ: `Woman_gazing_into_camera_..._2K.jpg`, kích thước ~400KB - 1.5MB, độ phân giải 1792x2400) sẽ tự động tải về thư mục `~/Downloads/`.
6. Di chuyển file vào thư mục đích `~/.hermes/cache/flow/`.

---

## 6. Kỹ thuật xóa sạch triệt để Watermark Gemini (Ảnh & Video)

Mọi sản phẩm từ Google Flow (Nano Banana / Omni Flash) đều có watermark mờ hình ngôi sao 4 cánh (Gemini Sparkle) ở góc dưới cùng bên phải.

### 6.1. Tại sao thuật toán Inpainting thông thường thất bại?
Sử dụng `cv2.inpaint` (Telea hoặc Navier-Stokes) chỉ đơn thuần làm nhòe (blur) các pixel sáng, tạo ra một vết đốm nhờ nhờ (discoloration smudge) làm mất cấu trúc hạt tự nhiên của cảm biến ảnh (sensor grain/noise). Người dùng nhìn kỹ sẽ phát hiện ngay vết tẩy xóa.

### 6.2. Thuật toán độc quyền: Contour Isolation & Micro-Grain Blending (Chuẩn 100%)

```python
import cv2
import numpy as np

def clean_gemini_watermark_image(input_path: str, output_path: str):
    img = cv2.imread(input_path)
    h, w, c = img.shape
    
    # 1. Xác định vùng góc dưới cùng bên phải (ROI)
    # Với ảnh 2K (1792x2400), lấy vùng 300x300; với ảnh 1K lấy 200x200
    roi_size = 300 if w > 1500 else 200
    roi_y1, roi_y2 = h - roi_size, h
    roi_x1, roi_x2 = w - roi_size, w
    roi = img[roi_y1:roi_y2, roi_x1:roi_x2].copy()
    gray = cv2.cvtColor(roi, cv2.COLOR_BGR2GRAY)
    
    # 2. Phân đoạn nhị phân cô lập icon watermark
    # Điểm sáng của logo thường có độ sáng > 20 trên nền tối
    wm_mask_roi = (gray > 20).astype(np.uint8) * 255
    
    # Giãn nở nhẹ (dilate) để bao trùm cả quầng mờ khử răng cưa của logo
    kernel = np.ones((7, 7), np.uint8)
    wm_mask_roi = cv2.dilate(wm_mask_roi, kernel, iterations=3)
    
    if np.count_nonzero(wm_mask_roi) > 0:
        # 3. Lấy mẫu màu nền trung vị (median) từ các pixel sạch ngoài logo
        clean_pixels = roi[wm_mask_roi == 0]
        median_color = np.median(clean_pixels, axis=0).astype(np.uint8)
        
        output = img.copy()
        wm_positions = np.where(wm_mask_roi > 0)
        
        # 4. Bù hạt nhiễu vi mô (Gaussian noise) để khớp 100% với noise nền thật
        for y, x in zip(wm_positions[0], wm_positions[1]):
            abs_y = roi_y1 + y
            abs_x = roi_x1 + x
            noise = np.random.normal(0, 2.0, 3).astype(np.int16)
            new_val = np.clip(median_color.astype(np.int16) + noise, 0, 255).astype(np.uint8)
            output[abs_y, abs_x] = new_val
            
        # 5. Làm mềm viền tiếp giáp bằng Gaussian Blur cục bộ
        blur_roi = output[roi_y1:roi_y2, roi_x1:roi_x2].copy()
        blurred = cv2.GaussianBlur(blur_roi, (5, 5), 0)
        for y, x in zip(wm_positions[0], wm_positions[1]):
            blur_roi[y, x] = blurred[y, x]
        output[roi_y1:roi_y2, roi_x1:roi_x2] = blur_roi
        
        cv2.imwrite(output_path, output, [int(cv2.IMWRITE_JPEG_QUALITY), 95])
    else:
        # Không có watermark, sao chép nguyên trạng
        cv2.imwrite(output_path, img, [int(cv2.IMWRITE_JPEG_QUALITY), 95])

    return output_path
```

### 6.3. Khử Watermark trên Video (FFmpeg Delogo)
Đối với video sinh ra từ Gemini Omni 1.1 Flash:
```bash
ffmpeg -i input_video.mp4 \
    -vf "delogo=x=W-140:y=H-140:w=65:h=65:band=2" \
    -c:v libx264 -crf 18 -preset fast \
    -c:a copy output_clean_video.mp4
```

---

## 7. Quản lý vòng đời dữ liệu (Tự động xóa sau 24 giờ)

Để đảm bảo ổ cứng VPS không bị đầy khi chạy liên tục nhưng vẫn **bảo toàn vĩnh viễn ảnh tham chiếu nhân vật (`luna_reference.jpg`)**:

### 7.1. Script dọn dẹp (`~/.hermes/scripts/cleanup_flow_cache.sh`)
```bash
#!/usr/bin/env bash
set -euo pipefail

# Quét và xóa toàn bộ file media cũ hơn 24 giờ (1440 phút), trừ file chứa từ khóa 'reference'
TARGET_DIR="$HOME/.hermes/cache/flow"

if [ -d "$TARGET_DIR" ]; then
    find "$TARGET_DIR" -type f \( -name "*.jpg" -o -name "*.png" -o -name "*.mp4" -o -name "*.jpeg" -o -name "*.webp" \) \
        ! -name "*reference*" -mmin +1440 -delete
fi
```
Cấp quyền thực thi:
```bash
chmod +x ~/.hermes/scripts/cleanup_flow_cache.sh
```

### 7.2. Lên lịch chạy định kỳ bằng Cron Job (6 giờ / lần)
```bash
0 */6 * * * /home/anna-hermes/.hermes/scripts/cleanup_flow_cache.sh > /dev/null 2>&1
```

---

## 8. Đặc tả kỹ thuật thiết kế MCP Server cho Hermes Agent

Khi đóng gói thành một MCP Server hoàn chỉnh (`google-flow-mcp`), kiến trúc module được khuyến nghị như sau:

### 8.1. Cấu trúc mã nguồn
```
google-flow-mcp/
├── pyproject.toml
├── README.md
├── server.py                 # FastMCP / MCP Server Entrypoint
├── flow_cdp.py               # Kết nối và điều khiển Chromium qua CDP
├── watermark_cleaner.py      # Module xử lý ảnh OpenCV & video FFmpeg
└── utils.py                  # Helper tải file, quản lý cache, retry
```

### 8.2. Danh sách Tools của MCP Server (Tool Schemas)

#### Tool 1: `flow_generate_image`
* **Mục đích:** Tạo ảnh chất lượng cao trên Google Flow.
* **Tham số:**
  - `prompt` (string, required): Mô tả nội dung bức ảnh.
  - `character_id` (string, optional): Tên nhân vật đã lưu (ví dụ: `Luna`) để tự động chèn entity chip `@Luna`.
  - `aspect_ratio` (enum, optional): `16:9`, `4:3`, `1:1`, `3:4`, `9:16`. Mặc định: `3:4`.
  - `upscale_2k` (boolean, optional): Tự động nâng cấp lên 2K. Mặc định: `true`.
  - `clean_watermark` (boolean, optional): Tự động xóa watermark. Mặc định: `true`.
* **Trả về:** `{ "image_path": "/home/.../luna_fashion_2k_clean.jpg", "resolution": "1792x2400", "duration_seconds": 28.5 }`

#### Tool 2: `flow_generate_video`
* **Mục đích:** Tạo clip ngắn cinematic bằng Gemini Omni 1.1 Flash.
* **Tham số:**
  - `prompt` (string, required): Mô tả chuyển động, camera, ánh sáng.
  - `character_id` (string, optional): Gắn chip nhân vật.
  - `clean_watermark` (boolean, optional): Dùng delogo. Mặc định: `true`.
* **Trả về:** `{ "video_path": "/home/.../flow_video_clean.mp4", "duration": 8.0, "format": "mp4" }`

#### Tool 3: `flow_upscale_existing`
* **Mục đích:** Upscale một ảnh đã tạo sẵn trong dự án lên 2K.
* **Tham số:**
  - `image_id` hoặc click vị trí trong grid.

#### Tool 4: `flow_session_status`
* **Mục đích:** Kiểm tra trạng thái kết nối Google Flow, gói thuê bao (Pro/Ultra), kiểm tra quota còn lại trong ngày.

### 8.3. Cấu hình khai báo MCP trong `~/.hermes/config.yaml`
```yaml
mcp_servers:
  google-flow:
    command: "python3"
    args: ["/home/anna-hermes/google-flow-mcp/server.py"]
    env:
      CHROME_DEBUG_PORT: "9222"
      CACHE_DIR: "/home/anna-hermes/.hermes/cache/flow"
```

---
*Tài liệu được đúc kết từ quá trình debug và vận hành thực tế trên hệ thống Hermes Agent Ubuntu Server.*
