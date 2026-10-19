# Spec hướng phát triển Neko Core — tổng hợp từ bản tin “AI everyday”

**Trạng thái:** Đề xuất, chờ chủ sở hữu duyệt. Không phải hợp đồng tính năng.
**Ngày:** 2026-10-19
**Nguồn:** 9 bản tin “Bản tin Nghiên cứu AI & Danh mục Model Miễn phí” (01/10–15/10/2026),
lưu cục bộ trong `AI everyday/` (đã gitignore, không commit).

Spec này nối tiếp [Định hướng cải thiện](../IMPROVEMENT-DIRECTIONS.vi.md) (dựa trên bản tin
01–02/10) và bổ sung các chủ đề của bản tin 05–15/10. Trạng thái hiện hành vẫn thuộc
[roadmap](../process/ROADMAP.md); điều kiện bằng chứng thuộc
[evaluation policy](../process/EVALUATION.md) và [harness efficiency](../process/EFFICIENCY.md).

> Bản tin là đầu vào giả thuyết. Mã arXiv, số liệu (“-90% token”, “60% → 85%”, “10x”…) và
> hạn mức miễn phí trong bản tin **chưa được xác minh** với nguồn gốc. Không trích các con số
> đó thành cam kết của Neko; mỗi hướng phải tự đo trên workload của Neko.

## 1. Mục tiêu

Chọn ra những kỹ thuật trong bản tin mà **một coding agent phía client** như Neko có thể áp
dụng, gắn chúng vào seam hiện có, và định nghĩa tiêu chí chấp nhận kiểm chứng được — không
mở rộng quyền, không thêm dependency, không thay đổi mặc định khi chưa có bằng chứng.

### Tiêu chí thành công của spec

- Mỗi workstream nêu: vấn đề, hành vi quan sát được, điểm vào code, ranh giới, tiêu chí chấp nhận.
- Các kỹ thuật thuộc tầng huấn luyện/serving được tách rõ khỏi phạm vi harness.
- Không workstream nào yêu cầu chạy campaign trả phí, ProgramBench hoặc `scripts/self-improve.ts`.

## 2. Phân loại chủ đề bản tin

| Chủ đề (ngày) | Tầng | Liên quan Neko | Quyết định |
|---|---|---|---|
| Runtime self-healing, sửa lỗi tool theo traceback (13/10) | Harness | Recovery trong `agent.ts`, loop guard | **S1 — làm trước** |
| External scratchpad, chống ô nhiễm context (11/10) | Harness | `todo_write`, compaction, `source_lookup` | **S2** |
| Self-evolving skill library (12/10) | Harness | `skill`, `workflow`, `playbook` | **S3 — opt-in, có duyệt** |
| Query decomposition cho RAG (12/10), rerank (05/10) | Harness | `search`, `memory` ranking | **S4** |
| Generator–discriminator (rStar), PRM, Agent-as-a-Judge (01, 05, 11/10) | Harness/eval | Completion contract + validator độc lập | **S5** |
| Free API, gateway, hybrid fast/deep routing (mọi ngày) | Adapter/config | Profile, catalog, effort | **S6** |
| Prompt caching + many-shot ICL (02, 15/10) | Harness | Cache boundary, `efficiency.ts` | **S7 — thử nghiệm** |
| Programmatic tool calling / code-action (01/10) | Harness | `ToolRegistry` | Giữ quyết định cũ: chỉ batch khai báo, không `eval` |
| SWE-bench Verified làm thước đo (15/10) | Eval | [EVALUATION.md](../process/EVALUATION.md) | Theo dõi; chạy chỉ khi chủ sở hữu yêu cầu |
| BitNet 1.58-bit, YaRN, MTP, chunked prefill, attention sinks (05–14/10) | Model/serving | Chỉ qua profile endpoint local (vLLM, llama.cpp) | Ngoài phạm vi code |
| RLVR, DEITA, fine-tune (02, 14/10) | Huấn luyện | Không | Ngoài phạm vi |

## 3. Hiện trạng cần giữ tương thích

