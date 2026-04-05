# 🎬 Image → Multi-Video → Social Media Workflow

Workflow n8n tự động tạo nhiều video từ 1 ảnh và đăng lên **Facebook Reels** + **TikTok**.

## 📋 Kiến trúc Workflow

```
[Webhook] → [Validate] → [SplitInBatches]
                               ↓
                    [RunwayML: Tạo video]
                               ↓
                    [Chờ 45s] → [Poll status]
                               ↓
                    [Video xong?] ──NO──→ [Retry/Timeout]
                        ↓ YES
              ┌──────────┴──────────┐
              ↓                     ↓
    [Facebook Reels]           [TikTok]
              ↓                     ↓
              └──────────┬──────────┘
                         ↓
                [Tổng hợp kết quả]
                         ↓
                   [Response 200]
```

## 🛠️ Yêu cầu & Cấu hình

### 1. RunwayML API Key
- Đăng ký tại: https://runwayml.com
- Vào **Settings > API Keys** → tạo key mới
- Trong n8n: **Credentials > HTTP Header Auth**
  ```
  Name: RunwayML API Key
  Header Name: Authorization
  Header Value: Bearer YOUR_RUNWAY_API_KEY
  ```

### 2. Facebook Page Access Token
Trong n8n, vào **Settings > Variables** và thêm:

| Tên biến | Giá trị |
|----------|---------|
| `FACEBOOK_PAGE_ID` | ID của Facebook Page |
| `FACEBOOK_PAGE_ACCESS_TOKEN` | Page Access Token (không hết hạn) |

**Cách lấy Page Access Token vĩnh cửu:**
1. Vào [Facebook Developers](https://developers.facebook.com)
2. Tạo App → thêm **Pages API**
3. Dùng Graph API Explorer để lấy token
4. Exchange lấy Long-lived token (60 ngày) → exchange tiếp lấy Page token vĩnh cửu

**Quyền cần thiết cho Facebook App:**
- `pages_manage_posts`
- `pages_read_engagement`
- `pages_show_list`
- `publish_video`

### 3. TikTok Access Token
Trong n8n **Settings > Variables**:

| Tên biến | Giá trị |
|----------|---------|
| `TIKTOK_ACCESS_TOKEN` | TikTok OAuth Access Token |

**Cách lấy:**
1. Đăng ký [TikTok for Developers](https://developers.tiktok.com)
2. Tạo App → bật **Content Posting API**
3. Scope cần thiết: `video.publish`, `video.upload`
4. OAuth flow để lấy access token

## 🚀 Cách sử dụng

### Import Workflow vào n8n
1. Mở n8n (thường tại `http://localhost:5678`)
2. Click **"+"** → **"Import from file"**
3. Chọn file `image-to-social-video.json`

### Gọi Workflow

```bash
curl -X POST http://localhost:5678/webhook/image-to-video \
  -H "Content-Type: application/json" \
  -d '{
    "imageUrl": "https://example.com/my-photo.jpg",
    "caption": "Khám phá vẻ đẹp tuyệt vời của thế giới ✨",
    "hashtags": "#travel #photography #AI #Reels #viral",
    "videoPrompts": [
      "Slow cinematic zoom in, warm golden hour light, 4K quality",
      "Dynamic pan from left to right, vibrant saturated colors, energetic",
      "Gentle floating upward movement, soft dreamy bokeh background"
    ]
  }'
```

### Ví dụ với Python

```python
import requests

url = "http://localhost:5678/webhook/image-to-video"

payload = {
    "imageUrl": "https://your-server.com/image.jpg",
    "caption": "Video AI tuyệt đẹp 🎬",
    "hashtags": "#AIVideo #Reels #TikTok",
    "videoPrompts": [
        "Smooth zoom out revealing landscape, cinematic drone shot",
        "Time-lapse style with glowing particles floating",
        "Slow motion water ripples effect, blue tones"
    ]
}

response = requests.post(url, json=payload)
print(response.json())
```

## 📤 Kết quả trả về

```json
{
  "message": "Đã xử lý 3 video thành công!",
  "results": [
    {
      "videoIndex": 1,
      "status": "SUCCESS",
      "videoUrl": "https://cdn.runwayml.com/...",
      "facebookStatus": "PUBLISHED",
      "tiktokPublishId": "v_pub_url~...",
      "timestamp": "2026-04-04T10:00:00.000Z"
    }
  ],
  "completedAt": "2026-04-04T10:05:30.000Z"
}
```

## ⚙️ Tùy chỉnh

### Thay đổi mô hình RunwayML
Trong node **"🎬 Tạo Video (RunwayML)"**, sửa tham số `model`:
- `gen3a_turbo` — Nhanh, 5 giây video (khuyên dùng)
- `gen3a` — Chất lượng cao hơn, 10 giây video

### Thay đổi độ dài video
Sửa tham số `duration`: `5` hoặc `10` (giây)

### Thay đổi tỉ lệ video
Sửa tham số `ratio`:
- `768:1280` — Dọc 9:16 (Reels/TikTok) ✅
- `1280:768` — Ngang 16:9
- `1024:1024` — Vuông 1:1

### Thêm nền tảng khác
Để thêm YouTube Shorts, Instagram Reels, v.v., thêm node HTTP Request sau node **"✔️ Video đã hoàn thành?"**.

## 🔄 Luồng xử lý chi tiết

| Bước | Node | Mô tả |
|------|------|--------|
| 1 | Webhook | Nhận request với imageUrl và danh sách prompts |
| 2 | Validate | Kiểm tra dữ liệu, tạo danh sách video cần làm |
| 3 | SplitInBatches | Xử lý từng video một |
| 4 | RunwayML API | Gửi yêu cầu tạo video |
| 5 | Wait 45s | Chờ RunwayML xử lý ban đầu |
| 6 | Poll Status | Kiểm tra xem video đã xong chưa |
| 7 | IF Node | Nếu SUCCEEDED → đăng; Nếu chưa → retry |
| 8 | Retry Loop | Tối đa 10 lần × 30 giây = 5 phút timeout |
| 9 | Facebook API | Khởi tạo upload + publish Reels |
| 10 | TikTok API | Upload video qua URL (PULL_FROM_URL) |
| 11 | Aggregate | Tổng hợp kết quả tất cả các video |
| 12 | Response | Trả về JSON kết quả cho caller |

## ⚠️ Lưu ý quan trọng

1. **Chi phí RunwayML**: Mỗi video tốn credits. Gen3a Turbo 5s ≈ 50 credits.
2. **Rate limit TikTok**: API TikTok có giới hạn số lần đăng mỗi ngày.
3. **Facebook review**: App Facebook cần được review trước khi publish lên Pages thật.
4. **Thời gian xử lý**: Mỗi video mất 1-5 phút để tạo.
5. **Video URL**: RunwayML trả về URL tạm thời, nên download và lưu trữ nếu cần dùng lâu dài.

## 🐛 Troubleshooting

**Lỗi 401 RunwayML**: Kiểm tra lại API Key trong Credentials.

**Lỗi Facebook "OAuthException"**: Token hết hạn hoặc thiếu quyền `publish_video`.

**TikTok lỗi "access_token_invalid"**: Refresh token đã hết hạn (24h), cần OAuth lại.

**Video không tạo được (FAILED)**: Ảnh không hợp lệ hoặc vi phạm content policy của RunwayML.
