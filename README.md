# Báo cáo hệ thống Automation — ZaloCRM + n8n
## Bài thuyết trình kỹ thuật — Ngày 25/04/2026

---

## PHẦN MỞ ĐẦU — Định vị bài trình bày

> **Cách nói đúng với hội đồng:**
> "Core ZaloCRM đã có UI và backend chạy được trên dev. Automation n8n đã được thiết kế, lint/smoke-test đạt yêu cầu kỹ thuật. Chúng ta đang ở giai đoạn cuối chuẩn bị vận hành — cần hoàn tất provider keys, n8n credentials, Zalo/FB/Google integrations trước khi chạy shadow 7 ngày, sau đó mới bật production."

### Trạng thái hiện tại của hệ thống (tính đến hôm nay)

| Thành phần | URL local | Trạng thái |
|---|---|---|
| ZaloCRM UI + Backend | http://localhost:3080 | ✅ Đang chạy — HTTP 200 |
| n8n Workflow Engine | http://localhost:5678 | ✅ Đang chạy — HTTP 200 |
| LiteLLM AI Gateway | http://localhost:4000 | ✅ Đang chạy — HTTP 200 |
| Uptime Kuma (Monitoring) | http://localhost:3001 | ✅ Đang chạy — HTTP 200 |

**Tóm tắt tiến độ:** 15 vòng Codex review đã hoàn tất. `npm run build` pass. Smoke-test 3 stage pass. 9 workflow JSON lint sạch. Chờ điền secrets và import credentials để bắt đầu shadow test.

---

## PHẦN 1 — Tổng quan kiến trúc hệ thống

### 1.1 Sơ đồ stack công nghệ

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXTERNAL CHANNELS                            │
│  [Zalo OA] [Facebook Messenger] [TikTok] [RSS Feeds]           │
└─────────────────┬───────────────┬───────────────────────────────┘
                  │               │
                  ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ZaloCRM (Core CRM)                           │
│  Backend: Node.js + TypeScript + Prisma + PostgreSQL 16        │
│  Frontend: React + Vite (port 3080)                             │
│  Redis: Rate limiter, Session, Cache (automation-redis)         │
└─────────────────┬───────────────────────────────────────────────┘
                  │ Webhook events (HMAC-SHA256)
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    n8n Automation Engine (port 5678)            │
│  9 workflows xử lý: CSKH bot, Content pipeline, Monitoring     │
└─────────────────┬───────────────────────────────────────────────┘
                  │ OpenAI-compatible API
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│               LiteLLM Gateway (port 4000)                       │
│  Multi-provider: Kimi Moonshot / Gemini / Qwen / Anthropic      │
│  Cost-first routing với fallback tự động                        │
└─────────────────────────────────────────────────────────────────┘
                  │ Metrics
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│        Monitoring Stack                                         │
│  Prometheus + Grafana (port 3000) + Uptime Kuma (port 3001)    │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Database schema — Phân vùng rõ ràng

```
PostgreSQL instance (zalocrm DB)
├── public schema        → ZaloCRM Prisma models (contacts, conversations, messages...)
├── content schema       → Automation content pipeline
│   ├── items            → Bài viết thô từ RSS/scraper
│   ├── drafts           → Bản nháp đã rewrite theo từng platform
│   ├── publish_log      → Lịch sử publish, kết quả, external_id
│   ├── metrics          → KPI theo ngày/platform
│   └── prompt_experiments → A/B test prompts
└── omni schema          → CSKH omni-channel
    ├── external_messages → Tin nhắn FB/TikTok/Zalo thô
    └── intent_log        → Kết quả phân loại AI + hành động + verdict nhân viên

n8n DB     → n8n internal (workflow definitions, executions)
litellm DB → LiteLLM spend logs, virtual keys
```

---

## PHẦN 2 — LiteLLM Gateway: Trung tâm điều phối AI

### 2.1 Mô hình hoạt động

LiteLLM đóng vai trò **single OpenAI-compatible endpoint** cho toàn bộ hệ thống. Các workflow n8n không gọi trực tiếp OpenAI/Gemini — tất cả đi qua `http://litellm:4000/v1/chat/completions`.

