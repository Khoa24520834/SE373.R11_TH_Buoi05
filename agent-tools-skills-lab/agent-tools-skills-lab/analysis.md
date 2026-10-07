# Analysis: tool list_files và skill refund-policy

Bản sao dùng để làm bài: `stage-01-files-baitap/`, `stage-02-skills-baitap/`. Bản mẫu `stage-01-files/` và `stage-02-skills/` giữ nguyên.

## Thay đổi

| File | Nội dung |
|---|---|
| `tools/files.py` | Thêm `_list()` và tool `list_files(path)`. Dùng lại `_resolve()` (chặn path rỗng, path tuyệt đối, `~`, `..`, symlink thoát workspace) và `_error()` (cấu trúc `{"ok": false, "error": {code, message}}`). Bỏ qua mục symlink trỏ ra ngoài workspace. |
| `tools/__init__.py`, `agent.py` | Đăng ký: `TOOLS = [list_files, read_file, write_file]` → xuất hiện trong schema tools gửi model. |
| `config.py` | `PROJECT_NAME` của bản sao, text mô tả tools trên UI. |
| `agent.py` (`GeminiChatOpenAI`) | Adapter provider, **ngoài phạm vi đề**, chỉ bật khi `OPENAI_BASE_URL` là endpoint Gemini. Gemini 3 qua endpoint OpenAI-compatible bắt buộc `thought_signature` trong tool call gửi lại; `ChatOpenAI` không giữ trường này nên mọi lượt có tool call đều lỗi 400. Adapter gắn chữ ký bỏ qua kiểm tra mà Google cho phép. Không đổi prompt, tool hay logic agent. |
| `workspace/data/policies/`, `fixtures/data/policies/` | 2 tài liệu chính sách (stage 01 và 02). |
| `workspace/skills/refund-policy/`, `fixtures/skills/refund-policy/` | `SKILL.md` + `references/answer-template.md` (stage 02). |
| `tests/` | Test cho `list_files`; cập nhật số tool/skill trong test cũ. |

System prompt (`prompts.py`) không đổi. Không tên file chính sách, nội dung chính sách hay đáp án nào nằm trong prompt hoặc mã nguồn tool; skill chỉ nêu thư mục `data/policies/`.

## Kiểm tra tool trực tiếp

Gọi `list_files.invoke(...)` trên workspace của `stage-02-skills-baitap`:

| Input | Kết quả |
|---|---|
| `data/policies` (thư mục hợp lệ) | `ok: true`, 2 entries sắp theo tên, mỗi entry có `name`, `path` (tương đối workspace), `type: "file"` |
| `data/policies/policy-from-oct.md` (file) | `ok: false`, `NOT_A_DIRECTORY` |
| `data/khong-ton-tai` (không tồn tại) | `ok: false`, `DIR_NOT_FOUND` |
| `../..` (vượt workspace) | `ok: false`, `PATH_OUTSIDE_WORKSPACE` |
| `C:/Windows` (đường dẫn tuyệt đối) | `ok: false`, `PATH_OUTSIDE_WORKSPACE` |

Unit test (`uv run pytest tests/test_files.py`): `test_list_dir_sorted_non_recursive`, `test_list_errors` (file / không tồn tại / `..` / `data/../../`), `test_list_absolute_and_symlink_escape_blocked`, `test_list_files_tool_finds_policies`: pass. Phần symlink bị skip trên Windows vì tạo symlink cần quyền admin hoặc Developer Mode (2 test symlink có sẵn của bản mẫu cũng fail vì lý do này).

## Stage 00: giới hạn của agent

Trace: `stage-00-chat/traces/20261007-103242_f9a0b9c8_turn01_0ce4fbe2.jsonl` (gemini-3.8-flash).

- #1 `user_submitted`: `tools=[]`. #2 `model_request` call#1: `tools_schema=[]`, chỉ 1 message. Cả lượt có 1 model call, 0 tool call.
- Câu trả lời: agent tự nói "không được cấp công cụ để ... xem tài liệu chính sách". Nó tính đúng 8 ngày, nhưng không kết luận được và đưa ra các giả định (7, 14, 30 ngày) lấy từ kiến thức chung, rồi đề nghị người dùng dán chính sách vào.
- Thiếu thông tin: tài liệu chính sách (ngày đổi chính sách 2026-10-01, thời hạn 7/14 ngày, phí 10%/0, điều kiện kích hoạt). Thiếu khả năng: tìm file (liệt kê thư mục) và đọc file. Agent không có kết quả đọc tài liệu nào; mọi con số chính sách trong câu trả lời đều là phỏng đoán.

## Kết quả từng trường hợp (stage-02-skills-baitap)

Mỗi trường hợp chạy trong một cuộc trò chuyện mới (conversation_id mới, history chỉ có câu hỏi), câu hỏi không nhắc tên skill hay tên file. Trace nằm trong `stage-02-skills-baitap/traces/`.