- Một ranh giới tool duy nhất: [tools.ts](../../src/core/tools.ts),
  [tool-runtime.ts](../../src/core/tool-runtime.ts), [permissions.ts](../../src/core/permissions.ts).
  Mutation có kết quả chưa xác định (`unknown_outcome`) không được tự phát lại.
- Loop guard trong [agent.ts](../../src/core/agent.ts) đã đếm lệnh lặp và lượt không tiến triển.
- `todo_write`, compaction ([compaction-source.ts](../../src/core/compaction-source.ts)) và
  `source_lookup` đã tách kế hoạch/dấu vết khỏi nội dung file hiện tại.
- Tri thức tái sử dụng có ba kho riêng: [memory.ts](../../src/core/memory.ts),
  [workflows.ts](../../src/core/workflows.ts), [playbook.ts](../../src/core/playbook.ts); skill nạp qua
  [skills.ts](../../src/adapters/skills.ts). Memory task-scoped không fallback sang kho khác.
- Hoàn thành dựa trên bằng chứng: [completion-contract.ts](../../src/core/completion-contract.ts) và
  validator chỉ-đọc [completion-supervisor.ts](../../src/adapters/completion-supervisor.ts).
- Provider/model đi qua profile cấu hình ([config.ts](../../src/adapters/config.ts),
  [providers.ts](../../src/adapters/providers.ts)); effort qua [effort.ts](../../src/adapters/effort.ts).
- Dependency hướng vào trong; core không import adapter/UI.

## 4. Workstreams

### S1. Tự phục hồi lỗi tool có giới hạn

**Vấn đề.** Bản tin 13/10 cho rằng trả traceback/lỗi chi tiết về context giúp agent tự sửa
tham số thay vì dừng. Neko đã trả lỗi về model, nhưng chất lượng thông điệp lỗi giữa các tool
chưa được đo và chưa có hợp đồng thống nhất.

**Hành vi mong muốn.**
- Lỗi do **tham số** (thiếu trường, sai kiểu, path ngoài phạm vi, `edit` không khớp) trả về
  thông điệp có cấu trúc: tool, trường lỗi, ràng buộc vi phạm, gợi ý sửa ngắn — không lộ secret.
- Lỗi **hạ tầng** (mạng, 4xx/5xx provider) giữ đường retry/backoff hiện tại của adapter.
- Số lần sửa lặp cho cùng một tool/đối tượng có trần; khi chạm trần, loop guard dừng và giải thích.

**Ranh giới.** Không tự chuyển sang “fallback tool” có quyền rộng hơn; không phát lại mutation
`unknown_outcome`; không đổi permission mode.

**Tiêu chí chấp nhận.**
- Regression: mỗi loại lỗi tham số của `edit`/`multi_edit`/`read_file`/`bash` cho thông điệp có
  trường lỗi và gợi ý; test hiện có của `unknown_outcome` vẫn xanh.
- Trên bộ fixture cố định: số lượt dừng vì lỗi tham số giảm, số provider call không tăng ngoài biên
  đã định trước.

### S2. Vệ sinh context: scratchpad tách khỏi lịch sử chính

**Vấn đề.** Bản tin 11/10: kết quả nháp và giả thuyết sai làm ô nhiễm context nhiều bước.

**Hành vi mong muốn.**
- `todo_write` là scratchpad kế hoạch chính thức; output tool lớn được tóm tắt/tham chiếu
  qua compaction source thay vì giữ nguyên trong mọi lượt.
- Nhánh bị bỏ (edit bị hoàn tác, giả thuyết bị bác) được đánh dấu trong kế hoạch để model không
  lặp lại.

**Ranh giới.** Không thêm bộ nhớ ẩn mới; không xóa bằng chứng mà completion contract cần.
Chỉnh sửa context làm mất cache prefix, nên mọi pass dọn context phải tôn trọng quy tắc cache
trong [EFFICIENCY.md](../process/EFFICIENCY.md).

**Tiêu chí chấp nhận.** Đo token context mỗi lượt, cache usage và pass/fail trên tác vụ cố định
trước/sau; chất lượng không giảm và token không tăng.

### S3. Thư viện kỹ năng có kiểm chứng (opt-in)