**Lợi ích:**
- Đổi model mà không sửa n8n workflow
- Cost-first routing: tự động chọn model rẻ nhất đủ đáp ứng
- Fallback tự động khi provider down
- Ghi log chi phí theo model/request → báo cáo tháng tự động (WF09)

### 2.2 Cấu trúc 4 tầng model

| Tầng | Model name | Provider | Dùng cho | Chi phí ước tính |
|---|---|---|---|---|
| **cheap-vi** | `cheap-vi` | Kimi moonshot-v1-8k | Intent CSKH, classify, summary | ~$0.60/1M token in |
| ↳ fallback | `cheap-vi-fallback` | Gemini 1.5 Flash | Backup khi Kimi down | Miễn phí 15 RPM |
| ↳ third | `cheap-vi-third` | Qwen Plus (Alibaba) | Backup thứ 3 | ~$0.80/1M |
| **rewrite-vi** | `rewrite-vi` | Kimi moonshot-v1-32k | Viết lại bài content | ~$1.20/1M token in |
| **guardrail** | `guardrail` | Gemini 2.0 Flash | Kiểm duyệt nội dung | Rất thấp |
| **premium-vi** | `premium-vi` | Anthropic Claude / OpenAI | Khi cần chất lượng cao | Theo yêu cầu |

### 2.3 Cơ chế bảo mật

- n8n sử dụng **LiteLLM virtual keys** (không phải API keys gốc)
- 2 virtual keys độc lập: `LITELLM_KEY_CONTENT` (content pipeline) và `LITELLM_KEY_CSKH` (CSKH bot)
- Có thể revoke từng key mà không ảnh hưởng key còn lại
- Spend cap theo từng key → kiểm soát chi phí theo team

---

## PHẦN 3 — 9 Workflows n8n: Thiết kế chi tiết

### Tổng quan 9 workflows

```
CONTENT PIPELINE (tự động hóa nội dung)
  WF01 → Ingest RSS          (mỗi 15 phút)
  WF06 → Fan-out rewrite     (mỗi 10 phút)   ←— thường dùng
  WF08 → Batch rewrite       (mỗi 20 phút)   ←— tiết kiệm chi phí
  WF03 → Review Sync Sheets  (mỗi 5 phút)
  WF02 → Publish Scheduler   (mỗi 5 phút)

CSKH BOT (chăm sóc khách hàng)
  WF04 → Zalo Intent Bot     (webhook realtime)
  WF05 → FB Messenger Bot    (webhook realtime)

MONITORING & REPORTING
  WF07 → SLA Watchdog        (mỗi 30 phút)
  WF09 → Monthly Cost Report (ngày 1 hàng tháng 09:00)
  WF99 → Error Alert         (trigger khi workflow lỗi)
```

---

### WF01 — Content Ingest + Guardrail v2

**Mục đích:** Thu thập bài viết từ RSS feeds, kiểm duyệt bằng AI, lưu vào database.

**Trigger:** Tự động mỗi 15 phút

**Luồng xử lý (8 bước):**

```
[Cron 15m]
    ↓
[Feed Catalogue] — Danh sách RSS: Tuổi Trẻ kinh doanh, VNExpress Business, CafeF chứng khoán
    ↓
[RSS Read] — Đọc từng feed, trả về danh sách articles
    ↓
[Normalize + Hash] — Chuẩn hóa text, tạo SHA-256 deduplication hash
    ↓
[Has title + body?] — Bỏ qua bài thiếu tiêu đề hoặc nội dung
    ↓
[Upsert content.items] — Lưu vào DB với ON CONFLICT DO UPDATE (idempotent)
    ↓
[Only new items] — Chỉ xử lý bài mới thực sự (RETURNING id)
    ↓
[Guardrail filter] → LiteLLM (model: guardrail / Gemini 2.0 Flash)
    ↓               Prompt: "Đánh giá bài có phù hợp đăng fanpage không?"
    ↓               Output JSON: { keep, reason, tone, risk, brandFit }
    ↓
[Approved?] → YES → Mark status = 'enriched' (sẵn sàng cho WF06/WF08)
              NO  → Mark status = 'filtered' + ghi lý do
```

