# Lộ trình học RabbitMQ – 2 tuần (Node.js)

> **Lưu ý:** Lộ trình gốc bạn đưa ra gồm 6 giai đoạn (Foundation → Patterns → Reliability → Production → Case Study → EIP), tổng thời lượng thực tế thường mất 2–3 tháng nếu học kỹ và code đầy đủ. Bản rút gọn này tập trung vào **Giai đoạn 1–3** (phần quan trọng nhất để đi làm/phỏng vấn), cộng một case study nhỏ ở cuối. Giai đoạn 4 (Production/Clustering) và 6 (EIP) nên học tiếp sau khi xong 2 tuần này, khi có nhiều thời gian hơn.

**Nguồn chính (đọc trực tiếp các trang này, đừng chỉ đọc mục lục):**

- Tutorials tổng: https://www.rabbitmq.com/tutorials
- AMQP 0-9-1 Concepts: https://www.rabbitmq.com/tutorials/amqp-concepts
- Connections: https://www.rabbitmq.com/docs/connections
- Channels: https://www.rabbitmq.com/docs/channels
- Queues: https://www.rabbitmq.com/docs/queues
- Publishers/Exchanges: https://www.rabbitmq.com/docs/publishers
- Consumers: https://www.rabbitmq.com/docs/consumers
- Consumer Acknowledgements & Publisher Confirms: https://www.rabbitmq.com/docs/confirms
- Consumer Prefetch: https://www.rabbitmq.com/docs/consumer-prefetch
- TTL: https://www.rabbitmq.com/docs/ttl
- Dead Letter Exchanges: https://www.rabbitmq.com/docs/dlx
- CloudAMQP Blog (loạt bài "RabbitMQ for beginners"): https://www.cloudamqp.com/blog

**Công cụ:** Node.js (thư viện `amqplib`), Docker Compose để chạy RabbitMQ + Management UI.

**Quy ước dùng chung cho mọi ngày:**

- Mỗi ngày = 1 thư mục riêng, ví dụ `day01-hello-world/`, `day02-connections/`...
- Mỗi ngày đều có: 1 file note (`notes.md` — tự tóm tắt bằng lời mình, không copy docs), code chạy được, và trả lời câu hỏi tự kiểm tra ngay trong `notes.md`.
- "Input" = thứ bạn có sẵn trước khi bắt đầu ngày đó. "Output" = thứ phải có trong tay khi kết thúc ngày đó (file, kết quả quan sát được, câu trả lời).

---

## Chuẩn bị (30 phút – trước khi bắt đầu)

- Cài Docker Desktop.
- Tạo `docker-compose.yml`:

```yaml
version: '3.8'
services:
    rabbitmq:
        image: rabbitmq:3-management
        ports:
            - '5672:5672' # AMQP
            - '15672:15672' # Management UI
        environment:
            RABBITMQ_DEFAULT_USER: guest
            RABBITMQ_DEFAULT_PASS: guest
```

- `docker compose up -d` → mở `http://localhost:15672` (user/pass: guest/guest).
- `npm init -y && npm install amqplib`

---

## TUẦN 1 – Foundation + Messaging Patterns

### Ngày 1: RabbitMQ Overview + AMQP 0-9-1 Concepts

**Đọc (theo đúng thứ tự, ~1.5–2h):**

1. https://www.rabbitmq.com/tutorials — đọc phần giới thiệu, không cần code ngày này.
2. https://www.rabbitmq.com/tutorials/amqp-concepts — đọc kỹ toàn bộ, đây là bài quan trọng nhất tuần 1. Chú ý các đoạn: "Messages are published to exchanges", "AMQP entities" (queue/exchange/binding), phần Exchange Types (Direct/Fanout/Topic/Headers), phần Acknowledgements, phần Virtual Hosts.

**Input:** Chưa có gì, máy đã cài Docker + Node.js theo phần Chuẩn bị.

**Việc phải làm:**