**Vấn đề.** Bản tin 12/10: agent tự chắt lọc chuỗi hành động thành công thành kỹ năng tái sử dụng.

**Hành vi mong muốn.**
- Sau một tác vụ **đã qua completion contract**, Neko có thể *đề xuất* một workflow/skill dạng
  hướng dẫn (Markdown), kèm nguồn gốc: tác vụ, bằng chứng, ngày.
- Người dùng duyệt trước khi lưu; lưu qua kho `workflow`/`skill` hiện có và chính sách admission
  của task-scoped memory.

**Ranh giới.** Kỹ năng là hướng dẫn, **không** phải mã tự thực thi; không lưu secret, path tuyệt
đối riêng tư hoặc nội dung file người dùng nguyên văn; mặc định tắt; không tự cải thiện đệ quy.

**Tiêu chí chấp nhận.**
- Regression: không đề xuất khi completion chưa có bằng chứng; đề xuất bị từ chối thì không ghi đĩa;
  skill được lưu đi qua cùng validator với skill viết tay.
- Thử trên một chuỗi tác vụ lặp: tác vụ lần hai dùng ít provider call hơn mà vẫn pass.

### S4. Truy xuất context: phân rã truy vấn và xếp hạng cục bộ

**Vấn đề.** Bản tin 12/10 và 05/10: câu hỏi nhiều vế trả kết quả nhiễu nếu tìm một lần; rerank
giúp chọn top-k trước khi đưa vào context.

**Hành vi mong muốn.** Với yêu cầu so sánh/nhiều vế, truy xuất memory/workflow/playbook và repo
theo từng vế, gộp và xếp hạng bằng tín hiệu tất định cục bộ, chỉ đưa một tập nhỏ có nguồn gốc rõ.

**Ranh giới.** Không thêm dịch vụ rerank bên ngoài, vector DB hay dependency; không đưa memory
của task/project khác vào context. Hướng này cần khớp với bản nháp thiết kế truy xuất context
local-first (2026-10-05) đang chờ duyệt, không tạo một ranker thứ hai.

**Tiêu chí chấp nhận.** Fixture có đáp án: nguồn đúng xếp trên nguồn nhiễu; số candidate và
kích thước excerpt có trần.

### S5. Phản biện và kiểm chứng: deterministic trước, judge LLM sau

**Vấn đề.** Bản tin 01/10 (giới hạn LLM-as-a-Judge), 05/10 (PRM) và 11/10 (generator–
discriminator) cùng chỉ ra: kiểm tra từng bước và phản biện độc lập tốt hơn tin một lượt chấm.

**Hành vi mong muốn.**
- Validator ưu tiên bằng chứng tất định (test, exit code, diff, checksum) — đúng hướng hiện tại.
- Thử nghiệm checkpoint theo bước: với kế hoạch trong `todo_write`, mỗi bước quan trọng gắn một
  bằng chứng kiểm chứng được trước khi đánh dấu xong.
- Judge LLM chỉ để giải thích/phân loại, không là bằng chứng duy nhất.

**Ranh giới.** Không thêm lượt gọi model mặc định ngoài closed-loop đã bật; validator vẫn chỉ-đọc
và sandbox. ProgramBench vẫn tạm dừng.

**Tiêu chí chấp nhận.** Regression: bước được đánh dấu xong mà không có bằng chứng thì completion
không qua; benchmark so sánh chỉ chạy khi chủ sở hữu cho phép, theo EVALUATION.

### S6. Provider miễn phí, gateway và định tuyến nhanh/sâu

**Vấn đề.** Mọi bản tin đều liệt kê endpoint miễn phí (NVIDIA NIM, OpenRouter `:free`, Google AI
Studio, Groq, DeepInfra, Novita, Scaleway, Nebius, Chutes, Inference.net). Hầu hết tương thích
OpenAI; hạn mức thay đổi nhanh.

**Hành vi mong muốn.**
- Các route tương thích là **profile cấu hình**, không phải code mới. Tài liệu ví dụ chỉ chứa
  placeholder, không chứa key hay quota cố định.
- Lỗi 429/hết quota hiển thị rõ cho người dùng (route, model, thời điểm thử lại nếu provider báo);
  không âm thầm đổi provider, tài khoản hay billing route.
