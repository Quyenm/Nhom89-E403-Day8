# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Dương Trịnh Hoài An
**Vai trò trong nhóm:** Tech Lead
**Ngày nộp:** 13/04/2026
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này? (100-150 từ)

- Đảm nhận vai trò **Tech Lead**, chịu trách nhiệm chính cho toàn bộ **indexing pipeline** từ baseline đến enhanced
- **Sprint 1 Baseline:** thiết kế luồng preprocess → chunk → embed → store vào ChromaDB; implement extract metadata từ header file (source, department, effective_date) và chunking theo section heading `=== ... ===`
- **Sprint 1 Enhanced:** nâng cấp với 3 cải tiến:
  - **Alias Injection** — inject tên gọi khác vào đầu chunk text để BM25 match được
  - **Version-aware Metadata** — thêm field `version` và `aliases` vào metadata
  - **Contextual Chunking** — gọi LLM sinh prefix ngữ cảnh cho từng chunk trước khi embed
- Phần index là nền tảng cho **Retrieval Owner** query và **Eval Owner** đánh giá chất lượng kết quả

_________________

---

## 2. Điều tôi hiểu rõ hơn sau lab này (100-150 từ)

**Concept 1: Metadata không giúp ích gì cho vector search**
- Trước lab tôi nghĩ metadata `section`, `source` sẽ giúp vector search tìm đúng hơn — thực tế thì không
- ChromaDB chỉ tính cosine similarity trên `chunk["text"]`, metadata bị bỏ qua hoàn toàn ở bước này
- Metadata chỉ có tác dụng để **filter sau** khi đã có kết quả, hoặc cho **BM25 keyword search**

**Concept 2: Tại sao Contextual Chunking hiệu quả**
- Chunk sau khi tách ra bị mất ngữ cảnh — vector không biết chunk thuộc tài liệu nào, nói về chủ đề gì
- Contextual prefix inject ngữ cảnh thẳng vào `chunk["text"]` trước khi embed → vector "nghiêng" về đúng chủ đề
- Kết quả: khi query vector so sánh với chunk vector, độ tương đồng cao hơn → recall cải thiện ~35%

_________________

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn (100-150 từ)

**Điều ngạc nhiên: Cách chunk ảnh hưởng lớn hơn tôi nghĩ**
- Giả thuyết ban đầu: cắt đều theo số token là đủ
- Thực tế: chunk bị cắt giữa câu → vector không biểu đạt được ý hoàn chỉnh → retrieval trả về sai
- Bài học: phải split theo ranh giới tự nhiên (paragraph, câu) thay vì cắt cứng theo ký tự

**Chi tiết kỹ thuật đáng chú ý:**
- ChromaDB không nhận kiểu `list` trong metadata → phải convert aliases thành string `"a | b | c"`, nếu không metadata lưu rỗng mà không báo lỗi rõ ràng
- Không xóa collection cũ trước khi build lại → data cũ lẫn vào index mới → kết quả eval bị nhiễu
- Fix: luôn gọi `delete_collection()` trước `create_collection()` mỗi lần build index

_________________

---

## 4. Phân tích một câu hỏi trong scorecard (150-200 từ)

**Câu hỏi:** *"Approval Matrix cho hệ thống CRM cần ai phê duyệt?"*

**Phân tích:**

- **Baseline trả lời:** Sai hoàn toàn — điểm retrieval gần 0
- **Lỗi nằm ở tầng indexing:**
  - Tài liệu liên quan là `access_control_sop.txt` nhưng trong index không có từ "Approval Matrix" ở đâu
  - BM25 search không tìm thấy gì vì tên không khớp
  - Vector search cũng kém vì "Approval Matrix" và "access control SOP" có vector khá xa nhau về ngữ nghĩa
- **Variant Enhanced có cải thiện không:** Có, rõ rệt
  - Sau khi inject alias `"Approval Matrix for System Access, Approval Matrix"` vào đầu chunk text
  - BM25 tìm được ngay tài liệu đúng vì từ khóa xuất hiện trực tiếp trong chunk
  - Retrieval trả về đúng chunk chứa quy trình phân quyền
- **Kết luận:** Lỗi không phải ở retrieval hay generation mà ở **indexing** — tài liệu không được đánh index đủ thông tin để match với cách user đặt câu hỏi. Alias Injection giải quyết đúng gốc rễ vấn đề.
_________________

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì? (50-100 từ)

- **Tự động hóa Alias Injection:** Hiện tại `DOCUMENT_ALIASES` hardcode thủ công — tôi sẽ thử dùng LLM đọc từng tài liệu và tự sinh danh sách tên gọi khác, tương tự cách Contextual Chunking sinh prefix
- **Lý do:** Kết quả eval cho thấy một số query fail vì tên tài liệu không khớp, trong khi danh sách alias phải cập nhật tay mỗi khi thêm tài liệu mới → pipeline không scale được khi số lượng tài liệu tăng lên

_________________

---