**State machine của content item:**
```
ingested → (guardrail reject) → filtered
ingested → (guardrail approve) → enriched → (WF06) → fanning_out → drafted
                                          → (WF08) → batch_rewriting → drafted
```

**Kỹ thuật đáng chú ý:**
- **Không dùng SplitInBatches** — tránh bug loop n8n khi fan-out 1→N. Feeds được iterate natively qua paired-item tracking.
- `ON CONFLICT DO UPDATE only when status='ingested'` — chỉ update bài đang bị kẹt ở ingested, không ghi đè bài đã enriched.
- Guardrail dùng `response_format: json_object` — đảm bảo LLM luôn trả JSON parse được.

---

### WF06 — Content Fan-out Multi-platform v3

**Mục đích:** Viết lại 1 bài gốc thành 3 phiên bản khác nhau cho Facebook, TikTok, LinkedIn — mỗi platform có ngữ điệu và format riêng.

**Trigger:** Tự động mỗi 10 phút

**Luồng xử lý:**

```
[Cron 10m]
    ↓
[Pick enriched items] — PostgreSQL: SELECT với FOR UPDATE SKIP LOCKED
    ↓                   → Chỉ lấy 1 item, lock ngay tránh race condition
    ↓
[Plan per-platform] — Tạo 3 task: facebook / tiktok / linkedin
    ↓                  Mỗi task có spec riêng:
    ↓                  • Facebook: "700-1200 ký tự, 1 hook + 3-5 hashtag"
    ↓                  • TikTok: "≤150 ký tự + 5 hashtag trending. Hook 1 câu"
    ↓                  • LinkedIn: "≤1500 chars, professional tone, 3 hashtags" (EN)
    ↓
[Rewrite per platform] → LiteLLM (model: rewrite-vi / Kimi moonshot-v1-32k)
    ↓                    Output JSON: { body, hashtags }
    ↓
[content_qa (OpenClaw)] — QA kiểm tra policy, plagiarism, risk
    ↓                      (graceful skip nếu OPENCLAW_API_KEY chưa có)
    ↓
[Shape draft] — Quyết định status: pending_review hoặc rejected
    ↓
[Upsert content.drafts] — ON CONFLICT DO UPDATE RETURNING id
    ↓
[Push to Google Sheets Review Queue] — Editor thấy bài để duyệt
    ↓
[Mark item drafted] — Cập nhật status item khi đủ N platform drafts
```

**Điểm kỹ thuật:**
- `FOR UPDATE SKIP LOCKED` — nhiều worker chạy song song an toàn, không bao giờ xử lý cùng một item
- `plannedCount` được nhúng vào mỗi task — downstream biết khi nào fan-out hoàn tất mà không cần đọc lại node state
- Zalo OA **không có trong danh sách** — Zalo Business API yêu cầu template được duyệt trước; chờ Phase 2

---

### WF08 — Batch Rewrite Fan-in (Cost Saver)

**Mục đích:** Thay thế WF06 khi cần tiết kiệm chi phí — dồn 5 bài vào 1 LLM call thay vì 5 call.

**Trigger:** Tự động mỗi 20 phút

**So sánh WF06 vs WF08:**

| | WF06 (Fan-out) | WF08 (Batch) |
|---|---|---|
| Gọi LLM per item | 3 calls (1 per platform) | 1 call cho 5 items |
| Output | Facebook + TikTok + LinkedIn | Facebook only |
| Chất lượng | Cao hơn (context đầy đủ) | Ổn định, có thể thiếu sót nhỏ |
| Chi phí | Cao hơn ~3x | Tiết kiệm ~70% |
| Dùng khi | Volume thấp, chất lượng ưu tiên | Volume cao, chi phí ưu tiên |

**Luồng xử lý:**