1. Vừa đọc vừa vẽ tay (giấy hoặc draw.io) sơ đồ: `Publisher → Exchange → Binding → Queue → Consumer`. Đây là sơ đồ bạn sẽ dùng lại suốt lộ trình.
2. Tạo project: `mkdir day01-hello-world && cd day01-hello-world && npm init -y && npm install amqplib`
3. Viết `send.js`: kết nối tới `amqp://localhost`, tạo channel, `assertQueue('hello', { durable: false })`, gửi 1 message text bằng `channel.sendToQueue('hello', Buffer.from('Hello World'))`.
4. Viết `receive.js`: kết nối, `assertQueue('hello', ...)`, `channel.consume('hello', msg => console.log(msg.content.toString()))`.
5. Chạy `node receive.js` ở 1 terminal, `node send.js` ở terminal khác → phải thấy log "Hello World".
6. Mở `http://localhost:15672` (guest/guest), vào tab **Queues**, tìm queue `hello`, quan sát số message tăng/giảm theo thời gian thực khi bạn chạy send/receive.

**Output (phải có cuối ngày):**

- Ảnh chụp hoặc file sơ đồ Publisher → Exchange → Binding → Queue → Consumer.
- 2 file `send.js`, `receive.js` chạy được, có log chứng minh message đi qua.
- Note ngắn trong `notes.md`: định nghĩa bằng lời của riêng bạn cho 5 khái niệm: Broker, Connection, Channel, Exchange, Binding.

**Tự kiểm tra:**

- Message thực chất được gửi tới queue hay tới exchange? (gợi ý: đọc lại phần "default exchange" trong AMQP Concepts)

---

### Ngày 2: Connections, Channels, Virtual Hosts

**Đọc:**

- https://www.rabbitmq.com/docs/connections — chú ý phần vì sao connection là TCP nặng, nên hạn chế mở nhiều connection.
- https://www.rabbitmq.com/docs/channels — chú ý phần channel là "lightweight connection" chạy multiplex trên 1 TCP connection.
- Phần Virtual Hosts trong https://www.rabbitmq.com/tutorials/amqp-concepts (đã đọc hôm qua, đọc lại đoạn vhost).

**Input:** Code `send.js`/`receive.js` của Ngày 1.

**Việc phải làm:**

1. Tạo `day02-connections/connection.js` — module dùng chung, export hàm `getChannel()` mở 1 connection rồi mở channel từ đó (không mở connection mới mỗi lần gọi).
2. Sửa lại `send.js` để dùng module này, viết thêm 1 file `send-multi-channel.js` mở 2 channel khác nhau từ cùng 1 connection, gửi 2 loại message riêng biệt qua 2 channel để thấy chúng độc lập nhau.
3. Vào Management UI → tab **Admin > Virtual Hosts** → tạo vhost mới tên `test-vhost`. Gán quyền cho user `guest` vào vhost này.
4. Sửa connection string thành `amqp://guest:guest@localhost:5672/test-vhost`, chạy lại `send.js`/`receive.js`, xác nhận queue `hello` ở vhost mới không liên quan gì tới queue `hello` ở vhost `/` mặc định.

**Output:**

- `connection.js` tái sử dụng được.
- 1 vhost mới `test-vhost` hoạt động độc lập, kiểm chứng bằng cách thấy 2 queue `hello` riêng biệt trên Management UI (mỗi vhost 1 queue, cùng tên nhưng khác dữ liệu).
- Trả lời trong `notes.md`.

**Tự kiểm tra:**

- Vì sao nên tái sử dụng connection nhưng mở nhiều channel thay vì mở nhiều connection?
- Vhost dùng để làm gì trong thực tế (gợi ý: tách môi trường dev/staging/prod hoặc tách theo team trên cùng 1 cụm RabbitMQ)?

---

### Ngày 3: Queues, Exchanges, Bindings, Routing Keys

**Đọc:**

- https://www.rabbitmq.com/docs/queues — phần khai báo queue, các thuộc tính (durable, exclusive, auto-delete, arguments).
- https://www.rabbitmq.com/docs/publishers — phần Exchanges và Routing (đọc đoạn "Routing in AMQP 0-9-1 is performed by exchanges").

