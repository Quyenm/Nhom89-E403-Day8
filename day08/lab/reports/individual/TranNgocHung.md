# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Trần Ngọc Hùng -- 2A202600429
**Vai trò trong nhóm:** Retrieval Owner
**Ngày nộp:** 2026-04-13  

---

## 1. Tôi đã làm gì trong lab này?

Trong lab này tôi phụ trách phần `rag_answer.py` từ dòng **1 đến 291**, tức là toàn bộ phần cấu hình, helper functions, `call_llm()`, `retrieve_dense()`, `retrieve_sparse()` và `retrieve_hybrid()`. Vì vậy công việc của tôi tập trung chủ yếu vào Sprint 2 và phần retrieval tuning đầu Sprint 3. Ở Sprint 2, tôi kết nối ChromaDB bằng `_get_collection()`, viết `call_llm()` để gọi model theo cấu hình `.env`, và implement `retrieve_dense()` để lấy các chunk gần nhất theo embedding. Tôi cũng đảm bảo output retrieval có cấu trúc thống nhất gồm `text`, `metadata` và `score` để các bước sau dùng lại được.

Sang Sprint 3, tôi mở rộng pipeline bằng `retrieve_sparse()` với BM25 và `retrieve_hybrid()` dùng Reciprocal Rank Fusion để kết hợp dense và sparse. Tôi cũng chuẩn hóa một số helper để việc fusion và xử lý score nhất quán hơn trong đúng phạm vi 1-291. Quyết định này xuất phát từ việc một số câu hỏi trong bộ test chứa alias hoặc keyword cụ thể mà dense retrieval đơn thuần không xử lý ổn định. Phần của tôi kết nối trực tiếp với người làm `index.py` vì retrieval chỉ tốt khi chunk và metadata đủ sạch, đồng thời cũng nối với Eval Owner vì scorecard là nơi xác nhận lựa chọn hybrid có thực sự cải thiện hay không.

---

## 2. Điều tôi hiểu rõ hơn sau lab này

Điều tôi hiểu rõ hơn sau lab là retrieval trong RAG không phải một bước “lấy tài liệu” đơn giản, mà là nơi quyết định gần như toàn bộ chất lượng của câu trả lời. Trước đây tôi nghĩ nếu prompt tốt và model đủ mạnh thì vẫn có thể “cứu” được kết quả. Sau khi làm bài này, tôi thấy nếu retrieve sai source thì prompt grounded chỉ giúp model trả lời sai một cách tự tin hơn hoặc buộc model phải abstain. Nghĩa là generation không sửa được lỗi retrieval, nó chỉ khuếch đại chất lượng của context đầu vào.

Khái niệm thứ hai tôi hiểu rõ hơn là vì sao dense và sparse lại bổ sung cho nhau. Dense retrieval mạnh khi câu hỏi và tài liệu dùng ngôn ngữ tự nhiên gần nghĩa nhau. Nhưng khi query chứa alias, tên cũ hoặc cụm từ đặc thù như “Approval Matrix”, BM25 lại hữu ích hơn vì nó match token trực tiếp. Hybrid retrieval không phải vì một phương pháp “tốt hơn hẳn”, mà vì mỗi phương pháp nhìn tài liệu theo một kiểu khác nhau. RRF hoạt động tốt vì nó không ép so sánh score tuyệt đối giữa hai hệ retrieval, mà chỉ tận dụng thứ hạng.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn

Khó khăn lớn nhất của tôi là phân biệt giữa lỗi retrieval thật sự và trường hợp corpus vốn không có câu trả lời. Khi test câu `ERR-403-AUTH`, ban đầu tôi nghi dense search chưa đủ mạnh nên không tìm được tài liệu liên quan. Sau đó khi thêm sparse retrieval, kết quả vẫn không thuyết phục. Lúc này tôi mới nhận ra vấn đề không nằm ở việc truy xuất sai, mà ở chỗ không có document nào trong kho dữ liệu mô tả mã lỗi đó. Nếu không nhìn bài toán theo hướng này, rất dễ lãng phí thời gian tối ưu retrieval cho một câu hỏi vốn phải được xử lý bằng abstain.

Điều làm tôi ngạc nhiên nữa là score của dense và RRF không thể đọc theo cùng một cách. Dense dùng similarity score có ý nghĩa tương đối trực tiếp, còn hybrid dùng RRF nên score rất nhỏ nhưng vẫn có thể đúng. Ban đầu tôi thử nghĩ dùng một threshold chung cho mọi mode, nhưng cách đó không hợp lý. Đây là chỗ khiến tôi hiểu rằng “điểm cao/thấp” trong retrieval chỉ có ý nghĩa khi gắn với đúng cơ chế sinh score của nó.

---

## 4. Phân tích một câu hỏi trong scorecard

**Câu hỏi:** “Approval Matrix để cấp quyền hệ thống là tài liệu nào?”

Đây là câu hỏi tôi thấy thú vị nhất vì nó cho thấy rõ sự khác biệt giữa baseline dense và variant hybrid. Ở baseline, câu hỏi này dễ fail dù thông tin thực ra có trong corpus. Lý do là query dùng tên cũ “Approval Matrix”, trong khi tài liệu thật dùng tên “Access Control SOP”. Về mặt nghĩa, hai cụm này có liên quan, nhưng dense retrieval không phải lúc nào cũng đủ ổn định để map chính xác alias sang tên hiện tại của tài liệu. Kết quả là chunk đúng có thể không đứng đầu, dẫn đến answer mơ hồ hoặc trả lời như thể thiếu dữ liệu.

Với hybrid retrieval, tình hình cải thiện rõ hơn vì BM25 tận dụng được alias đã inject trong content text khi index. Dense vẫn đóng vai trò giữ ngữ nghĩa chung của câu hỏi, còn sparse giúp match đúng từ khóa quan trọng. Sau khi fuse bằng RRF, tài liệu `access_control_sop` có cơ hội lên top cao hơn. Như vậy lỗi chính của baseline nằm ở retrieval chứ không phải generation. Generation chỉ phản ánh chất lượng context đầu vào. Variant hybrid không giải quyết hoàn toàn mọi trường hợp alias phức tạp, nhưng nó là cải tiến hợp lý nhất với chi phí implement thấp và tác động thấy rõ trong scorecard.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì?

Nếu có thêm thời gian, tôi sẽ thử query expansion cho các alias query vì kết quả eval cho thấy hybrid đã cải thiện nhưng chưa xử lý triệt để các câu dùng tên cũ của tài liệu. Cụ thể, tôi muốn thêm một bước rewrite query thành 2-3 biến thể như “Access Control SOP”, “quy trình cấp quyền hệ thống”, hoặc “approval matrix for system access” trước khi retrieve. Ngoài ra, tôi cũng muốn thử một cross-encoder reranker nhẹ để lọc lại top chunk sau hybrid, vì đây là cách trực tiếp để giảm noise mà không cần thay đổi toàn bộ index.
