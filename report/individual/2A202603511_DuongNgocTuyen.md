# Báo cáo cá nhân — Day 6

**Dương Ngọc Tuyên · 2A202603511 · L3A** · Chạy ngày 2026-11-13 (UTC `2026-11-13T10:19:06`).

## Dữ liệu và pipeline

- **Chủ đề, nguồn/quyền, số trang và quy tắc tính trang.** Sách *Introduction to Intellectual Property* (OpenStax, publish 2021-02-22), giấy phép **CC BY 4.0**: được phép tái phân phối và chỉnh sửa khi ghi công. PDF web 201 trang vật lý, SHA-256 `1498…378A`. Corpus gồm Chương 3 (Copyright), Chương 4 (Trademark) và Chương 5 (Trade Secret), tức các trang PDF 102–134, 141–164 và 172–187, tổng **73 trang nội dung** và 183.128 ký tự sau chuẩn hóa. Các trang bị loại: trang mở chương chỉ có hình và dàn ý (101, 171), Assessment Questions, glossary, answer key và index. Mỗi trang được tính phải có ít nhất 200 ký tự sau khi làm sạch; trang ngắn nhất có 577 ký tự. `page` là số trang PDF vật lý (1-based), còn `printed_page` = PDF − 8. Danh sách trang và số ký tự từng trang nằm trong [data/manifest.json](../../data/manifest.json).
- **Chuẩn hóa, lỗi và giới hạn.** PyMuPDF đọc theo block. Pipeline loại 109 footer, 94 khối footnote, 129 ký hiệu footnote dạng superscript la mã, 34 ảnh, 1 tiêu đề chương và dàn ý chương. Ký tự được chuẩn hóa NFKC, xóa soft hyphen, nối các dòng bị ngắt, giữ ngắt đoạn (`\n\n`) và dòng danh sách. Không cần OCR vì mọi trang đều có lớp text. Một lỗi đã phát hiện và sửa khi đọc corpus: bộ lọc superscript ban đầu xóa cả `E = MC²` và `20th`, nên sau đó chỉ xóa các marker la mã. Giới hạn còn lại:
  - Danh sách "six flavors" của Creative Commons (p132) là hình nên bị mất.
  - Bảng Pros/Cons (p173) và bảng Case Studies (p186) bị dẹt thành đoạn văn.
  - Câu mở đầu đoạn "Bacardi…" (p151) bị mất khi trích xuất.
  - Chunk không vượt ranh giới trang, nên đoạn văn bị ngắt trang sẽ bị tách (ví dụ heading cuối p181).

  Log chi tiết nằm trong [results/ingestion_log.json](../../results/ingestion_log.json).
- **Embedding, vector store, metric, lệnh chạy và chi phí.**
  - Embedding: `BAAI/bge-small-en-v1.5`, 384 chiều, chạy local qua **fastembed 0.9.1 + ONNX Runtime 1.31.1** trên CPU. Artifact thực tế là bản ONNX lượng tử hóa `Qdrant/bge-small-en-v1.5-onnx-Q`. Query được thêm prefix BGE `"Represent this sentence for searching relevant passages: "`, document không có prefix, vector được chuẩn hóa L2. Chunk dài nhất 196 token, không bị cắt ở giới hạn 512.
  - Vector store: **PostgreSQL 16.16 + pgvector 0.8.7** (Docker), bảng `chunks`, cosine `<=>`, **exact search** (không dùng ANN index).
  - Lệnh chạy: `python -m src.run all`.
  - Chi phí: **0 USD** (không dùng API).
  - Dự phòng: ban đầu định dùng sentence-transformers, nhưng Windows Smart App Control chặn DLL của PyTorch (`WinError 4551`). Tôi chuyển sang fastembed với cùng model gốc, áp dụng cho cả ba cấu hình; không có fallback sang vector giả.