**Input:** Module `connection.js` từ Ngày 2.

**Việc phải làm:**

1. Tạo `day03-exchange-binding/`. Viết `setup.js`: `assertExchange('logs_direct', 'direct', { durable: true })`, tạo 2 queue `queue-error` và `queue-info`, bind `queue-error` với routing key `error`, bind `queue-info` với routing key `info`.
2. Viết `producer.js`: nhận routing key và nội dung từ `process.argv`, publish vào `logs_direct` với routing key tương ứng (`node producer.js error "DB down"`).
3. Viết `consumer-error.js` và `consumer-info.js`: mỗi file consume đúng 1 queue tương ứng.
4. Chạy thử: gửi routing key `error` → chỉ `consumer-error.js` nhận được, không phải `consumer-info.js`.
5. Vào Management UI → tab **Exchanges** → click vào `logs_direct` → xem sơ đồ Bindings trực quan (RabbitMQ vẽ sẵn cho bạn).

**Output:**

- 1 exchange, 2 queue, 2 binding hoạt động đúng — chứng minh bằng log console cho thấy message đi đúng consumer theo routing key.
- Ảnh chụp màn hình sơ đồ Bindings trên Management UI.

**Tự kiểm tra:**

- Nếu bind cả `queue-error` và `queue-info` cùng routing key `error`, chuyện gì xảy ra khi publish message với key `error`?
- Routing key hoạt động khác gì giữa Direct Exchange và Fanout Exchange (Fanout có đọc routing key không)?

---

### Ngày 4: Producer & Consumer, Acknowledgements

**Đọc:**

- https://www.rabbitmq.com/docs/consumers — toàn bộ, chú ý phần Acknowledgement Modes.
- https://www.rabbitmq.com/docs/confirms — chỉ đọc phần đầu về Consumer Acknowledgements (basic.ack/basic.nack/basic.reject), phần Publisher Confirms để dành tới Ngày 10.

**Input:** Code exchange/binding từ Ngày 3.

**Việc phải làm:**

1. Tạo `day04-ack/consumer-autoack.js`: consume với `{ noAck: true }`, cố tình `throw Error` giữa xử lý (không catch), quan sát: message đã bị xoá khỏi queue dù xử lý lỗi.
2. Tạo `consumer-manualack.js`: consume với `{ noAck: false }`, xử lý xong mới gọi `channel.ack(msg)`. Thử: tắt process (`Ctrl+C`) giữa lúc đang xử lý (thêm `setTimeout` giả lập việc xử lý chậm) trước khi kịp ack → mở lại consumer, quan sát message có quay lại queue để xử lý lại không.
3. Ghi lại kết quả quan sát (số message trong queue trước/sau mỗi lần test) bằng cách nhìn tab Queues trên Management UI, cột **Ready** và **Unacked**.

**Output:**

- Bảng so sánh (viết trong `notes.md`) 2 cột: "Auto-ack" vs "Manual-ack", với ít nhất 2 tình huống thực nghiệm và kết quả quan sát được (mất message hay không).

**Tự kiểm tra:**

- Vì sao auto-ack có thể làm mất message khi consumer crash?
- Cột "Unacked" trên Management UI thể hiện điều gì, và tại sao con số này quan trọng khi debug production?

---

### Ngày 5: Durability & Persistent Messages

**Đọc:**

- https://www.rabbitmq.com/docs/queues — phần Queue Durability.
- https://www.rabbitmq.com/docs/publishers — phần message properties (`deliveryMode`/`persistent`).

**Input:** Code Ngày 4.

**Việc phải làm:**

1. Tạo `day05-durability/`. Tạo 4 tổ hợp để so sánh:
    - Queue durable=false, message persistent=false
    - Queue durable=true, message persistent=false
    - Queue durable=false, message persistent=true
    - Queue durable=true, message persistent=true
