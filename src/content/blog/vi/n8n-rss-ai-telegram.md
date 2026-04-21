---
title: "[Workflow Automation Fundamentals] Setup và triển khai n8n: RSS → AI Summary → Telegram"
pubDate: 2026-04-21
heroImage: "../_shared/n8n-rss-ai-telegram/hero.png"
description: "Thiết lập một workflow n8n self-hosted để lấy tin từ RSS, rút gọn dữ liệu bằng JavaScript, tóm tắt bằng Ollama và gửi thông báo sang Telegram."
lang: "vi"
tags: ["n8n", "workflow", "automation", "rss", "ollama", "telegram", "javascript", "self-hosted"]
---

**Tác giả:** Felix Doan  

---

> Xây dựng một workflow automation self-hosted với n8n để lấy tin tức từ RSS feed, tóm tắt nội dung bằng AI và gửi kết quả vào Telegram cá nhân.

## 1. Tóm tắt (TL;DR)

Bài viết này ghi lại quá trình dựng một workflow automation cơ bản trong n8n theo mô hình:

![image-24](../_shared/n8n-rss-ai-telegram/image24.png)

Mục tiêu của flow là:
- Theo dõi một RSS feed chính thức.
- Lấy các trường dữ liệu quan trọng từ bài viết mới.
- Tóm tắt nội dung bằng AI chạy qua Ollama self-hosted.
- Gửi bản tóm tắt sang Telegram cá nhân.

Phiên bản ban đầu dùng nguồn test để kiểm tra kỹ thuật, sau đó được cập nhật sang feed chính thức của BBC để chạy thực tế ổn định hơn.

## 2. Bài toán thực hành (The Practice Goal)

Mục tiêu của bài thực hành ngày đầu với n8n không phải là xây một hệ thống phức tạp, mà là hiểu 4 thành phần cốt lõi của workflow automation:

- **Trigger:** điều gì làm flow bắt đầu chạy.
- **Data transformation:** dữ liệu đi qua từng node được làm sạch và rút gọn như thế nào.
- **AI processing:** nội dung được đưa sang model để xử lý ra sao.
- **Delivery:** kết quả cuối cùng được gửi tới đâu.

Bài toán cụ thể ở đây là: theo dõi tin tức từ một RSS feed, lấy những trường quan trọng nhất, tóm tắt lại bằng AI và gửi sang Telegram để người dùng nhận thông báo tự động.

## 3. Phạm vi triển khai (Scope)

Workflow hiện tại tập trung vào một use case tối giản nhưng hoàn chỉnh:

- Chạy trên **n8n self-hosted**.
- Dùng **RSS Feed Trigger** để theo dõi nguồn tin.
- Dùng **Code node (JavaScript)** để trích xuất dữ liệu cần thiết.
- Dùng **Ollama** trên server cá nhân để chạy model AI.
- Dùng **Telegram Bot** để nhận thông báo.

Các phần **chưa ưu tiên** ở giai đoạn này:

- Không xử lý phân loại tin theo chủ đề phức tạp.
- Không chống trùng lặp nâng cao ngoài logic mặc định của RSS trigger.
- Không lưu trữ lịch sử bài đã gửi vào database.
- Không tối ưu prompt engineering chuyên sâu.

## 4. Kiến trúc workflow (Workflow Architecture)

Flow hoàn chỉnh:

```text
RSS Feed Trigger -> Code -> Message a model -> Telegram
```

### 4.1. RSS Feed Trigger

![image-02](../_shared/n8n-rss-ai-telegram/image2.png)


Node đầu tiên chịu trách nhiệm theo dõi RSS feed và khởi động workflow khi có item phù hợp.

Trong quá trình hoàn thiện, nguồn test ban đầu được thay bằng feed chính thức ổn định hơn:

![image-03](../_shared/n8n-rss-ai-telegram/image3.png)

RSS output trả về khá đầy đủ dữ liệu, ví dụ:
- `title`
- `link`
- `pubDate`
- `content`
- `contentSnippet`
- `isoDate`

Điều này rất phù hợp để làm nguồn vào cho AI.


### 4.2. Code node

![image-05](../_shared/n8n-rss-ai-telegram/image5.png)

Dữ liệu RSS gốc thường có nhiều key không cần thiết cho bài toán tóm tắt. Thay vì đẩy nguyên payload sang model, workflow dùng một node Code để rút gọn dữ liệu.

![image-04](../_shared/n8n-rss-ai-telegram/image4.png)

Logic hiện tại:
- lấy item đầu tiên trong tập dữ liệu đầu vào,
- chỉ giữ các trường quan trọng,
- chuẩn hóa ngày và nội dung để các node sau dùng dễ hơn.

![image-06](../_shared/n8n-rss-ai-telegram/image6.png)

Output sẽ thu được trông như này:

![image-07](../_shared/n8n-rss-ai-telegram/image7.png)

### 4.3. AI node qua Ollama

