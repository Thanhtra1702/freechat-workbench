# Mô tả chi tiết dự án RAG

## 1. Tổng quan

Dự án này là một ứng dụng RAG (Retrieval-Augmented Generation) chạy локально, cho phép người dùng:

- Tạo nhiều phiên chat độc lập.
- Tải lên tài liệu và trò chuyện trực tiếp với nội dung trong tài liệu.
- Dùng tìm kiếm ngữ nghĩa để lấy đúng đoạn liên quan trước khi sinh câu trả lời.
- Phân tích tài liệu dạng PDF và ảnh theo bố cục, OCR, bảng biểu và hình ảnh.
- Hỏi đáp riêng trên từng tài liệu đã được parse.
- Ghi âm giọng nói và chuyển thành văn bản để chat.
- Xem tiến trình upload và xử lý tài liệu theo thời gian thực.

Ứng dụng được thiết kế để chạy hoàn toàn cục bộ với Ollama, ChromaDB và các model xử lý ngôn ngữ/thị giác được cài trên máy của người dùng.

## 2. Mục tiêu của dự án

Mục tiêu chính của dự án là xây dựng một chatbot tài liệu có các đặc điểm:

- Bảo mật hơn vì dữ liệu không phải gửi lên dịch vụ đám mây.
- Có thể chat theo ngữ cảnh từng phiên làm việc.
- Hỗ trợ nhiều định dạng file phổ biến.
- Tận dụng cả văn bản thô lẫn phân tích bố cục tài liệu để trả lời tốt hơn.
- Có giao diện web đơn giản, dễ dùng và phản hồi nhanh.

## 3. Công nghệ sử dụng

### Backend

- `FastAPI` cho API và WebSocket.
- `Uvicorn` làm ASGI server.
- `Pydantic` cho model request/response.
- `LangChain` để xây dựng chain sinh câu trả lời.
- `ChromaDB` để lưu vector embedding và truy hồi ngữ nghĩa.

### AI / ML

- `Ollama` cho LLM và embedding model.
- `qwen2.5:3b` là model chat mặc định.
- `bge-m3` là model embedding mặc định.
- `glm-ocr:latest` dùng để mô tả hình ảnh cắt từ tài liệu.
- `Faster-Whisper` cho chuyển giọng nói thành văn bản.
- `PaddleOCR-VL-1.6` và các thành phần OCR/layout để phân tích tài liệu PDF và ảnh.

### Frontend

- HTML, CSS, JavaScript thuần.
- `pdf.js` để hiển thị PDF.
- Giao diện một trang với sidebar, khung chat, khu vực parser tài liệu và tab hiển thị kết quả.

## 4. Cấu trúc tổng thể

Luồng hoạt động chính của hệ thống:

1. Người dùng tạo hoặc mở một phiên chat.
2. Người dùng tải tài liệu lên theo phiên.
3. Tài liệu được phân loại theo hai kiểu xử lý:
   - Chỉ trích xuất văn bản và chunk để đưa vào vector store.
   - Parse theo bố cục, OCR và caption hình ảnh nếu bật chế độ parser.
4. Nội dung tài liệu được chia nhỏ thành các đoạn.
5. Các đoạn được embedding và lưu vào ChromaDB kèm metadata.
6. Khi người dùng đặt câu hỏi, hệ thống tìm các đoạn liên quan theo session và file.
7. Prompt được ghép cùng lịch sử chat và context tài liệu.
8. LLM sinh câu trả lời và gửi về frontend.

## 5. Kiến trúc các thành phần

### `server.py`

Đây là điểm vào chính của backend. File này:

- Khởi tạo FastAPI app.
- Phục vụ frontend tĩnh trong thư mục `static/`.
- Cung cấp API cho:
  - chat
  - WebSocket streaming
  - session management
  - upload file
  - transcribe audio
  - xem, xóa, tải document
  - parse document

### `rag_engine.py`

Đây là lõi RAG của dự án. Nó chịu trách nhiệm:

- Khởi tạo LLM, embeddings, ChromaDB và Whisper.
- Tạo prompt cho chatbot.
- Truy hồi tài liệu liên quan theo session và filename.
- Sinh câu trả lời dạng thường và dạng stream.
- Chuyển giọng nói thành text.
- Quản lý tài liệu trong vector store.
- Quản lý session JSON trên ổ đĩa.