2. Với mỗi tổ hợp: publish 3 message vào queue tương ứng, KHÔNG chạy consumer, sau đó `docker compose restart rabbitmq`.
3. Sau khi container lên lại, mở Management UI kiểm tra từng queue: còn tồn tại không, còn bao nhiêu message.
4. Ghi kết quả vào bảng trong `notes.md` (4 dòng, cột: Queue durable? / Message persistent? / Còn sau restart?).

**Output:**

- Bảng thực nghiệm 4 dòng như trên, tự rút ra kết luận.

**Tự kiểm tra:**

- Durable queue mà message không persistent thì sau restart có mất dữ liệu không? Vì sao (gợi ý: durable chỉ đảm bảo bản thân queue được khai báo lại, không đảm bảo nội dung message)?
- Persistent message có đảm bảo 100% không mất dữ liệu không? (gợi ý: đọc thêm về cơ chế fsync theo batch trong docs/confirms — vẫn có cửa sổ rủi ro nhỏ nếu broker chết đúng lúc chưa kịp ghi đĩa)

---

### Ngày 6: Work Queue + Competing Consumers

**Đọc:**

- https://www.rabbitmq.com/docs/consumer-prefetch — đọc kỹ, đây là khái niệm hay bị hiểu sai.

**Input:** Code Ngày 5 (dùng lại queue durable).

**Việc phải làm:**

1. Tạo `day06-work-queue/`. Viết `producer.js` gửi 20 message, mỗi message có nội dung là 1 số biểu thị "độ khó" (VD: số dấu chấm `.` càng nhiều càng lâu xử lý), dùng `persistent: true`.
2. Viết `worker.js` nhận tham số tên worker qua argv, xử lý bằng cách `setTimeout` theo độ khó, sau đó `channel.ack`. Chạy 3 instance: `node worker.js W1`, `node worker.js W2`, `node worker.js W3`.
3. Chạy `producer.js` gửi 20 message, quan sát log 3 worker — mặc định RabbitMQ chia round-robin bất kể worker nào đang bận.
4. Thêm `channel.prefetch(1)` vào mỗi worker, chạy lại — quan sát: worker nào rảnh trước sẽ được nhận message tiếp theo (fair dispatch), khác với round-robin cứng nhắc.

**Output:**

- 2 lần log so sánh: không có prefetch vs có `prefetch(1)`, chỉ rõ sự khác biệt về cách chia việc.

**Tự kiểm tra:**

- Vì sao Work Queue bắt buộc cần Ack (không dùng auto-ack)? Điều gì xảy ra nếu 1 worker chết giữa chừng khi đang xử lý mà không có ack?
- Nếu để `prefetch` mặc định (unlimited) với nhiều consumer, rủi ro gì có thể xảy ra (gợi ý: 1 consumer chậm vẫn bị dồn hết message, các consumer nhanh rảnh việc)?

---

### Ngày 7: Publish/Subscribe – Fanout, Direct, Topic, Headers Exchange

**Đọc:**

- https://www.rabbitmq.com/tutorials/amqp-concepts — đọc lại kỹ đúng 4 đoạn: Direct exchange, Fanout exchange, Topic exchange, Headers exchange.

**Input:** Module `connection.js` dùng xuyên suốt.

**Việc phải làm — code 4 ví dụ nhỏ, mỗi ví dụ có 1 producer + ít nhất 2 consumer khác nhau để thấy rõ khác biệt:**

1. `day07a-fanout/`: exchange `notifications` (fanout). 2 consumer (`consumer-email.js`, `consumer-sms.js`) đều nhận được mọi message, không cần routing key.
2. `day07b-direct/`: exchange `logs` (direct). Consumer bind theo đúng 1 key (`error` hoặc `warning`) — dùng lại ý tưởng Ngày 3 nhưng thêm mức `warning`.
3. `day07c-topic/`: exchange `order_events` (topic). Routing key dạng `order.<region>.<action>`, VD `order.vn.created`. Consumer A bind pattern `order.*.created` (mọi vùng, chỉ sự kiện created). Consumer B bind pattern `order.vn.#` (mọi sự kiện của riêng VN).
4. `day07d-headers/`: exchange `tasks` (headers). Publish message kèm headers `{ format: 'pdf', type: 'report' }`. Consumer bind với `x-match: all` yêu cầu cả 2 điều kiện khớp mới nhận.