```
[Cron 20m]
    ↓
[Pick up to 5 items] — FOR UPDATE SKIP LOCKED, LIMIT 5
    ↓                   Status: enriched → batch_rewriting
    ↓
[Pack batch] — Gói 5 bài vào 1 payload JSON với index
    ↓
[Single LLM call] → LiteLLM (model: rewrite-vi)
    ↓               Một prompt xử lý 5 bài cùng lúc
    ↓               Output: { results: [{idx, facebook: {body, hashtags}}] }
    ↓
[Explode to drafts] — Parse JSON, validate cardinality, dedup by idx
    ↓                  Bỏ qua idx trùng, bỏ qua bài LLM trả body rỗng
    ↓
[Save drafts] — ON CONFLICT DO UPDATE RETURNING (đảm bảo emit dù conflict)
    ↓
[Mark items drafted] — Chỉ mark drafted nếu EXISTS draft + queued_at IS NOT NULL
```

**Xử lý edge case:** Nếu LLM trả về ít kết quả hơn batch size, các item thiếu giữ status `batch_rewriting`. WF07 watchdog phát hiện sau 30 phút và cảnh báo on-call.

---

### WF03 — Google Sheets Review Sync v1

**Mục đích:** Đồng bộ quyết định duyệt/từ chối của editorial team từ Google Sheets về database, sau đó kích hoạt publish.

**Trigger:** Tự động mỗi 5 phút

**Luồng xử lý:**

```
[Cron 5m]
    ↓
[Read pending rows] — Google Sheets (Review Queue sheet)
    ↓                  Filter: status = 'pending_review'
    ↓
[Filter actionable] — Chỉ lấy row đã có verdict: approve / reject / edit
    ↓
[Route verdict] ────┬── approve → UPDATE drafts SET status='approved'
                    ├── reject  → UPDATE drafts SET status='rejected'
                    └── edit    → UPDATE drafts SET body=new_content, status='approved'
    ↓
[Update Google Sheets] — Đánh dấu row đã xử lý trong sheet
```

**Workflow này là cầu nối con người — AI:** Editorial team không cần vào database, chỉ cần sửa cột "verdict" trong Google Sheets là hệ thống tự cập nhật.

---

### WF02 — Content Publish Scheduler v1

**Mục đích:** Kiểm tra mỗi 5 phút xem có draft nào đã được duyệt và đến giờ publish không, sau đó đẩy lên từng platform.

**Trigger:** Tự động mỗi 5 phút

**Luồng xử lý:**

```
[Cron 5m]
    ↓
[Fetch due drafts] — PostgreSQL: SELECT WHERE status='approved' AND scheduled_at <= now()
    ↓                FOR UPDATE SKIP LOCKED, LIMIT 1
    ↓
[Route by platform] ────┬── facebook → POST /graph.facebook.com/v20.0/{PAGE_ID}/feed
                        │              retry 3 lần, wait 5s
                        ├── zalo_oa  → Stub: "MANUAL_BROADCAST_REQUIRED"
                        │              (Zalo Business API chưa hỗ trợ broadcast v1)
                        ├── tiktok   → OpenClaw skill tiktok_post (Phase 2.2)
                        └── other    → "UNSUPPORTED_PLATFORM" error
    ↓
[Normalize response] — Phân loại outcome:
    ↓                  • success → cập nhật published + lưu external_id
    ↓                  • manual_required → không tính là lỗi (không bóp metrics)
    ↓                  • failed → alert on-call
    ↓
[Log result] — INSERT INTO content.publish_log với đầy đủ thông tin
```

**Lưu ý Zalo OA:** Zalo Business API yêu cầu pre-approved broadcast template. V1 thiết kế cẩn thận: không để draft bị kẹt mãi ở trạng thái `scheduled`, mà chuyển sang `manual_required` và ping editor qua Telegram để broadcast thủ công qua Zalo Business Dashboard.

---

### WF04 — CSKH Zalo Intent Bot v1

**Mục đích:** Bot CSKH thông minh cho Zalo — tự động phân loại intent tin nhắn, tạo draft reply, quyết định auto-reply hoặc chuyển nhân viên.

