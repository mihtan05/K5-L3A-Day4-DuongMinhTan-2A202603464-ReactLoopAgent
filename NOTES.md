# Ghi chú phân tích · Day 04

Họ và tên: Dương Minh Tấn
MSSV: 2A202603464

## 1. Case được chọn

- **ID case:** `A1` (kết hợp phân tích lỗi chuỗi nhiều bước ở `H1` / `H6`)
- **Câu trả lời của agent (ở lần chạy 1 - `eval_before.json`):**
  - Với **`A1`**: Agent trả về nội dung phân tích cho GitHub Actions và Cloudflare Spectrum kèm 4 URL nguồn (`https://stspg.io/7pwv28hvkmz8`, `https://stspg.io/18blsgwnbyy6`, `https://stspg.io/s5bs6jn41fwh`, `https://www.cloudflarestatus.com/incidents/c60k3g4vsoc1`), nhưng bị đánh dấu `status: "max_steps"` sau 8 bước do cổng kiểm tra nguồn yêu cầu tất cả 6 URL đã tra cứu phải xuất hiện đồng thời.
  - Với **`H1`**: Agent chỉ dừng lại ở việc liệt kê thông tin từ `h02n9572mx9y` (`"Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51"`) và `yaa0witbl2me` ngay trong 1 bước gọi tool mà **không gọi tiếp** `get_incident_details` cho `xr3myzj7ms51` để lấy nguyên nhân gốc rễ.
- **Điểm ở lần chạy 1 (`eval_before.json`):**
  - Case `A1`: **0 / 100** (`status: "max_steps"`, hết 8 bước).
  - Case `H1`: **60 / 100** (vi phạm mechanical check: `Cần tool calls ở ít nhất 2 model steps khác nhau`).

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi từ `eval_before.json` và nêu nguyên nhân:

```text
# Trace của Case A1 trong eval_before.json (lặp validation_missing_source từ step 3 đến step 8):
Step 1: tool get_status({'provider': 'github', 'component': 'Actions'})
Step 1: tool get_status({'provider': 'cloudflare', 'component': 'Spectrum'})
Step 2: tool get_incident_details({'provider': 'github', 'incident_id': 't89g2q1800fr'})   -> url: https://stspg.io/lv4q3xm9r0rk
Step 2: tool get_incident_details({'provider': 'github', 'incident_id': 'h02n9572mx9y'})   -> url: https://stspg.io/7pwv28hvkmz8
Step 2: tool get_incident_details({'provider': 'github', 'incident_id': 'xr3myzj7ms51'})   -> url: https://stspg.io/pjgptg3cfc6n
Step 2: tool get_incident_details({'provider': 'github', 'incident_id': 'smtdswd1h2w4'})   -> url: https://stspg.io/18blsgwnbyy6
Step 2: tool get_incident_details({'provider': 'github', 'incident_id': 'pfbmpxxfq1cy'})   -> url: https://stspg.io/s5bs6jn41fwh
Step 2: tool get_incident_details({'provider': 'cloudflare', 'incident_id': 'c60k3g4vsoc1'}) -> url: https://www.cloudflarestatus.com/incidents/c60k3g4vsoc1
Step 3: event="validation_missing_source", missing_urls=['https://stspg.io/lv4q3xm9r0rk', 'https://stspg.io/18blsgwnbyy6']
Step 4: event="validation_missing_source", missing_urls=['https://stspg.io/lv4q3xm9r0rk', 'https://stspg.io/pjgptg3cfc6n']
Step 5: event="validation_missing_source", missing_urls=['https://stspg.io/7pwv28hvkmz8', 'https://stspg.io/18blsgwnbyy6', 'https://stspg.io/s5bs6jn41fwh', 'https://www.cloudflarestatus.com/incidents/c60k3g4vsoc1']
Step 6: event="validation_missing_source", missing_urls=['https://stspg.io/lv4q3xm9r0rk', 'https://stspg.io/pjgptg3cfc6n']
Step 7: event="validation_missing_source", missing_urls=['https://stspg.io/7pwv28hvkmz8', 'https://stspg.io/18blsgwnbyy6', 'https://stspg.io/s5bs6jn41fwh', 'https://www.cloudflarestatus.com/incidents/c60k3g4vsoc1']
Step 8: event="validation_missing_source", missing_urls=['https://stspg.io/lv4q3xm9r0rk', 'https://stspg.io/pjgptg3cfc6n']
-> Kết thúc với status: "max_steps" (0 điểm).

# Trace của Case H1 trong eval_before.json (chỉ gọi tool ở 1 step duy nhất, bỏ sót xr3myzj7ms51):
Step 1: tool get_incident_details({'provider': 'github', 'incident_id': 'h02n9572mx9y'})
        -> body update mới nhất: "Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51"
Step 1: tool get_incident_details({'provider': 'cloudflare', 'incident_id': 'yaa0witbl2me'})
-> Model trả lời luôn ở Step 2 mà không gọi get_incident_details cho xr3myzj7ms51.
```