**Output:**

- 4 thư mục code chạy được, mỗi cái có ít nhất 1 đoạn log chứng minh routing đúng như kỳ vọng (VD: Consumer A không nhận message `order.vn.paid` nhưng Consumer B thì có).

**Tự kiểm tra (bắt buộc trả lời được, đây là câu hay bị hỏi khi phỏng vấn):**

- Tại sao dùng Topic thay vì Direct? (gợi ý: khi cần match theo pattern/nhiều tiêu chí thay vì 1 key cố định)
- Khi nào dùng Fanout? (gợi ý: broadcast, mọi consumer đều cần bản sao, không quan tâm nội dung định tuyến)
- Headers Exchange giải quyết vấn đề gì mà Topic không làm được? (gợi ý: khi tiêu chí lọc không thể biểu diễn gọn thành 1 chuỗi routing key phân cấp bằng dấu chấm)

---

## TUẦN 2 – Reliability + Case Study

### Ngày 8: RPC Pattern

**Đọc:**

- https://www.rabbitmq.com/tutorials/tutorial-six-javascript (RPC tutorial, bản Node.js/JavaScript chính thức).

**Input:** Module `connection.js`.

**Việc phải làm:**

1. Tạo `day08-rpc/server.js`: consume queue `rpc_queue`, xử lý xong publish kết quả vào `msg.properties.replyTo` kèm `correlationId: msg.properties.correlationId`, rồi `channel.ack(msg)`.
2. Tạo `client.js`: tạo 1 queue tạm (exclusive, tên do RabbitMQ tự sinh) làm `replyTo`, sinh `correlationId` ngẫu nhiên (VD: `uuid`), publish request vào `rpc_queue` kèm 2 property trên, sau đó consume trên queue tạm, so khớp `correlationId` để lấy đúng response.
3. Chạy thử với 2 client gọi đồng thời (2 terminal), xác nhận mỗi client nhận đúng response của mình, không bị lẫn.

**Output:**

- `server.js` + `client.js` chạy được, log chứng minh 2 client song song không bị trộn kết quả.

**Tự kiểm tra:**

- RPC qua RabbitMQ có phù hợp cho mọi trường hợp không? Khi nào nên tránh (gợi ý: cần độ trễ thấp, đồng bộ chặt — REST/gRPC trực tiếp thường phù hợp hơn; RabbitMQ RPC hợp khi cần tận dụng lại hạ tầng messaging sẵn có hoặc cần decoupling giữa các service)?

---

### Ngày 9: Manual Ack, Nack, Reject, Requeue

**Đọc:**

- https://www.rabbitmq.com/docs/confirms — phần Negative Acknowledgements (basic.nack, basic.reject, tham số requeue).

**Input:** Code Work Queue Ngày 6.

**Việc phải làm:**

1. Tạo `day09-nack/worker.js`: giả lập lỗi ngẫu nhiên (VD: `if (Math.random() < 0.3) throw new Error('fail')`).
2. Case A: khi lỗi, gọi `channel.nack(msg, false, true)` (requeue=true) → quan sát message quay lại đầu/gần đầu queue, có thể bị xử lý lặp vô hạn nếu lỗi luôn xảy ra với message đó cụ thể (poison message).
3. Case B: khi lỗi, gọi `channel.nack(msg, false, false)` (requeue=false) → không có DLX thì message bị drop hẳn (mất), quan sát trong Management UI thấy message biến mất mà không vào đâu cả.
4. Case C: dùng `channel.reject(msg, false)` — so sánh với `nack`, ghi chú lại sự khác biệt (gợi ý: `reject` không hỗ trợ reject nhiều message cùng lúc như `nack`).

**Output:**

- 3 đoạn log tương ứng 3 case, note lại quan sát: message có bị lặp vô hạn (case A), có bị mất (case B) hay không.