**Trigger:** Webhook realtime từ ZaloCRM (mỗi tin nhắn Zalo đến)

**Luồng xử lý (đầy đủ):**

```
[ZaloCRM Webhook] — POST /webhook/zalocrm/events
    ↓
[Verify signature] — HMAC-SHA256 với ZALOCRM_WEBHOOK_SECRET
    ↓                timingSafeEqual (chống timing attack)
    ↓                Throw ngay nếu chữ ký sai
    ↓
[Ack 200 ngay] — Trả HTTP 200 về ZaloCRM trong 10s deadline
    ↓             (song song với xử lý AI — không block nhau)
    ↓
[Only inbound contact messages] — Lọc: chỉ xử lý event='message.received' + senderType='contact'
    ↓
[Normalize + hash] — Chuẩn hóa text Vietnamese (NFC, lowercase, trim)
    ↓                  Hash SHA-256 để cache intent
    ↓
[Intent cache GET] — Redis: key = "intent:{orgId}:{messageHash}"
    ↓
[Cache hit?] ──── YES ──→ Dùng kết quả cache (tiết kiệm ~$0.0006/message)
              |
              └── NO ──→ [Classify intent] → LiteLLM (model: cheap-vi / Kimi moonshot-v1-8k)
                          Prompt: "Phân loại tin nhắn CSKH"
                          Labels: greeting | faq_price | faq_shipping | faq_product |
                                  order_status | complaint | lead_qualified | spam | other
                          Output JSON: { intent, confidence, urgency, leadScore, language,
                                         needsHuman, reason }
    ↓
[Parse intent] — Merge cache hoặc LLM result
    ↓
[Intent cache SET] — Redis TTL 3600s (1 giờ)
    ↓
[Compose reply] → LiteLLM (cheap-vi)
    ↓              Tạo draft reply phù hợp với intent
    ↓
[Shadow mode?] ──── SHADOW=true ──→ Log only → omni.intent_log (action_taken='shadow')
               |                     Không gửi cho khách hàng
               └─── SHADOW=false ─┬─ needsHuman=false → Auto-reply qua ZaloCRM API
                                  └─ needsHuman=true  → Tạo task handoff cho nhân viên
    ↓
[Log to omni.intent_log] — Ghi đầy đủ: intent, confidence, urgency, leadScore,
                            action_taken, draft, tokens dùng, from_cache
```

**Phân loại intent (9 nhãn):**

| Intent | Xử lý | Ví dụ |
|---|---|---|
| `greeting` | Auto-reply | "Xin chào", "Hi" |
| `faq_price` | Auto-reply | "Giá sản phẩm bao nhiêu?" |
| `faq_shipping` | Auto-reply | "Ship bao lâu?" |
| `faq_product` | Auto-reply | "Sản phẩm có màu đen không?" |
| `order_status` | Handoff | "Đơn hàng của tôi đâu?" |
| `complaint` | **Handoff ngay** | "Hàng bị lỗi rồi!" |
| `lead_qualified` | Handoff + Tag | Khách hỏi mua số lượng lớn |
| `spam` | Bỏ qua | |
| `other` | Handoff | Không rõ ý định |

**Shadow Mode — cơ chế an toàn trước khi go-live:**
- `CSKH_SHADOW_MODE=true` → bot chạy nhưng KHÔNG gửi reply thật
- Mọi classification + draft đều lưu vào `omni.intent_log`
- Nhân viên chấm điểm mỗi ngày: `approved / edited / rejected`
- Go-live gate: ≥200 graded rows, precision ≥85%, complaint handoff = 0 sai trong 7 ngày liên tiếp

---

### WF05 — CSKH FB Messenger v1

**Mục đích:** Bot CSKH cho Facebook Messenger — tương tự WF04 nhưng theo đặc thù Meta Webhook Protocol.

**Trigger:** Webhook realtime từ Meta Platform

**Đặc điểm kỹ thuật so với WF04:**