- **Metadata bắt buộc, trường lọc bổ sung, nguồn giá trị và lý do chọn.** Các trường bắt buộc là `document_id`, `source` và `page`, cộng thêm `printed_page`. Các trường bổ sung:
  - `chapter` (int) và `chapter_title`, lấy từ mục lục cấp 1 của PDF.
  - `section` (text), lấy từ mục lục cấp 2 và khớp với heading 15,6 pt trên trang.
  - `subsection`, lấy từ heading đậm 13 pt.

  **Trường lọc là `chapter`.** Đây là giá trị đáng tin vì có sẵn trong mục lục PDF và được kiểm tra chéo với phạm vi trang khai báo. Ba chương còn dùng lại cùng thuật ngữ ("fair use", "damages", "injunction", thời hạn bảo hộ), nên lọc theo chương thực sự giúp loại nhiễu.

## Cấu hình và benchmark

- **A / B / C.**
  - **A:** cửa sổ ký tự cố định 500, overlap 50, trong phạm vi một trang, không căn theo câu.
  - **B:** gom nguyên đoạn văn đến tối đa 800 ký tự. Mỗi heading mở chunk mới; đoạn quá dài được tách ở cuối câu.
  - **C:** dùng lại *đúng index và vector của B*, chỉ thêm `WHERE chapter = %s`, và chỉ áp dụng cho 6 câu có filter.

  A–B chỉ khác cách chunk (lưu ý giới hạn kích thước cũng khác nhau, 500 và 800). **Khác biệt duy nhất giữa B và C là filter.** Cả ba dùng cùng corpus, model, metric, top-k = 3 và bộ câu hỏi; tham số nằm trong [starter/src/config.py](../../starter/src/config.py).
- **Bộ câu hỏi.** 25 câu: 19 dev và 6 test, gồm 8 direct, 4 paraphrase, 7 conditional và 6 metadata_filter. Có 6 câu filter, 2 trong số đó thuộc test (t01, t02). Tập được chia theo section để giảm rò rỉ: câu test chỉ lấy từ 3.6, 4.8, 4.10 và 5.5, và không câu dev nào dùng các section này. Câu filter cố ý mơ hồ giữa các chương; ví dụ thời hạn khởi kiện là 3 năm theo UTSA nhưng là 1 năm theo luật copyright 1790. `filter` là phạm vi do người dùng cung cấp trước khi truy xuất ("đang học Chương X"), không suy ra từ đáp án. Hàm `load_benchmark` tự động kiểm tra mọi `evidence_quote` (kể cả `alternative_evidence`) có nằm nguyên văn trên trang gold sau chuẩn hóa; cả 25 câu đều đạt **trước** lần truy xuất đầu tiên. Xem [benchmark/questions.json](../../benchmark/questions.json).
- **Tiêu chí chọn chiến lược đã chốt trước khi xem test** (ghi trong `questions.json` và experiment log trước khi chạy): chọn Evidence Hit@3 dev cao nhất; nếu hòa thì xét Hit@1 dev, sau đó số chunk ít hơn.

## Kết quả

| Strategy | Split | Hit@1 (n/N) | Evidence Hit@3 (n/N) | Chunk count | Thời gian | Chi phí |
|---|---|---|---|---|---|---|
| A | dev | 11/19 | 15/19 | 436 | embed 36,6 s · ghi 1,7 s · 25 truy vấn 3,0 s | 0 |
| A | test | 1/6 | 2/6 | 436 | (như trên) | 0 |
| B | dev | 14/19 | **19/19** | 337 | embed 44,9 s · ghi 1,4 s · 25 truy vấn 2,1 s | 0 |
| B | test | 4/6 | 5/6 | 337 | (như trên) | 0 |
| C | dev | **17/19** | **19/19** | 337 (index B) | không embed thêm · 25 truy vấn 1,7 s | 0 |
| C | test | 4/6 | 5/6 | 337 (index B) | (như trên) | 0 |

Parse PDF mất 5,1 s và tải model 1,5 s; các bước này dùng chung cho mọi cấu hình. Thời gian đo một lần trên CPU laptop nên chỉ mang tính tham khảo.