**Tự kiểm tra:**

- Requeue vô hạn có nguy cơ gì? (gợi ý: 1 message lỗi cố định làm nghẽn cả queue, chiếm CPU liên tục — chính là "poison message")
- `nack` và `reject` khác nhau ở điểm nào?

---

### Ngày 10: Publisher Confirm + Consumer Prefetch

**Đọc:**

- https://www.rabbitmq.com/docs/confirms — phần Publisher Confirms, đọc kỹ đoạn "basic.ack is sent when a message has been accepted by all the queues... for persistent messages routed to durable queues, this means persisting to disk".
- https://www.rabbitmq.com/docs/consumer-prefetch — đọc lại, lần này chú ý mối liên hệ giữa prefetch và throughput.

**Input:** Code Ngày 5 (durable queue + persistent message).

**Việc phải làm:**

1. Tạo `day10-confirms/publisher.js`: dùng `channel.waitForConfirms()` (hoặc API confirm channel của `amqplib`) sau khi publish 1 loạt message, log ra khi nào broker xác nhận đã ghi nhận toàn bộ.
2. Thử tắt RabbitMQ giữa lúc publish (VD: `docker compose stop rabbitmq` ngay khi vừa gọi publish) để thấy `waitForConfirms` bị treo/lỗi — đây là lúc bạn biết message có thể chưa an toàn.
3. Với worker Ngày 6, thử 3 giá trị `prefetch`: `1`, `10`, `50` — gửi 200 message nhẹ, đo thời gian xử lý hết bằng `console.time`/`console.timeEnd`. So sánh throughput.

**Output:**

- Bảng so sánh thời gian xử lý 200 message theo 3 mức prefetch.
- Note giải thích publisher confirm giúp phát hiện case nào mà durable queue một mình không đảm bảo được.

**Tự kiểm tra:**

- Publisher Confirm giải quyết vấn đề gì mà Durable Queue không giải quyết được? (gợi ý: durable queue đảm bảo message _nếu đã tới broker_ sẽ sống sót qua restart, nhưng không đảm bảo message có _thực sự tới broker_ hay không — đó là việc của Confirm)
- Prefetch quá cao có thể gây vấn đề gì với 1 consumer xử lý chậm?

---

### Ngày 11: TTL + Dead Letter Exchange + Dead Letter Queue

**Đọc:**

- https://www.rabbitmq.com/docs/ttl — cả Queue TTL (`x-message-ttl`) và Per-Message TTL.
- https://www.rabbitmq.com/docs/dlx — đọc kỹ toàn bộ, chú ý các lý do khiến message bị dead-letter: hết TTL, bị `nack`/`reject` với `requeue=false`, hoặc queue vượt `max-length`.

**Input:** Code Ngày 9.

**Việc phải làm:**

1. Tạo `day11-dlx/setup.js`: tạo exchange `dlx` (direct), queue `dead-letters` bind vào `dlx` với key `failed`.
2. Tạo queue chính `orders` với arguments: `x-dead-letter-exchange: 'dlx'`, `x-dead-letter-routing-key: 'failed'`, `x-message-ttl: 10000` (10 giây).
3. Publish 1 message vào `orders`, KHÔNG chạy consumer, đợi 10 giây, kiểm tra Management UI: message đã tự chuyển từ `orders` sang `dead-letters` chưa.
4. Viết `consumer-orders.js`: cố tình `channel.nack(msg, false, false)` với vài message cụ thể (VD: message có `id` chẵn) — xác nhận các message này cũng rơi vào `dead-letters`.
5. Viết `consumer-dlq.js`: consume `dead-letters`, in ra header `x-first-death-reason` (RabbitMQ tự gắn header này) để biết message chết vì lý do gì (`expired` hay `rejected`).

**Output:**

- Log chứng minh 2 luồng dead-letter riêng biệt (do TTL và do nack) đều đổ về đúng `dead-letters`, kèm lý do đọc được từ header.

**Tự kiểm tra:**