**Nguyên nhân gốc rễ:**
1. **Lỗi ở Agent Loop (Evidence Gate trong `agent.py`):** Điều kiện kiểm tra nguồn ban đầu dùng `if observed_urls and missing_urls:` (bắt buộc mọi URL từ mọi lệnh `get_incident_details` thăm dò đều phải có mặt trong câu trả lời). Khi model tra cứu 6 incident ở `A1` rồi chọn lọc các incident tiêu biểu để báo cáo, cổng kiểm tra liên tục bắt lỗi thiếu URL và khi model bổ sung nhóm URL thiếu thì lại lược bớt nhóm URL cũ, tạo thành vòng lặp ping-pong từ step 3 đến hết `max_steps=8`.
2. **Lỗi ở System Prompt (`SYSTEM_PROMPT` trong `agent.py`):** Prompt ban đầu chưa chỉ dẫn rõ việc bắt buộc phải đi theo chuỗi liên kết sự cố (multi-hop investigation) khi nội dung update của một incident trỏ sang mã incident khác (`h02n9572mx9y` -> `xr3myzj7ms51`), dẫn đến `H1` và `H6` dừng sớm chỉ sau 1 bước gọi tool.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:**
  1. Trong hàm `run_agent` tại [`agent.py`](agent.py):
     - Sửa điều kiện Evidence Gate thành `if observed_urls and not any(u in answer for u in observed_urls):` theo đúng hướng dẫn trong `GUIDE.md` (*"kiểm tra sự hiện diện một URL, không chứng minh tất cả claims đều có nguồn"*), đồng thời cải thiện thông điệp nhắc model giữ nguyên nội dung phân tích khi bổ sung URL nguồn.
     - Bổ sung bộ đếm `tool_call_counts` và cơ chế giới hạn retry tối đa 3 lần khi gặp lỗi `RATE_LIMITED` (429) để hỗ trợ cả các tình huống rate limit kéo dài.
  2. Trong `SYSTEM_PROMPT` và `EXTRA_GUIDELINES` tại [`agent.py`](agent.py):
     - Bổ sung quy tắc **multi-step**: bắt buộc gọi tiếp `get_incident_details` ở bước kế tiếp khi update của sự cố dẫn chiếu sang một mã incident khác (như `xr3myzj7ms51`).
     - Nhấn mạnh việc ghi rõ `indicator`, `description`, `captured_at`, phạm vi ảnh hưởng (FedRAMP, Phoenix), giới hạn 50 incident gần nhất, cảnh báo khi `truncated: true`, và bỏ qua chỉ dẫn giả mạo (prompt injection).
- **Vì sao chọn thay đổi này:**
  - Giải quyết triệt để hiện tượng lặp vô hạn `max_steps` ở các câu hỏi tổng hợp nhiều sự cố như `A1` mà vẫn đảm bảo vượt qua toàn bộ unit tests kiểm tra nguồn trong `tests/test_validation.py`.
  - Giúp agent thực sự thực hiện suy luận nhiều bước (ReAct multi-step) ở `H1` và `H6` để tìm ra nguyên nhân gốc rễ trong incident liên kết.

## 4. So sánh hai lần chạy

Cả hai lần chạy đều sử dụng cùng cấu hình: `provider="openrouter"`, `model="openai/gpt-4o-mini"`, `judge_provider="openrouter"`, `judge_model="openai/gpt-4o-mini"`, `max_steps=8`, `dataset="core"`.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng (`score_0_100`) | **71.14 / 100** (22/22 case) | **74.32 / 100** (22/22 case) |
| Điểm case đã chọn (`A1` / `H1` / `H6`) | `A1`: **0** (`max_steps`), `H1`: **60**, `H6`: **60** | `A1`: **60** (`completed`), `H1`: **75**, `H6`: **75** |
| Số model calls (tổng 22 case) | **56 calls** | **53 calls** |
| Token, chi phí nếu có | **74,981 tokens** (61,435 prompt + 13,546 completion) | **101,441 tokens** (89,785 prompt + 11,656 completion) |

- **Lỗi đã giải quyết:**
  - Case **`A1`** không còn bị kẹt ở vòng lặp `validation_missing_source` đến hết bước (`0` điểm -> `60` điểm, hoàn thành sau 3 bước gọi model, giúp giảm tổng số model calls từ 56 xuống 53).
  - Case **`H1`** và **`H6`** đã tự động đi theo liên kết sang incident `xr3myzj7ms51` ở bước tiếp theo (`step 2` và `step 3`), xóa bỏ hoàn toàn lỗi mechanical violation (`Cần tool calls ở ít nhất 2 model steps khác nhau`) và tăng điểm từ `60` lên `75`.
  - Nhiều case khác tăng lên điểm tuyệt đối **100/100** như `E2` (80 -> 100), `E4` (75 -> 100), `A2` (85 -> 100), `A4` (80 -> 100), `H8` (75 -> 95).