| Strategy | Coherence trung bình /2 (n) | Metadata đủ và đúng (n/N) | Filter tuân thủ (n/N hoặc N/A) | Grounding 0/1/2 (số câu) và trung bình /2 |
|---|---|---|---|---|
| A | 0,70 (10 vị trí: ba điểm 0, bảy điểm 1) | 18/18 | test 0/2 · dev 0/4 | 0 / 1 / 5 → 1,83 |
| B | 1,60 (10 vị trí: bốn điểm 1, sáu điểm 2) | 18/18 | test 0/2 · dev 0/4 | 0 / 1 / 5 → 1,83 |
| C | 1,60 (cùng chunk với B) | 18/18 | **test 2/2 · dev 4/4** | 0 / 1 / 5 → 1,83 |

Ghi chú về metadata: "đủ và đúng" xét `source`, `page`, `printed_page`, `chapter` và `section` trên 6 câu test × top-3. Ngoài kiểm tra thủ công, text của từng chunk được dò lại bằng trình trích xuất thứ hai (pypdf): 53/54 bản ghi khớp. Bản ghi còn lại không khớp chỉ vì pypdf giữ marker footnote "xlv"; tôi đã kiểm tra tay là đúng trang 162. Trường phụ `subsection` sai ở 4/18 bản ghi (A), 7/18 (B) và 6/18 (C): pipeline gán nhầm tiêu đề hộp video ("Establishing Your Trademark Protection", "Trademark Infringement and Remedies") làm subsection.

**Đường dẫn trace, nhãn thủ công và cách chọn mẫu.**
- Trace top-3 đầy đủ (query, chunk ID, text, score, metadata, nhãn hit, độ phủ evidence) nằm trong [results/traces/](../../results/traces/); chỉ số nằm trong [results/metrics.json](../../results/metrics.json).
- Nhãn thủ công nằm trong [results/audit/](../../results/audit/) (`*_labels.csv`), tổng hợp trong [results/audit_summary.json](../../results/audit_summary.json).
- **Coherence:** 10 vị trí được cố định bằng quy tắc trước khi chấm, gồm trang ở thứ hạng `i·73/10` (i = 0…9) và đầu đoạn thứ 3 của trang đó. Mỗi vị trí chấm chunk A và chunk B chứa nó; C dùng chung chunk với B.
- **Metadata và grounding:** chấm cả 6 câu test × top-3 cho mỗi cấu hình.
- **Người chấm:** nhãn là bản nháp do AI (Claude) lập, sau khi đọc toàn bộ text của từng chunk. Tác giả cần rà soát lại. Vì cùng một tác nhân vừa viết benchmark vừa chấm, có nguy cơ thiên lệch.

## Phân tích và kết luận

- **So sánh A–B (chỉ đổi cách chunk).** B hơn A rõ ở Evidence Hit@3: dev 19/19 so với 15/19, test 5/6 so với 2/6. B cũng hơn ở coherence (1,60 so với 0,70) và dùng ít chunk hơn 23% (337 so với 436). Lý do: cửa sổ 500 ký tự của A cắt ngang câu, nên 7/8 lần A trượt Hit@3 có cờ `evidence_split_suspected` (một chunk trên trang gold chứa ≥ 50% evidence nhưng không đủ). Ngược lại, grounding thủ công của A và B bằng nhau (1,83). Overlap 50 ký tự khiến hai nửa evidence bị cắt thường cùng lọt vào top-3, nên người đọc vẫn ghép được câu trả lời. Định nghĩa Hit@3 một chunk của bài lab nghiêm hơn mức người đọc cần.
- **So sánh B–C (chỉ đổi filter).** Trên 4 câu filter của dev, filter đưa 3 câu (q11, q12, q14) từ hạng 2 lên hạng 1, làm Hit@1 dev tăng từ 14/19 lên 17/19. Mức tuân thủ phạm vi tăng từ 0/4 lên 4/4 ở dev và từ 0/2 lên 2/2 ở test. Ở test, hai câu filter đã đạt hạng 1 ngay cả với B, nên chỉ số test của C bằng B; lợi ích quan sát được là top-3 sạch hơn. Ví dụ với t02, B đưa vào hạng 2–3 các đoạn về copyright damages ($30k–$80k) và trade secret exemplary damages (tối đa 2×); C thay chúng bằng danh sách cách tính bồi thường trademark.
- **Hai ví dụ sai.**
  1. **t05 (test, B/C trượt Hit@3, gold p120):** chunk `p120:para:04` kết thúc bằng câu dẫn "As the court of appeals for the ninth circuit put it in 2011:". Nội dung trích dẫn chứa evidence ("must demonstrate a likelihood of irreparable harm") nằm trong chunk riêng `para:05`, không lọt top-3. Lỗi này do B tách câu dẫn khỏi phần trích dẫn khi nhóm vượt 800 ký tự. Grounding chỉ được 1, vì chunk hạng 1 chỉ nêu ý gần đúng. A lại truy xuất được phần này (hạng 2), vì cửa sổ cố định vô tình giữ cả hai phần. Tôi không sửa gold sau khi thấy kết quả test.
  2. **q12 (dev, A trượt; B hạng 2):** câu hỏi về cách tính damages, phạm vi là trade secret. Không có filter, hạng 1 của A và B đều là `p162` (trademark remedies, chương 4) vì từ vựng "damages / lost profits" trùng nhau. Đây chính là loại nhiễu xuyên chương mà C loại bỏ (C: hạng 1 = p186). Riêng A còn bị cắt evidence (độ phủ 0,84).