```
[GET /webhook/meta/messenger] — Meta webhook verification (hub.challenge)
    ↓ verify hub.verify_token với FB_VERIFY_TOKEN
    ↓ Echo hub.challenge về Meta

[POST /webhook/meta/messenger] — Events thực tế
    ↓
[Verify signature] — X-Hub-Signature-256 với FB_APP_SECRET
    ↓                 PHẢI verify TRƯỚC KHI ack (khác WF04)
    ↓                 Throw 'missing_signature' ngay nếu FB_APP_SECRET rỗng
    ↓
[Ack 200 ngay] — Meta yêu cầu ack trong 20s
    ↓
[Parse events] — Flatten events array từ Meta format phức tạp
    ↓             Bỏ qua echo messages (is_echo=true)
    ↓
→ Tiếp tục pipeline tương tự WF04:
  classify → cache → shadow/live → log to omni.intent_log
```

**Ghi chú bảo mật:** Meta bắt buộc verify `X-Hub-Signature-256` trước khi process. WF05 throw error ngay nếu `FB_APP_SECRET` rỗng — không có cách nào "chạy mà không có secret".

---

### WF07 — Editorial SLA + Fan-out Watchdog (30m)

**Mục đích:** Cảnh báo 2 loại vấn đề: (A) bài viết chờ duyệt quá lâu, (B) content items bị kẹt trong trạng thái xử lý.

**Trigger:** Tự động mỗi 30 phút

**Luồng xử lý:**

```
[Cron 30m]
    ↓
[Fetch SLA alerts] — 1 SQL query, 2 CTE:
    ↓
    ├── CTE "overdue": drafts.status='pending_review' + queued_at IS NOT NULL
    │                  + created_at < now() - 2 hours
    │                  → Bài chờ duyệt quá 2 tiếng → vi phạm SLA editorial
    │
    └── CTE "stuck": items.status IN ('fanning_out','batch_rewriting')
                     + locked_at < now() - 30 minutes
                     → Item bị kẹt trong trạng thái xử lý > 30 phút
                     → Có thể WF06/WF08 crashed giữa chừng
    ↓
[Summarize] — Code node: format message Telegram
    ↓          Bỏ qua nếu cả 2 bucket đều rỗng (không spam khi OK)
    ↓
[Telegram breach alert] → Send đến TELEGRAM_ALERT_CHAT_ID
    ↓   Message: "3 drafts > 2h | 1 item stuck 45m | Requeue SQL: ..."
    ↓   Kèm link Google Sheets Review Queue để editor vào duyệt ngay
```

**Requeue command cho on-call:**
```sql
UPDATE content.items SET status='enriched', locked_at=NULL WHERE id='...'
```
Dùng khi item bị stuck — đưa về enriched để WF06/WF08 pick lại.

---

### WF09 — Monthly Cost Review

**Mục đích:** Tự động tổng hợp chi phí vận hành hàng tháng và gửi báo cáo qua Telegram.

**Trigger:** Ngày 1 hàng tháng, 09:00

**Báo cáo gồm 3 phần:**

```
[1st of month 09:00]
    ↓
[LiteLLM spend by model] — Query LiteLLM_SpendLogs DB
    ↓   Output: model | requests | spend_usd | avg_cost_per_req
    ↓   Tháng trước, top 20 model
    ↓
[Publish last month] — Query content.publish_log
    ↓   Output: platform | automated_attempts | successes | failures | manual_required
    ↓   Lưu ý: manual_required KHÔNG bị tính vào failures (để metrics sạch)
    ↓
[CSKH last month] — Query omni.intent_log
    ↓   Output: intent | messages | leads | handoffs | auto_replies | staff_approved | staff_rejected
    ↓
[Compose report] — Format Markdown report
    ↓
[Telegram] → TELEGRAM_SALES_CHAT_ID (gửi báo cáo cho Sales/Management)
```

**Ví dụ output:**
```
Monthly automation review

Total LLM spend: $12.47

model           requests  spend_usd  avg_cost_per_req
cheap-vi        4,821     $3.12      $0.000647
rewrite-vi      892       $7.23      $0.008107
guardrail       1,205     $2.12      $0.001759

Publish outcomes:
platform    auto_attempts  successes  failures  manual_required
facebook    312            298        4         10
tiktok      89             81         3         5
```