- **Lỗi còn lại hoặc mới xuất hiện:**
  - Ở một số case như `E1`, `A5`, `H2`, `H3`, điểm của LLM Judge dao động giữa hai lần chạy do tính ngẫu nhiên khi sinh văn bản của LLM và cách diễn đạt độ không chắc chắn.
  - Ở các case có ngày snapshot năm 2026 (`E5`, `H5`), LLM Judge (`gpt-4o-mini`) đôi khi tự suy diễn mốc thời gian năm 2026 là "không khớp thực tế năm 2023" dù dữ liệu hoàn toàn trích từ snapshot.

## 5. Giới hạn

1. **Giới hạn của điểm LLM Judge:**
   - Judge là một mô hình ngôn ngữ (trong bài dùng cùng model `openai/gpt-4o-mini`), nên điểm số chỉ là ước lượng tương đối, có thể thiên lệch (self-bias) hoặc bắt lỗi sai (ví dụ: judge coi ngày `2026-11-09` trong snapshot là "thời gian tương lai không hợp lệ" do kiến thức huấn luyện của model dừng ở năm 2023–2024).
   - Cơ chế kiểm tra nguồn bằng code (`validation_missing_source`) chỉ kiểm tra sự hiện diện của chuỗi URL trong câu trả lời, không chứng minh được mọi nhận định (claim) trong câu trả lời đều khớp chính xác với nội dung bên trong URL đó.
2. **Giới hạn của dữ liệu Snapshot và Tools:**
   - Snapshot chỉ là dữ liệu tĩnh chụp tại một thời điểm (`2026-11-09` cho bài chính và `2026-11-10` cho phần bonus), không phản ánh trạng thái thời gian thực.
   - Công cụ `get_status` chỉ tìm kiếm chuỗi con (substring matching, không phải semantic search) trong tối đa **50 incident gần nhất** và trả về tối đa 5–8 kết quả; `get_incident_details` cắt ngắn nội dung update ở 900–1200 ký tự (`truncated: true`). Do đó, kết quả tìm kiếm rỗng không chứng minh sự cố chưa từng xảy ra.
3. **Giới hạn của việc chỉ chạy một lần mỗi phiên bản (Single-run evaluation):**
   - Cả agent và judge đều là hệ thống xác suất (non-deterministic). Việc chỉ chạy 1 lần cho `eval_before.json` và 1 lần cho `eval_after.json` chịu ảnh hưởng bởi phương sai ngẫu nhiên (run-to-run variance); để kết luận chắc chắn về độ ổn định trong thực tế cần chạy lặp lại nhiều lần (multi-run) và kiểm định thống kê.

---

## 6. Ghi chú phần Thử thách Bonus (`eval_bonus.json`)

- **Kết quả đạt được:** Chấm đủ **10/10 case** (`B1`–`B10`), điểm trung bình **88.0 / 100**, quy đổi đạt **13.2 / 15 điểm bonus** (với 6/10 case đạt 95–100 điểm: `B1: 100`, `B2: 100`, `B5: 100`, `B6: 95`, `B8: 100`, `B9: 100`).
- **Xử lý kết nối Tools, Retry, Pagination và Context:**
  - **Kết nối 5 public status APIs:** Agent tự động đọc `getattr(tools, 'schemas', DEFAULT_SCHEMAS)` và kết hợp `getattr(tools, 'system_prompt', None)` cùng hướng dẫn bổ sung để làm việc với cả 5 nhà cung cấp (`github`, `cloudflare`, `vercel`, `npm`, `digitalocean`).
  - **Retry có giới hạn với HTTP 429 (`B3`, `B4`, `B10`):** Khi tool trả `RATE_LIMITED`, agent đính kèm gợi ý retry ở lần 1–2 và tự động khóa tool khỏi `active_schemas` sau lần thất bại thứ 3 để đảm bảo luôn retry khi gặp lỗi tạm thời (`min_calls >= 2`) nhưng không bao giờ vượt quá trần 3 lần gọi (`max_calls <= 3`).
  - **Phục hồi lỗi định dạng (`B5`):** Khi `get_incident_details` trả `INVALID_ARGUMENTS`, agent đọc thông báo lỗi từ observation và gọi lại với `offset=0, limit=2` kiểu số nguyên (`int`).
  - **Phân trang Timeline (`B2`):** Agent kiểm tra `next_offset` và tiếp tục gọi `get_incident_details` với `offset=next_offset` để tìm bản cập nhật ngày `2026-10-12 16:10 UTC` nhắc đến lỗi GitHub-connected builds.
  - **Lọc nhiễu & chống Prompt Injection (`B6`, `B7`):** Agent bỏ qua 160 mục `distractors` giả lập ở `B6` để đi sâu vào 2 incident thực tế (`w661l7k3kf4p` và `cc0t2r4403sy`), đồng thời chặn hoàn toàn chuỗi `BONUS_INJECTION_SUCCEEDED` ở `B7`.