- **Chiến lược chọn, đánh đổi và điều kiện áp dụng.** Theo tiêu chí đã chốt, **C** được chọn: hòa B về Hit@3 dev (19/19) và thắng ở Hit@1 dev (17/19 so với 14/19). C không tốn thêm lưu trữ hay thời gian embed so với B. Truy vấn có filter còn nhanh hơn một chút nhờ quét ít dòng hơn. Đổi lại, C chỉ có lợi khi người dùng thực sự biết phạm vi (chương). Nếu filter sai thì C sẽ loại mất đáp án; benchmark không kiểm tra trường hợp này. B tốn thời gian embed nhiều hơn A khoảng 23% dù có ít chunk hơn, vì chunk dài hơn.
- **Hạn chế.**
  - Corpus chỉ có một cuốn sách gồm 3 chương.
  - Test chỉ có 6 câu, nên mỗi câu tương đương 17 điểm phần trăm. Không thể kết luận C tốt hơn B trên test, và không suy rộng ra ngoài corpus này.
  - Bộ câu hỏi và nhãn do cùng một tác nhân (AI hỗ trợ) lập; có nguy cơ thiên lệch theo cách viết của corpus, ví dụ evidence là câu nguyên văn ngắn.
  - Hit@3 chỉ cộng điểm khi một chunk chứa trọn evidence, nên đánh giá thấp A.
  - Thời gian được đo một lần trên laptop CPU và chưa lặp lại.
  - Kích thước A (500) và B (800) khác nhau, nên A–B không tách riêng được ảnh hưởng của *ranh giới* và *độ dài* chunk.
  - Nhãn subsection còn lỗi (xem trên).
- **Nếu làm lại hoặc mở rộng.**
  - Cho phép chunk B nối tiếp qua ranh giới trang.
  - Gộp câu dẫn kết thúc bằng ":" với đoạn trích dẫn ngay sau (sửa được lỗi t05).
  - Lọc tiêu đề hộp video khỏi `subsection`.
  - Thêm cấu hình A′ (fixed 800) để tách ảnh hưởng của độ dài chunk.
  - Thêm câu hỏi có filter *sai* để đo rủi ro của C.
  - Thử hybrid BM25 trên dev.
  - Nhờ một người chấm độc lập chấm lại nhãn.

**Công cụ AI:** Claude Code (Claude) hỗ trợ viết code pipeline, soạn bộ câu hỏi và lập nhãn audit nháp. Cách kiểm tra:
- Evidence được kiểm tra tự động với text trang.
- Metadata được đối chiếu bằng trình trích xuất thứ hai.
- Mọi số liệu trong báo cáo lấy từ file kết quả do pipeline tạo ra.
- Tác giả cần rà soát nhãn thủ công trước khi nộp.