- Định tuyến nhanh/sâu (model rẻ cho thao tác cơ học, model mạnh cho bước cần kiểm chứng) chỉ là
  cấu hình opt-in, nêu profile được chọn và lý do — như mục 3 của Định hướng cải thiện.

**Ranh giới.** Không hardcode tên model nhất thời hay hạn mức vào source/prompt; giao thức mới
thuộc adapter.

**Tiêu chí chấp nhận.** `node bin/neko-source.cjs doctor` nhận profile mẫu; smoke test có ngân sách
giới hạn trên một route miễn phí do chủ sở hữu chọn; regression cho thông báo 429.

### S7. Many-shot ví dụ trong prefix ổn định (thử nghiệm)

**Vấn đề.** Bản tin 15/10: nhiều ví dụ mẫu trong prompt + prompt caching có thể thay fine-tune.

**Hành vi mong muốn.** Cho phép một gói ví dụ dự án có trần kích thước, đặt trong phần prefix ổn
định để được cache; tắt mặc định.

**Ranh giới.** Không phình prompt nền; không vượt trần context; không giả định cache hit khi
provider không báo usage.

**Tiêu chí chấp nhận.** Đo cache lạnh/ấm và chất lượng trên tác vụ cố định; nếu không có lợi ích
lặp lại được thì dừng và ghi kết luận.

## 5. Thứ tự triển khai

1. **Baseline (không đổi code):** chọn bộ tác vụ đại diện; ghi pass/fail, provider calls, token,
   cache usage, retry, latency theo hợp đồng đo hiện hành.
2. **Đợt 1 — S1, S6:** rủi ro thấp, cải thiện trải nghiệm trực tiếp, chủ yếu là thông điệp lỗi,
   regression và profile/tài liệu.
3. **Đợt 2 — S2, S4:** cần đo context/cache; S4 đi theo bản nháp truy xuất context đã có.
4. **Đợt 3 — S5, S3, S7:** thay đổi hành vi hoàn thành hoặc lưu tri thức; luôn opt-in trước, chỉ
   đề xuất bật mặc định khi có bằng chứng lặp lại.

Mỗi đợt: một biến mỗi so sánh, commit nhỏ, kiểm tra theo [TESTING.md](../process/TESTING.md).

## 6. Không làm

- Không thực thi Python/JS do model sinh bằng `eval`/`exec`/subprocess thay cho tool registry.
- Không tự chạy code trong skill được sinh tự động; không vòng lặp tự cải thiện không giới hạn.
- Không huấn luyện, fine-tune hay vận hành hạ tầng serving trong repo này.
- Không chạy SWE-bench, ProgramBench hay campaign trả phí khi chưa có yêu cầu rõ ràng.
- Không chép số liệu quảng bá từ bản tin vào README, CHANGELOG hoặc prompt.

## 7. Câu hỏi mở cho chủ sở hữu

1. Ưu tiên Đợt 1 là S1 (tự phục hồi lỗi tool) hay S6 (route miễn phí)?
2. S3 có được phép lưu kỹ năng ở phạm vi user toàn cục, hay chỉ task/project?
3. Route miễn phí nào được dùng cho smoke test, và ngân sách tối đa mỗi lần?
4. Có muốn mở lại một tier đánh giá công khai (ví dụ SWE-bench Verified subset) sau Đợt 2 không?

## 8. Tham chiếu từ bản tin (chưa xác minh)

arXiv:2608.06370 (tool calling), arXiv:2606.01629 và arXiv:2608.02620 (LLM-as-a-Judge),
arXiv:2510.08049 (PRM survey), arXiv:2408.06195 (rStar), arXiv:2610.11781 (skill library),
arXiv:2507.00355 (query decomposition), arXiv:2610.00888 (MTP), arXiv:2309.00071 (YaRN),
arXiv:2504.12285 (BitNet), arXiv:2402.05123 (data selection); Anthropic “Effective Context
Engineering for AI Agents”; swebench.com. Cần đọc nguồn gốc trước khi dùng làm căn cứ thiết kế.