- DLQ khác gì so với việc tự code try/catch để log lỗi ra file? (gợi ý: DLQ là cơ chế của broker, đảm bảo message không mất ngay cả khi consumer chưa kịp code xử lý lỗi, và cho phép xử lý lại/giám sát tập trung)
- Header `x-first-death-reason` dùng để làm gì trong việc debug production?

---

### Ngày 12: Retry Pattern + Poison Message

**Đọc:**

- CloudAMQP hoặc bất kỳ bài viết nào về "RabbitMQ delayed retry with TTL + DLX" — tìm và đọc 1 bài để tham khảo cách người khác thiết kế queue trung gian có TTL để delay retry.

**Input:** Code Ngày 11.

**Việc phải làm — thiết kế chuỗi 3 exchange/queue để tự làm retry có backoff:**

1. `orders` (queue chính) → khi lỗi, dead-letter sang `retry-exchange`.
2. `retry-5s` (queue trung gian, TTL=5000ms, dead-letter trỏ ngược lại `orders`) — message "nghỉ" 5 giây rồi tự quay lại hàng chính để thử lại.
3. Trong `consumer-orders.js`: đếm số lần retry bằng cách đọc header `x-death` (RabbitMQ tự thêm mảng lịch sử dead-letter mỗi lần), nếu số lần retry vượt quá 3 thì `nack` sang thẳng `dead-letters` (final DLQ) thay vì `retry-exchange`.
4. Test: cho consumer luôn luôn fail với 1 message cụ thể, quan sát nó lặp qua `orders → retry-5s → orders` đúng 3 lần rồi rơi vào `dead-letters` hẳn, không lặp vô hạn.

**Output:**

- Log đầy đủ 1 message đi qua đúng 3 vòng retry rồi dừng lại ở `dead-letters`, kèm số đếm retry in ra mỗi vòng.

**Tự kiểm tra:**

- Vì sao không nên dùng `nack(msg, false, true)` (requeue trực tiếp) làm cơ chế retry? (gợi ý: message quay lại gần như ngay lập tức, không có độ trễ, dễ gây vòng lặp lỗi tốc độ cao chiếm hết CPU — đây chính là "poison message" gây nghẽn queue)
- Retry có giới hạn (3 lần) giải quyết vấn đề gì so với retry vô hạn?

---

### Ngày 13–14: Case Study – Order System (tổng hợp toàn bộ)

Tự thiết kế và code một hệ thống nhỏ mô phỏng **Order Processing**:

- **Exchange:** Topic exchange `order_events`
- **Routing key:** `order.created`, `order.paid`, `order.failed`
- **Queues:**
    - `email-service` (bind `order.created`, `order.paid`)
    - `inventory-service` (bind `order.created`)
    - `payment-service` (bind `order.created`)
- **Reliability áp dụng:**
    - Durable queue + persistent message
    - Manual ack/nack
    - DLX + DLQ cho message lỗi
    - Retry pattern có giới hạn số lần
    - Idempotency: dùng `messageId` để consumer không xử lý trùng nếu nhận lại
- **Monitoring:** dùng Management UI để quan sát queue depth, unacked messages, message rate.

Kết quả cuối: 1 repo Node.js nhỏ chạy được qua `docker compose up`, mô phỏng đầy đủ luồng Order → Payment → Inventory → Email, có retry và DLQ hoạt động thật.

---

## Sau 2 tuần này, học tiếp gì?

- **Giai đoạn 4 (Production):** Quorum Queue, Lazy Queue, Streams, Clustering, HA, Federation, Shovel, Prometheus, TLS — cần môi trường multi-node thật để hiểu sâu, nên dành riêng 1–2 tuần khác.
- **Giai đoạn 6 (EIP):** đọc sau khi đã vững Reliability, lúc đó bảng đối chiếu EIP ↔ RabbitMQ (Fanout = Pub-Sub Channel, Topic = Content-Based Router, DLQ = Dead Letter Channel...) sẽ dễ hiểu và áp dụng ngay được sang Kafka/NATS/ActiveMQ.