![image-08](../_shared/n8n-rss-ai-telegram/image8.png)

Sau khi dữ liệu được làm gọn, node AI sẽ nhận phần input đã chuẩn hóa để tóm tắt nội dung.

Workflow này dùng **Ollama self-hosted** nên chỉ cần:
- IP hoặc hostname của server Ollama,
- cổng mặc định `11434`,
- không cần API key.

![image-09](../_shared/n8n-rss-ai-telegram/image9.png)

Model dùng để test:
- `qwen2.5-coder:0.5b`

Đây là model nhẹ, phù hợp cho mục tiêu kiểm tra end-to-end flow nhanh, chấp nhận đánh đổi một phần chất lượng output để lấy tốc độ phản hồi.

![image-10](../_shared/n8n-rss-ai-telegram/image10.png)


Prompt tiếng Anh được chỉnh lại để thống nhất với nguồn tin:

![image-11](../_shared/n8n-rss-ai-telegram/image11.png)

Với cấu hình này, output thực tế của node AI trả về khá gọn, chỉ còn một field chính:

![image-12](../_shared/n8n-rss-ai-telegram/image12.png)


Điều này giúp node Telegram map dữ liệu dễ hơn.


### 4.4. Telegram node

Node cuối cùng nhận dữ liệu từ AI và gửi về Telegram qua bot cá nhân.

Trước khi dùng được node này, cần chuẩn bị:
- tạo bot bằng `@BotFather`,

![image-13](../_shared/n8n-rss-ai-telegram/image13.png)

- lấy bot token,
- lấy chat ID cá nhân qua `@userinfobot`.

![image-16](../_shared/n8n-rss-ai-telegram/image16.png)


Sau khi có đủ thông tin, chỉ cần cấu hình Telegram node và map nội dung cần gửi.

![image-19](../_shared/n8n-rss-ai-telegram/image19.png)


Cách map đúng trong flow hiện tại:
- summary lấy từ AI node: `{{$json.content}}`
- title, link, pubDate có thể lấy lại từ node Code bằng cú pháp tham chiếu node.

Ví dụ message hoàn chỉnh:

![image-20](../_shared/n8n-rss-ai-telegram/image20.png)

Kiểm tra từ phía telegram có thông báo chưa

![image-22](../_shared/n8n-rss-ai-telegram/image22.png)

Xem nội dung chi tiết 

![image-23](../_shared/n8n-rss-ai-telegram/image23.png)

## 5. Điều quan trọng về trigger: “Every Minute” không có nghĩa là spam mỗi phút

Một điểm dễ gây nhầm khi mới dùng n8n là phần `Poll Times`.

Trong workflow này, `Mode = Every Minute` có nghĩa là:
- cứ mỗi phút node RSS sẽ **kiểm tra** feed một lần,
- nhưng chỉ khi có item mới phù hợp thì flow mới xử lý tiếp và gửi Telegram.

Nó **không** có nghĩa là cùng một bài sẽ bị gửi lại mỗi phút.

Sau khi workflow được kích hoạt và theo dõi trong mục **Executions**, hệ thống đã cho thấy nhiều lần chạy thành công liên tiếp. Điều này xác nhận:
- trigger đang hoạt động,
- workflow self-hosted chạy nền ổn định,
- luồng từ RSS đến Telegram đã hoàn thiện end-to-end.

![image-25](../_shared/n8n-rss-ai-telegram/image25.png)

## 6. Hướng mở rộng tiếp theo (Next Iterations)

Sau khi flow cơ bản đã chạy ổn định, có thể mở rộng theo các hướng sau:

- **Lấy nhiều bài hơn:** thay vì chỉ lấy `firstItem`, có thể dùng `slice(0, 3)` để xử lý 3 bài mới nhất.
- **Định dạng Telegram đẹp hơn:** thêm bullet, emoji hoặc chia block rõ hơn.
- **Đổi nguồn RSS theo chủ đề:** business, technology, world news...
- **Thêm filter node:** chỉ gửi các bài chứa keyword nhất định.
- **Thêm storage/logging:** ghi lại những bài đã gửi để tránh trùng lặp chủ động hơn.
- **Chuyển sang Schedule Trigger:** nếu muốn ép workflow chạy lại theo chu kỳ, bất kể item có mới hay không.

## 7. Kết luận

Workflow này đã đạt được mục tiêu thực hành ban đầu:
- dùng n8n self-hosted để kết nối các node,
- lấy dữ liệu từ một RSS feed thật,
- xử lý JSON bằng JavaScript,
- gọi AI cục bộ qua Ollama,
- và gửi kết quả sang Telegram.

Quan trọng hơn, nó giúp hình dung rõ cấu trúc cơ bản của một workflow automation hiện đại: **nguồn vào -> xử lý dữ liệu -> AI -> kênh phân phối**.

Đây là một nền tảng rất tốt để tiếp tục học các flow phức tạp hơn như chatbot, content pipeline, alerting system hoặc AI agent orchestration.