### `file_processor.py`

File này xử lý việc đọc và chunk tài liệu dạng text truyền thống:

- PDF
- DOCX
- TXT
- CSV
- JSON
- XLSX
- Markdown
- PPTX
- Ảnh ở chế độ placeholder khi chưa parse OCR

Sau khi trích xuất text, dữ liệu được cắt thành các chunk bằng `RecursiveCharacterTextSplitter`.

### `document_parser.py`

Đây là module xử lý tài liệu có cấu trúc:

- Hỗ trợ PDF và ảnh.
- Rasterize PDF thành ảnh trang.
- Chạy OCR/layout pipeline để bóc tách block.
- Phân loại block như text, table, figure.
- Cắt ảnh figure và caption bằng model Ollama.
- Tạo output dạng:
  - JSON
  - Markdown
  - danh sách block có bounding box

### `static/index.html`

Giao diện chính của ứng dụng, gồm:

- Sidebar danh sách session.
- Khu chat chính.
- Input bar hỗ trợ text, upload, voice.
- Document Parser view chia đôi màn hình:
  - bên trái là viewer PDF/ảnh/text
  - bên phải là Markdown, JSON và tab chat theo tài liệu

### `static/app.js`

Xử lý toàn bộ tương tác phía client:

- Gọi API.
- Quản lý session trên UI.
- Gửi chat qua WebSocket để stream token.
- Upload file và hiển thị tiến trình.
- Điều khiển trình xem tài liệu parser.
- Chuyển tab, copy, download output.
- Thu âm giọng nói.

## 6. Các tính năng nổi bật

### 6.1 Chat theo từng session

Mỗi cuộc trò chuyện được lưu thành một session riêng trong thư mục `chat_sessions/`. Điều này giúp:

- Tách biệt các cuộc hội thoại.
- Giữ lịch sử chat.
- Cho phép đổi tên session tự động theo câu hỏi đầu tiên.
- Xóa session mà không ảnh hưởng session khác.

### 6.2 Hỏi đáp theo ngữ cảnh tài liệu

Khi một file được upload vào session:

- File được trích xuất thành các chunk.
- Chunk được lưu vào ChromaDB kèm `session_id` và `source`.
- Khi người dùng hỏi, engine lấy context theo session và theo file nếu có.
- Nếu tìm đủ chunk liên quan, chỉ trả những chunk đó.
- Nếu chưa đủ, engine bổ sung thêm các chunk đại diện ở đầu, giữa và cuối tài liệu để có bối cảnh rộng hơn.

### 6.3 Stream câu trả lời

Ứng dụng hỗ trợ trả lời dạng stream qua WebSocket:

- Token được đẩy dần về frontend.
- Người dùng thấy phản hồi xuất hiện ngay khi model đang sinh.
- Khi hoàn tất, session được cập nhật vào file JSON.

### 6.4 Upload và xử lý tài liệu

Người dùng có thể upload tài liệu ngay trong chat hoặc trong Document Parser view.

Hệ thống hỗ trợ:

- PDF
- DOCX
- TXT
- CSV
- JSON
- XLSX
- MD
- PPTX
- PNG, JPG, JPEG, GIF, WEBP, BMP

### 6.5 Document Parser view

Nếu bật parse mode, ứng dụng có thể:

- Hiển thị tài liệu theo trang.
- Vẽ bounding box lên layout.
- Liệt kê từng block được parse.
- Hiển thị JSON parse.
- Hiển thị Markdown được sinh từ cấu trúc tài liệu.
- Cho phép hỏi đáp riêng trên chính tài liệu đó.

### 6.6 Voice input

File âm thanh được gửi lên `/api/transcribe`, sau đó:

- Faster-Whisper chuyển audio thành text.
- Nội dung transcript được đưa vào luồng chat như một tin nhắn người dùng.

## 7. Luồng xử lý tài liệu

### 7.1 Chế độ knowledge-only

Khi upload bình thường:

1. File được lưu vào `uploaded_files/`.
2. `file_processor.process_file()` trích xuất text.
3. Text được cắt chunk.
4. Chunk được embedding.
5. Chunk được lưu vào ChromaDB.
6. Session được cập nhật thông báo upload.

### 7.2 Chế độ parse document

Khi upload với `parse_document=true`:

1. File được lưu vào `uploaded_files/`.
2. `analyze_document()` chạy OCR/layout pipeline.
3. PDF được rasterize sang ảnh trang.
4. Ảnh/trang được phân tích block theo layout.
5. Figure được crop và caption bằng model Ollama.
6. Layout được chuyển thành documents cho RAG.
7. Kết quả parse được cache trong `_layout_cache`.
8. Frontend có thể xem PDF/ảnh, box, JSON, Markdown và hỏi đáp riêng.

## 8. API chính

### Chat

- `POST /api/chat`
- `WS /api/chat/ws`

### Session

- `GET /api/sessions`
- `POST /api/sessions`
- `GET /api/sessions/{session_id}`
- `DELETE /api/sessions/{session_id}`

### Upload và transcribe

- `POST /api/upload`
- `POST /api/transcribe`

### Tài liệu

- `GET /api/documents`
- `DELETE /api/documents`
- `GET /api/documents/download`
- `GET /api/parse`
- `POST /api/parse/chat`

### Thông tin phụ trợ

- `GET /api/supported-formats`

## 9. Dữ liệu lưu trữ cục bộ

Ứng dụng lưu dữ liệu ngay trong workspace:

- `chat_sessions/`: session, lịch sử chat, metadata hội thoại.
- `db/`: dữ liệu vector của ChromaDB.
- `uploaded_files/`: file người dùng tải lên.
- `uploaded_files/_layout_cache/`: cache layout/parse output.
- `tools/`: cache và môi trường hỗ trợ cho PaddleOCR.

Các thư mục runtime này được sinh ra trong quá trình chạy và thường không đưa lên Git.

## 10. Cách cấu hình

Dự án đọc biến môi trường từ file `.env`:

- `OLLAMA_BASE_URL`
- `OLLAMA_FIGURE_CAPTION_MODEL`
- `OMNICHAT_LAYOUT_MODEL`

Giá trị mặc định trong code:

- Ollama server: `http://127.0.0.1:11434`
- Model chat: `qwen2.5:3b`
- Model embedding: `bge-m3`
- Model caption hình: `glm-ocr:latest`
- Whisper: `small`

## 11. Cách chạy

### Cài đặt

```bash
python -m venv venv
pip install -r requirements.txt
```

### Tải model Ollama

```bash
ollama pull qwen2.5:3b
ollama pull bge-m3
ollama pull glm-ocr:latest
```

### Khởi động

```bash
python server.py
```

Sau đó mở:

```text
http://localhost:8000
```

## 12. Điểm mạnh của thiết kế hiện tại

- Chạy local, phù hợp với dữ liệu nội bộ.
- Có cả chat thông thường và parse tài liệu có cấu trúc.
- Tách rõ luồng xử lý: extraction, embedding, retrieval, generation.
- Có streaming nên phản hồi tốt hơn.
- Session và tài liệu được quản lý riêng biệt theo từng cuộc trò chuyện.
- Có cache parse giúp giảm thời gian xử lý lại cùng một file.

## 13. Hạn chế và lưu ý

- Chất lượng trả lời phụ thuộc mạnh vào model Ollama và chất lượng file nguồn.
- OCR/layout của PDF và ảnh có thể tốn thời gian, nhất là lần chạy đầu.
- Một số định dạng cần thư viện hệ thống phù hợp để parse tốt hơn.
- Dữ liệu lưu cục bộ nên cần backup thủ công nếu muốn bảo toàn.
- Khi ChromaDB hoặc model chưa sẵn sàng, một số API sẽ trả lỗi khởi tạo.

## 14. Cấu trúc thư mục

```text
.
├── static/
├── config.py
├── document_parser.py
├── file_processor.py
├── rag_engine.py
├── server.py
├── requirements.txt
├── README.md
├── chat_sessions/
├── db/
└── uploaded_files/
```

## 15. Kết luận

Đây là một dự án RAG hoàn chỉnh theo hướng local-first, kết hợp:

- retrieval ngữ nghĩa,
- streaming chat,
- xử lý tài liệu đa định dạng,
- OCR/layout parsing,
- voice transcription,
- và giao diện web tương tác.

Nếu tiếp tục phát triển, dự án có thể mở rộng thêm:

- phân quyền người dùng,
- upload nhiều file theo batch,
- search trong toàn bộ kho tài liệu,
- export session,
- và các chế độ parse nâng cao hơn cho bảng, biểu đồ, và tài liệu scan.