| Trường hợp | Model | Kết quả | Trace |
|---|---|---|---|
| A (tên file gốc) | gemini-3.8-flash | Chính sách trước tháng 10; 8 ngày (giới hạn 7); **không đủ điều kiện**; căn cứ `data/policies/policy-before-oct.md` | `20261007-103528_c3180830_turn01_e7cb4a58.jsonl` |
| Thiếu thông tin | gemini-3.8-flash | Chỉ hỏi "sản phẩm đã kích hoạt hay chưa", **không kết luận** | `20261007-103611_da55f4c8_turn01_ffb402af.jsonl` |
| A (sau đổi tên) | gemini-3.7-flash | Chính sách trước tháng 10; 8 ngày (giới hạn 7); **không đủ điều kiện**; căn cứ `data/policies/tai-lieu-2.md` | `20261007-103826_17da8b1d_turn01_1cbbe958.jsonl` |
| B (sau đổi tên) | gemini-3.7-flash | Chính sách từ tháng 10; 10 ngày (giới hạn 14); **đủ điều kiện, không thu phí**; căn cứ `data/policies/tai-lieu-1.md` | `20261007-103901_2cc26a99_turn01_b8d6d318.jsonl` |

Đổi tên: `policy-before-oct.md` → `tai-lieu-2.md`, `policy-from-oct.md` → `tai-lieu-1.md`, nội dung giữ nguyên. Tên mới cố ý đảo thứ tự và không chứa thông tin ngày, để agent phải chọn chính sách theo nội dung. Kết luận A và B không thay đổi.

Hai lần chạy sau đổi tên dùng `gemini-3.7-flash` vì `gemini-3.8-flash` đã hết quota miễn phí trong ngày (20 request/ngày/model). Trace lỗi 429 `20261007-103317_*` và `20261007-103641_*` không phải bài nộp.

### Bằng chứng trong trace (sequence `#`)

**A (tên gốc)**, `e7cb4a58`:
- #1 `user_submitted`: `tools=["list_files","read_file","write_file"]`; #2 call#1: `tools_schema` có `list_files`, chỉ 1 message (system prompt chỉ có catalog).
- #5 `tool_finished read_file skills/refund-policy/SKILL.md` ok → #6 call#2 có tool result SKILL.md trong messages.
- #11 `tool_finished list_files data/policies` ok, entries `policy-before-oct.md`, `policy-from-oct.md`; #10 `read_file .../answer-template.md` ok.
- #16, #17 `read_file` 2 file chính sách ok → #18 call#4 có đủ 5 tool result; câu trả lời dẫn `policy-before-oct.md`.

**Thiếu thông tin**, `ffb402af`: #5 `read_file SKILL.md` ok → #6 call#2 trả lời câu hỏi lại. Không gọi `list_files` hay đọc chính sách, không giả định "chưa kích hoạt".

**A (sau đổi tên)**, `1cbbe958`: #11 `list_files data/policies` trả entries `tai-lieu-1.md`, `tai-lieu-2.md` (tên mới). #16, #17 `read_file` 2 file tên mới ok. Câu trả lời dẫn `data/policies/tai-lieu-2.md`.

**B (sau đổi tên)**, `b8d6d318`:
- Skill và reference trong history: #5 `read_file skills/refund-policy/SKILL.md` ok, #10 `read_file skills/refund-policy/references/answer-template.md` ok; #12 call#3 có cả hai tool result trong messages.
- #11 `list_files` tìm thấy tên mới; #16 `read_file tai-lieu-2.md`, #17 `read_file tai-lieu-1.md` ok; câu trả lời dẫn `data/policies/tai-lieu-1.md`.
- Conversation mới: #2 call#1 chỉ có 1 message, không có tài liệu nào từ lịch sử cũ; mọi đường dẫn đọc được đều là tên mới.

## Câu hỏi cuối bài

**Vì sao cần tool để tìm file và skill để hướng dẫn chọn chính sách?**

Tool và skill giải quyết hai việc khác nhau. Tool `list_files` là *khả năng*: model không nhìn thấy hệ thống file, chỉ biết những gì nằm trong context. Không có tool liệt kê, model chỉ có thể đoán tên file rồi thử `read_file`; đoán sai thì không đọc được. `list_files` trả về tên file thật tại thời điểm chạy. Skill là *quy trình*: nó cho model biết tìm ở đâu (`data/policies/`), phải đọc phạm vi hiệu lực để chọn chính sách theo ngày mua chứ không theo tên file, cách tính số ngày, khi nào phải hỏi lại, và câu trả lời phải có những mục nào. Skill chỉ được nạp khi câu hỏi khớp description, nên không làm nặng context của các câu hỏi khác.

**Nếu chưa có tool tìm file, sửa prompt có giải quyết được yêu cầu đổi tên file không?**

Không. Prompt là văn bản cố định, viết trước khi chạy. Muốn agent đọc được file chỉ bằng `read_file` thì prompt phải ghi sẵn tên file; khi file bị đổi tên, tên trong prompt sai và `read_file` trả `FILE_NOT_FOUND`. Mỗi lần đổi tên lại phải sửa prompt, tức là con người làm thay việc tìm file, và đề bài cũng cấm ghi cố định tên file trong prompt. Prompt chỉ thay đổi cách model *dùng* khả năng nó có, không tạo ra khả năng mới. Thông tin "hiện có những file nào" chỉ có thể vào context qua một tool result lúc chạy, nên cần `list_files`.
