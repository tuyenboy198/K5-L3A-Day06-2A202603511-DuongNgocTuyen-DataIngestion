# Bài làm · Dương Ngọc Tuyền (2A202603511)

Corpus: OpenStax *Introduction to Intellectual Property* (CC BY 4.0), Chương 3–5, 73 trang nội dung. Embedding: `BAAI/bge-small-en-v1.5` chạy local qua fastembed/ONNX (384 chiều), lưu và truy xuất bằng PostgreSQL 16 + pgvector 0.8.7 với cosine exact search; chi phí 0 USD. Báo cáo: [report/individual/2A202603511_DuongNgocTuyen.md](report/individual/2A202603511_DuongNgocTuyen.md).

| Strategy | Dev Hit@1 | Dev Hit@3 | Test Hit@1 | Test Hit@3 |
|---|---|---|---|---|
| A · fixed 500/50 | 11/19 | 15/19 | 1/6 | 2/6 |
| B · paragraph ≤800 | 14/19 | 19/19 | 4/6 | 5/6 |
| **C · B + chapter filter (chọn)** | 17/19 | 19/19 | 4/6 | 5/6 |

### Chạy lại (Windows PowerShell; macOS/Linux đổi lệnh kích hoạt venv)

Yêu cầu: Docker Desktop và Python 3.11–3.14. Chạy từ thư mục `starter/`:

```powershell
cd starter
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
Copy-Item .env.example .env
docker compose -p day6_student_starter up -d --wait
python scripts/check_setup.py          # Setup OK
python scripts/download_corpus.py      # tải PDF (24,6 MB) và kiểm tra SHA-256
python -m src.run all                  # prepare -> index -> evaluate
python scripts/summarize_audit.py      # tổng hợp nhãn thủ công -> results/audit_summary.json
```

Có thể chạy từng bước: `prepare` (đọc PDF, chuẩn hóa, manifest, xuất chunk), `index` (embed, upsert, kiểm tra nạp lại và mô phỏng tài liệu thay đổi), `evaluate` (kiểm tra benchmark, top-3 cho A/B/C, trace, chỉ số, file audit). Lần đầu chạy sẽ tải model (khoảng 64 MB) vào `starter/.cache/`. Nếu chạy lại `index` trên database đã có dữ liệu, mọi chunk được bỏ qua vì hash không đổi (đó chính là kiểm tra nạp lại). Muốn đo lại thời gian nạp lần đầu, xóa bảng trước: `docker compose -p day6_student_starter exec -T postgres psql -U day6 -d day6_lab -c "DROP TABLE chunks;"`.

| Đường dẫn | Nội dung |
|---|---|
| [starter/src/](starter/src/) | Pipeline: `ingestion`, `chunking`, `embeddings`, `store`, `evaluation`, `run`, `config` |
| [data/manifest.json](data/manifest.json), [data/pages.jsonl](data/pages.jsonl) | Manifest (nguồn, giấy phép, trang được tính, số ký tự) và text trang đã chuẩn hóa |
| [benchmark/questions.json](benchmark/questions.json) | 25 câu (19 dev / 6 test), gold page, evidence, filter, tiêu chí chọn đã chốt trước |
| [artifacts/experiment_log.csv](artifacts/experiment_log.csv) | Cấu hình, giả thuyết và kết quả A/B/C |
| [results/](results/) | `ingestion_log.json`, `chunk_stats.json`, `index_report.json`, `metrics.json`, `traces/`, `chunks/`, `audit/`, `audit_summary.json` |

PDF gốc không được commit (đã đưa vào `.gitignore`); `download_corpus.py` tải lại đúng phiên bản nhờ kiểm tra SHA-256.

---

# Day 06 · Data Ingestion Engineering

**Lab cá nhân** · **4 giờ** · **Dữ liệu tự chọn**

Biến tài liệu thành kho tri thức có thể truy xuất cho AI agent, rồi đo xem cách chuẩn bị dữ liệu nào hiệu quả nhất.

| 📄 Dữ liệu | 🧩 Chiến lược | 🎯 Benchmark |
|---|---|---|
| Tối thiểu **30 trang nội dung** | Tối thiểu **3 cấu hình** | Tối thiểu **20 câu hỏi** |

## Bắt đầu nhanh

Dùng [starter PostgreSQL + pgvector](starter/README.md) để dựng database và hoàn thiện các hàm TODO, hoặc tự chọn vector store tương đương. Starter chỉ là khung: **bạn tự hiện thực pipeline và benchmark**.

## 1. Xây pipeline

![Đọc tài liệu, chuẩn hóa, chia chunk, tạo vector, lưu và kiểm tra](assets/ingestion-pipeline.svg)

Chọn tài liệu được phép sử dụng; giữ **ID tài liệu, nguồn, trang/vị trí** và metadata phù hợp. Dùng embedding model thật qua API hoặc chạy local; lưu và truy xuất bằng pgvector, Qdrant, Chroma hoặc một store tương đương.

## 2. So sánh và đánh giá

![Ba chiến lược được đánh giá trên cùng corpus và benchmark dev/test](assets/benchmark-workflow.svg)

Dùng cùng dữ liệu, model và câu hỏi cho cả ba cấu hình. Tách **15 câu dev / 5 câu test** trước khi chạy; chọn cấu hình trên dev, rồi đánh giá test sau khi cố định. Mỗi câu hỏi cần có đoạn bằng chứng và trang nguồn; thiết kế bộ câu hỏi và đánh giá theo [BENCHMARK.md](BENCHMARK.md).

Đánh giá theo năm khía cạnh: **độ chính xác truy xuất**, **tính mạch lạc của chunk**, **tính hữu dụng của metadata**, **chất lượng grounding** và **tác động của chiến lược dữ liệu**. Giải thích lựa chọn bằng kết quả, ví dụ lỗi và các đánh đổi.

## 3. Chuẩn bị bài nộp

1. Chạy pipeline và lưu cấu hình, kết quả top-3, trace cùng nhãn đánh giá thủ công.
2. Điền [thông tin học viên](STUDENT.md), [nhật ký thí nghiệm](artifacts/experiment_log.csv) và báo cáo cá nhân [2A202603511_DuongNgocTuyen.md](report/individual/2A202603511_DuongNgocTuyen.md).
3. Kiểm tra mã nguồn, cách chạy, manifest dữ liệu và checklist trong [SUBMISSION.md](SUBMISSION.md).

## Tài liệu hướng dẫn

| Bạn cần tìm | Tài liệu |
|---|---|
| Setup baseline PostgreSQL + pgvector | [starter/README.md](starter/README.md) |
| Các bước làm và cách tính chỉ số | [GUIDE.md](GUIDE.md) |
| Tiến độ trong 4 giờ | [CHECKPOINTS.md](CHECKPOINTS.md) |
| Năm tiêu chí benchmark và bộ câu hỏi | [BENCHMARK.md](BENCHMARK.md) |
| Tiêu chí chấm điểm | [RUBRIC.md](RUBRIC.md) |
| Quy tắc sử dụng dữ liệu, AI và API | [RULES.md](RULES.md) |
| Cách nộp bài | [SUBMISSION.md](SUBMISSION.md) |

[data/format-example.json](data/format-example.json) chỉ minh họa định dạng, không phải corpus đủ điều kiện. Phần xử lý và benchmark do bạn tự hiện thực; không bắt buộc xây chatbot.