---

### WF99 — Error Alert (Shared)

**Mục đích:** Bắt lỗi từ **tất cả** workflow khác, gửi cảnh báo ngay qua Telegram.

**Trigger:** Bất kỳ workflow nào fail → n8n tự kích hoạt WF99

**Cấu hình:**
```json
"settings": { "errorWorkflow": "99-error-alert" }
```
Tất cả 8 workflow còn lại đều có config này.

**Luồng:**
```
[Error Trigger] — n8n gọi tự động khi workflow lỗi
    ↓   Payload: { workflow.name, execution.id, execution.lastNodeExecuted, execution.error.message }
    ↓
[Telegram alert] → TELEGRAM_ALERT_CHAT_ID
    Message: "n8n workflow failed | Workflow: WF04 | Node: Classify intent | Error: ..."
```

**Lưu ý:** WF99 dùng env var `TELEGRAM_BOT_TOKEN` trực tiếp (không phải n8n credential). Nếu token rỗng, WF99 cũng sẽ fail — đây là lý do cần fill Telegram secrets sớm nhất.

---

## PHẦN 4 — Cơ chế Shadow Mode & Go-live

### 4.1 Tại sao cần Shadow Mode?

Bot AI không bao giờ nên gửi reply thật ngay từ đầu. Shadow mode cho phép:
1. Đo độ chính xác trên traffic thực (không phải test data)
2. Nhân viên chấm điểm và cải thiện prompt
3. Phát hiện edge case ngôn ngữ tiếng Việt đặc thù

### 4.2 Quy trình 7 ngày shadow test

```
Ngày 1-7:
  09:00 — Chạy shadow report:
          psql -U crmuser -d zalocrm < shadow-report.sql
  Liên tục — Nhân viên grade: approved / edited / rejected
  18:00 — Check metrics, dừng nếu precision < 80%

Go-live gate (tất cả phải PASS):
  ✅ ≥ 200 graded rows trong 7 ngày
  ✅ Overall precision (approved + edited) ≥ 85%
  ✅ Exact-match precision (approved only) ≥ 60%
  ✅ Per-intent precision (mỗi bucket) ≥ 75%
  ✅ Complaint handoff false-negative = 0 trong 7 ngày
  ✅ Tất cả known regressions đã cleared
```

### 4.3 Flip switch sang live

```bash
# Trong automation/.env:
CSKH_SHADOW_MODE=false

# Restart n8n để load env mới:
docker compose restart n8n
```

---

## PHẦN 5 — Roadmap và bước tiếp theo

### 5.1 Trạng thái hiện tại (Phase 1.5)

```
[DONE] Phase 1.0 — Core CRM
  ✅ ZaloCRM UI + Backend (contacts, conversations, messages)
  ✅ Zalo account management
  ✅ Webhook system với HMAC-SHA256

[DONE] Phase 1.3 — Automation Design
  ✅ 9 n8n workflows thiết kế hoàn chỉnh
  ✅ LiteLLM gateway cấu hình multi-provider
  ✅ Database schema content + omni
  ✅ Monitoring stack (Prometheus + Grafana + Uptime Kuma)
  ✅ Shadow mode infrastructure

[PENDING] Phase 1.6 — Setup & Shadow
  ⏳ Fill provider API keys (KIMI, GEMINI tối thiểu)
  ⏳ Tạo LiteLLM virtual keys
  ⏳ Import n8n credentials (Redis, Postgres, Google Sheets, Telegram)
  ⏳ Fix webhook URL inter-container
  ⏳ Scan Zalo account
  ⏳ Chạy shadow 7 ngày
```

### 5.2 Lộ trình phát triển

```
Phase 2.0 — Go-live (sau shadow test pass)
  → Bật CSKH_SHADOW_MODE=false
  → WF04/WF05 gửi reply thật
  → Monitor 2 tuần đầu

Phase 2.1 — Facebook & Google integrations
  → Điền FB credentials (App Secret, Page Token)
  → Setup Google Service Account + Review Queue Sheet
  → WF05 + WF03/WF06 hoạt động đầy đủ

Phase 2.2 — TikTok
  → Đăng ký TikTok Business API
  → WF02 TikTok path thay vì OpenClaw stub

Phase 3.0 — Scale
  → Redis-based rate limiter (đã implement, chờ deploy)
  → LeadActivity table split (50K+ contacts)
  → Multiple Zalo accounts
  → Multi-tenant organization
```

---

## PHẦN 6 — Kịch bản demo an toàn

### Thứ tự trình bày khuyến nghị

**Bước 1: ZaloCRM UI** — http://localhost:3080
- Mở dashboard, giới thiệu quản lý hội thoại
- Mở màn hình contacts, conversation timeline
- Giới thiệu Zalo account management

**Bước 2: n8n Workflows** — http://localhost:5678
- Import workflows từ `H:\Task1SaoViet\automation\n8n\workflows\` (nếu chưa có)
- Mở **WF04** — giải thích luồng CSKH bot từ webhook đến AI classify đến log
- Mở **WF01** — giải thích luồng content từ RSS đến guardrail đến database
- Mở **WF06** — giải thích fan-out 1 bài → 3 platform
- Mở **WF07** — giải thích SLA watchdog

**Bước 3: LiteLLM** — http://localhost:4000
- Trình bày cấu trúc 4 model tier, cost-first routing
- Nói: "Đây là gateway AI — khi có key sẽ kích hoạt ngay"

**Bước 4: Uptime Kuma** — http://localhost:3001
- Trình bày monitoring dashboard
- Giải thích 4 service đang được theo dõi

**Bước 5: Roadmap**
- Trình bày Phase 1.6 → 2.0 → 2.1

### Câu nói kết bài

> "Dự án đã qua giai đoạn xây dựng core CRM và thiết kế automation workflow. Hiện dev stack mở được UI ZaloCRM, n8n, LiteLLM, monitoring đầy đủ. Phần automation đã có 9 workflow và smoke-test đạt yêu cầu kỹ thuật. Bước tiếp theo là setup secrets, import credentials, chạy shadow 7 ngày với threshold precision ≥85%, sau đó mới bật production. Dự kiến pilot nội bộ trong 2 tuần tới."

---

## PHỤ LỤC — Câu hỏi thường gặp

**Q: Tại sao chọn n8n thay vì tự code?**
> n8n là workflow engine có visual editor, dễ chỉnh sửa flow mà không cần deploy lại code. Mỗi workflow là 1 JSON file, version control được. Quan trọng hơn: team non-technical có thể hiểu và điều chỉnh logic sau này.

**Q: LiteLLM có thể đổi sang model khác không?**
> Có. Chỉ cần sửa `litellm/config.yaml` và restart container. Workflow n8n không cần thay đổi gì — chúng chỉ gọi đến model name như `cheap-vi` hay `rewrite-vi`.

**Q: Nếu provider AI down thì sao?**
> LiteLLM có fallback chain: Kimi → Gemini → Qwen. Nếu cả 3 down thì workflow fail và WF99 alert qua Telegram ngay.

**Q: Dữ liệu khách hàng được bảo vệ thế nào?**
> ZaloCRM dùng HMAC-SHA256 cho webhook signature. WF04/WF05 verify signature trước khi process bất kỳ data nào. Redis cache dùng hash của message text (không lưu text gốc). Database chỉ accessible qua internal Docker network.

**Q: Chi phí AI hàng tháng dự kiến bao nhiêu?**
> Với mô hình cheap-vi (Kimi), khoảng $0.60-1.20/1M tokens. Một tin nhắn CSKH tốn khoảng 500-800 tokens → $0.0003-0.001/message. WF09 tự báo cáo chi phí thực tế vào ngày 1 hàng tháng.

---

*Tài liệu này được tạo từ source code thực tế tại `H:\Task1SaoViet\automation\`. Ngày cập nhật: 25/04/2026.*
