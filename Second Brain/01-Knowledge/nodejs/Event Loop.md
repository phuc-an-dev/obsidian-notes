---
created: 2026-04-17
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/nodejs"
  - "#topic/async"
related:
  - "[[Non-blocking IO]]"
---

# Event Loop

## 1. What

Event Loop là cơ chế cốt lõi cho phép Node.js thực hiện các thao tác non-blocking I/O mặc dù JavaScript là ngôn ngữ đơn luồng (single-threaded). Nó liên tục giám sát Call Stack và các hàng đợi (queues) để quyết định khi nào thì thực hiện callback tiếp theo. Dưới lớp abstraction của JavaScript, Event Loop được xây dựng trên **libuv** — thư viện C viết vòng lặp sự kiện và quản lý I/O bất đồng bộ đa nền tảng.

## 2. Why

Trong mô hình truyền thống (Java EE trước kia, Python WSGI), mỗi request cần một thread riêng. Khi có 10,000 request đồng thời đều đang chờ kết quả từ database, server cần 10,000 thread — tiêu thụ ~10GB RAM chỉ riêng thread stack. CPU chủ yếu nằm chờ (idle) trong khi các thread đang block chờ I/O. Node.js giải quyết điều này bằng cách dùng **một thread duy nhất** với vòng lặp sự kiện — thread đó không bao giờ bị block; nó chỉ xử lý callback khi I/O xong. Kết quả: Node.js xử lý được hàng nghìn kết nối đồng thời với RAM tối thiểu.

## 3. Mental Model

Hãy tưởng tượng Event Loop là một **Điều phối viên tại tổng đài điện thoại** (call center dispatcher) với một bộ phận xử lý chính (Call Stack) và nhiều hàng chờ khác nhau.

Khi một cuộc gọi mới đến (request), điều phối viên ghi nhận và chuyển cho chuyên gia xử lý (Web API / libuv). Thay vì ngủ trên dây máy chờ chuyên gia xong, điều phối viên quay lại nhận tiếp cuộc gọi mới. Khi chuyên gia xử lý xong, họ đặt kết quả vào một trong các hàng chờ:
- **Hàng VIP (Microtask Queue)**: các cuộc gọi gấp gáp — `Promise.then`, `queueMicrotask`, `process.nextTick`. Điều phối viên xử lý hết hàng này trước khi chuyển sang hàng khác.
- **Hàng thường (Macrotask Queue)**: `setTimeout`, `setInterval`, I/O callbacks — xử lý từng cuộc một theo thứ tự ưu tiên.

Vòng lặp sự kiện chính là quy trình: kiểm tra hàng VIP -> xử lý hết -> lấy một việc từ hàng thường -> xử lý -> kiểm tra hàng VIP lại -> lặp lại. Đây là lý do tại sao Promise callback luôn chạy trước `setTimeout(fn, 0)`.

## 4. Where it fits

```
V8 JavaScript Engine
        |
   [Call Stack]  <--- nơi code đang chạy
        |
   [Event Loop]  <--- điều phối viên
        |
   +----+----+
   |         |
[Microtask  [Macrotask
 Queue]      Phases]
   |              |
[Promise.then  [Timers Phase]     -> setTimeout, setInterval
 process.nextTick  [Pending I/O]  -> hầu hết I/O callbacks
 queueMicrotask]   [Idle/Prepare] -> nội bộ libuv
                   [Poll Phase]   -> lấy I/O events mới, block nếu cần
                   [Check Phase]  -> setImmediate callbacks
                   [Close Phase]  -> socket.on('close', ...)
        |
   [libuv Thread Pool] (4 threads mặc định)
        |
   [OS Kernel: epoll/kqueue/IOCP]
        |
   [Network / Disk / DNS]
```

## 5. When to use

Event Loop luôn hiện diện — bạn không "chọn" dùng hay không. Quan trọng là hiểu nó để:
- Viết code I/O-heavy (REST API, file server, WebSocket) tận dụng tối đa Event Loop.
- Tránh block Event Loop với code CPU-intensive.
- Kiểm soát thứ tự thực hiện của các tác vụ bất đồng bộ: khi nào dùng `process.nextTick`, khi nào dùng `setImmediate`, khi nào dùng `Promise`.
- Debug tình huống "tại sao callback của tôi không chạy?" hoặc "tại sao setTimeout(fn,0) chạy sau promise?".

## 6. When NOT to use

Event Loop **không phù hợp** cho:
- **CPU-intensive tasks**: nhận diện ảnh (image recognition), nén video, mã hóa mật khẩu bằng bcrypt với cost cao, tính toán số học phức tạp. Các tác vụ này chiếm Call Stack, ngăn Event Loop xử lý request khác.
- **Blocking synchronous I/O trong server code**: `fs.readFileSync`, `crypto.pbkdf2Sync` trong request handler sẽ block tất cả request khác.
- **Long-running computation trong single process**: nên tách ra Worker Threads hoặc process riêng.

## 7. Trade-offs

| Pros | Cons |
|------|------|
| Xử lý hàng nghìn kết nối đồng thời với ít thread | CPU-bound task block toàn bộ server |
| RAM thấp (không cần thread mới cho mỗi connection) | Khó debug: stack trace bất đồng bộ khó đọc |
| Đơn giản hơn multi-threading (không có race condition trên shared memory) | Một lỗi không catch có thể crash cả process |
| Latency thấp cho I/O-heavy workload | Không tối ưu cho tính toán nặng |
| Ecosystem npm phong phú cho async patterns | Callback hell nếu không dùng async/await |
| Phù hợp cho microservices, API gateway, BFF | Event Loop lag khó detect trong production |

## 8. Alternatives

| Approach | Ưu điểm | Nhược điểm |
|----------|---------|------------|
| Node.js Event Loop | Đơn giản, I/O nhanh | Không tốt cho CPU work |
| Java Virtual Threads (Java 21) | Viết code blocking nhưng JVM quản lý non-blocking | Cần Java 21, ecosystem khác |
| Go goroutines | Xử lý tốt cả I/O lẫn CPU | Ngôn ngữ khác, learning curve |
| Python asyncio | Tương tự Node nhưng có GIL | GIL giới hạn CPU parallelism |
| Worker Threads (Node.js) | CPU work trên thread riêng, vẫn dùng Node | Overhead communication, chia sẻ state phức tạp |
| Cluster module | Nhiều Node process trên các CPU core | Không chia sẻ memory, phải dùng IPC |

## 9. How

```javascript
// --- Ví dụ 1: Thứ tự thực hiện cơ bản ---
console.log('1. Synchronous start');     // Call Stack

setTimeout(() => {
    console.log('4. setTimeout (macrotask)');
}, 0);

Promise.resolve()
    .then(() => console.log('3. Promise.then (microtask)'));

process.nextTick(() => {
    console.log('2. process.nextTick (runs before other microtasks)');
});

console.log('1b. Synchronous end');     // Call Stack

// Thứ tự in: 1 -> 1b -> 2 -> 3 -> 4
// Giải thích:
//   Call Stack chạy hết (1, 1b)
//   process.nextTick chạy (trước cả Promise.then)
//   Promise.then chạy (phần còn lại của Microtask Queue)
//   setTimeout chạy (Macrotask - Timers phase)


// --- Ví dụ 2: setImmediate vs setTimeout(fn, 0) ---
// Khi chạy trong I/O callback, setImmediate LUÔN chạy trước setTimeout(fn, 0)
const fs = require('fs');

fs.readFile(__filename, () => {
    setTimeout(() => console.log('setTimeout'), 0);
    setImmediate(() => console.log('setImmediate'));
});
// Trong I/O callback: setImmediate -> setTimeout
// Lý do: sau I/O callback, Event Loop ở Poll phase -> tiếp theo là Check phase (setImmediate)
//        -> mới đến lại Timers phase (setTimeout)


// --- Ví dụ 3: Microtask drain trước mỗi macrotask ---
setTimeout(() => console.log('timeout 1'), 0);
setTimeout(() => console.log('timeout 2'), 0);

Promise.resolve().then(() => {
    console.log('promise 1');
    // Thêm microtask mới trong microtask
    Promise.resolve().then(() => console.log('promise 2 (nested)'));
});

// Thứ tự: promise 1 -> promise 2 (nested) -> timeout 1 -> timeout 2
// Microtask Queue bị "drain" hoàn toàn (kể cả microtask mới thêm) trước khi
// bắt đầu macrotask tiếp theo


// --- Ví dụ 4: Detect Event Loop lag ---
const { monitorEventLoopDelay } = require('perf_hooks');

const monitor = monitorEventLoopDelay({ resolution: 20 }); // check mỗi 20ms
monitor.enable();

setInterval(() => {
    const lagMs = monitor.mean / 1e6; // chuyển từ nanoseconds sang ms
    if (lagMs > 100) {
        console.warn(`Event Loop lag: ${lagMs.toFixed(2)}ms`);
        // Alert Prometheus/Datadog tại đây
    }
}, 1000);


// --- Ví dụ 5: Worker Threads cho CPU-intensive task ---
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

if (isMainThread) {
    // Main thread: Event Loop vẫn chạy bình thường
    const worker = new Worker(__filename, {
        workerData: { numbersToProcess: Array.from({length: 1e7}, (_, i) => i) }
    });

    worker.on('message', result => {
        console.log('CPU task done:', result);
    });

    worker.on('error', err => console.error('Worker error:', err));

    // Event Loop vẫn xử lý request khác trong lúc worker tính toán
    setInterval(() => console.log('Main thread still alive'), 500);

} else {
    // Worker thread: tính toán nặng không ảnh hưởng Event Loop chính
    const sum = workerData.numbersToProcess.reduce((a, b) => a + b, 0);
    parentPort.postMessage({ sum });
}


// --- Ví dụ 6: Chia nhỏ CPU task tránh block Event Loop ---
function processInChunks(items, chunkSize, processFn) {
    return new Promise((resolve) => {
        let index = 0;
        const results = [];

        function processChunk() {
            const end = Math.min(index + chunkSize, items.length);
            for (; index < end; index++) {
                results.push(processFn(items[index]));
            }

            if (index < items.length) {
                setImmediate(processChunk); // nhường CPU cho Event Loop xử lý request khác
            } else {
                resolve(results);
            }
        }

        processChunk();
    });
}

// Sử dụng
const items = Array.from({length: 1_000_000}, (_, i) => i);
await processInChunks(items, 10_000, x => x * x);
// Mỗi 10,000 item được xử lý, Event Loop có cơ hội xử lý request HTTP khác


// --- Ví dụ 7: Ứng dụng Express với Event Loop monitoring ---
const express = require('express');
const app = express();

// Middleware theo dõi Event Loop lag cho mỗi request
app.use((req, res, next) => {
    const start = process.hrtime.bigint();
    res.on('finish', () => {
        const durationMs = Number(process.hrtime.bigint() - start) / 1e6;
        if (durationMs > 500) {
            console.warn(`Slow request: ${req.method} ${req.path} took ${durationMs}ms`);
        }
    });
    next();
});

app.get('/users/:id', async (req, res) => {
    try {
        const user = await userRepository.findById(req.params.id); // non-blocking
        res.json(user);
    } catch (err) {
        res.status(500).json({ error: err.message });
    }
});
```

## 10. Production concerns

**Scaling**

- Một Node.js process chỉ chạy trên **một CPU core**. Để sử dụng nhiều core, dùng `cluster` module hoặc chạy nhiều container (Docker + Kubernetes).
- `PM2` với `pm2 start app.js -i max` tự động spawn `N` process tương ứng số CPU core.
- Với microservices, scale horizontal bằng cách deploy nhiều instance sau load balancer (nginx, HAProxy).
- Worker Threads (Node 12+) phù hợp hơn cho CPU work trong cùng một process vì chia sẻ memory qua `SharedArrayBuffer`.

**Failure modes**

- **Unhandled rejection**: từ Node.js 15+, unhandled Promise rejection crash toàn bộ process. Luôn có `process.on('unhandledRejection', ...)` handler và fix root cause.
- **Event Loop starvation**: Microtask queue chạy quá nhiều (ví dụ, mỗi microtask thêm microtask mới để quy) có thể giữ Event Loop mãi mãi trong microtask phase, không bao giờ xử lý macrotask.
- **Memory leak**: closure giữ reference đến object lớn, EventEmitter leak khi không remove listener. Dùng `--expose-gc` và heap profiler để detect.

**Monitoring**

```javascript
// Prometheus metrics với prom-client
const client = require('prom-client');

const eventLoopLag = new client.Histogram({
    name: 'nodejs_event_loop_lag_seconds',
    help: 'Event loop lag in seconds',
    buckets: [0.001, 0.01, 0.1, 0.5, 1, 5]
});

// Đo lag bằng cách lập một setTimeout(0) và đo thời gian thực tế
function measureLag() {
    const start = process.hrtime.bigint();
    setTimeout(() => {
        const lag = Number(process.hrtime.bigint() - start) / 1e9;
        eventLoopLag.observe(lag);
        setTimeout(measureLag, 1000); // lịch lần sau
    }, 0);
}
measureLag();
```

Các công cụ quan trọng:
- `clinic.js` (nearForm): profiling, doctor, flame graph để tìm Event Loop bottleneck.
- `0x` (Nearform): flamegraph CPU profiling.
- `node --inspect`: Chrome DevTools debugging.
- Datadog APM / New Relic: production monitoring Event Loop lag, GC pauses.

## 11. Common mistakes

- Mistake: Gọi `fs.readFileSync`, `crypto.pbkdf2Sync`, hoặc bất kỳ hàm `*Sync` nào trong request handler của HTTP server.
  Fix: Dùng phương thức bất đồng bộ tương đương (`fs.readFile`, `crypto.pbkdf2`) hoặc phiên bản Promise (`fs.promises.readFile`). Mỗi `*Sync` call trong hot path là bug performance nghiêm trọng — nó block Event Loop và làm chết hết request đang chạy.

- Mistake: Hiểu nhầm rằng `setTimeout(fn, 0)` chạy "ngay lập tức" hoặc "trước Promise".
  Fix: `setTimeout(fn, 0)` là macrotask — chạy sau khi Call Stack trống và Microtask Queue đã bị drain hoàn toàn. Nếu cần chạy sau call stack hiện tại nhưng trước I/O events, dùng `process.nextTick` (ưu tiên cao nhất) hoặc `Promise.resolve().then(fn)`.

- Mistake: Thêm event listener mà không remove khi component destroy, dẫn đến memory leak.
  Fix: Lưu tham chiếu đến listener và gọi `emitter.removeListener(event, handler)` hoặc `emitter.off(event, handler)` khi không cần. Với EventEmitter, Node.js cảnh báo khi số listener vượt quá 10 (có thể chỉnh bằng `emitter.setMaxListeners(n)`).

- Mistake: Recursion với `process.nextTick` mà không có điều kiện dừng.
  Fix: `process.nextTick` có ưu tiên cao hơn cả Promise. Nếu `nextTick` callback lặp lại gọi `nextTick`, Event Loop sẽ bị kẹt ở nextTick queue mãi mãi. Đảm bảo có điều kiện thoát rõ ràng.

## 12. Sample project

**Constraint cứng: Xây dựng HTTP log analyzer đọc file log 2GB. Server vẫn phải phản hồi request HTTP khác trong lúc đọc. Không được dùng `readFileSync`. Phải report event loop lag và memory usage theo thời gian thực.**

```javascript
const http = require('http');
const fs = require('fs');
const readline = require('readline');
const { monitorEventLoopDelay } = require('perf_hooks');

// Event Loop monitoring
const lagMonitor = monitorEventLoopDelay({ resolution: 10 });
lagMonitor.enable();

// Trạng thái phân tích
const stats = {
    totalLines: 0,
    errorCount: 0,
    warnCount: 0,
    isProcessing: false,
    progress: 0,
    fileSize: 0,
    bytesRead: 0
};

// Đọc file log theo streaming — không load hết vào memory
async function analyzeLogFile(filePath) {
    if (stats.isProcessing) return;
    stats.isProcessing = true;
    stats.totalLines = 0;
    stats.errorCount = 0;
    stats.warnCount = 0;

    const fileStat = fs.statSync(filePath);
    stats.fileSize = fileStat.size;
    stats.bytesRead = 0;

    const fileStream = fs.createReadStream(filePath, { highWaterMark: 64 * 1024 }); // 64KB chunks
    fileStream.on('data', chunk => { stats.bytesRead += chunk.length; });

    const rl = readline.createInterface({
        input: fileStream,
        crlfDelay: Infinity
    });

    // Xử lý từng dòng mà không block Event Loop
    // readline là EventEmitter — mỗi 'line' là một event, trả lại control cho Event Loop
    for await (const line of rl) {
        stats.totalLines++;
        if (line.includes('ERROR')) stats.errorCount++;
        if (line.includes('WARN'))  stats.warnCount++;
        stats.progress = (stats.bytesRead / stats.fileSize) * 100;

        // Mỗi 50,000 dòng, nhường Event Loop một nhịp
        if (stats.totalLines % 50_000 === 0) {
            await new Promise(resolve => setImmediate(resolve));
        }
    }

    stats.isProcessing = false;
}

// HTTP server vẫn phản hồi trong lúc xử lý file
const server = http.createServer((req, res) => {
    const url = new URL(req.url, 'http://localhost');

    if (req.method === 'POST' && url.pathname === '/analyze') {
        let body = '';
        req.on('data', chunk => { body += chunk; });
        req.on('end', () => {
            const { filePath } = JSON.parse(body);
            analyzeLogFile(filePath).catch(err =>
                console.error('Analysis error:', err)
            );
            res.writeHead(202, { 'Content-Type': 'application/json' });
            res.end(JSON.stringify({ message: 'Analysis started' }));
        });
        return;
    }

    if (req.method === 'GET' && url.pathname === '/status') {
        const eventLoopLagMs = lagMonitor.mean / 1e6;
        const memUsage = process.memoryUsage();

        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
            ...stats,
            eventLoopLagMs: eventLoopLagMs.toFixed(2),
            heapUsedMB: (memUsage.heapUsed / 1024 / 1024).toFixed(1),
            rssMB: (memUsage.rss / 1024 / 1024).toFixed(1)
        }));
        return;
    }

    res.writeHead(404);
    res.end('Not found');
});

server.listen(3000, () => {
    console.log('Log analyzer server running on :3000');
    console.log('POST /analyze { "filePath": "/path/to/log" }');
    console.log('GET  /status  -> realtime stats + event loop lag');
});

// Xử lý lỗi toàn cục
process.on('unhandledRejection', (reason, promise) => {
    console.error('Unhandled Rejection at:', promise, 'reason:', reason);
    // Trong production: gửi alert, graceful shutdown
});
```

## 13. Interview

### Core Q&A

**Q1: Event Loop là gì và nó làm việc như thế nào trong Node.js?**
A: Event Loop là vòng lặp vô hạn kiểm tra xem Call Stack có trống không. Nếu trống, nó lấy callback từ hàng đợi (microtask trước, macrotask sau) và đẩy vào Call Stack để thực thi. Đây là cơ chế cho phép Node.js đơn luồng xử lý I/O bất đồng bộ: trong khi đợi I/O xong, Call Stack trống và Event Loop có thể xử lý request khác.

**Q2: Các phase của Event Loop là gì? Thứ tự là gì?**
A: Các phase của libuv Event Loop theo thứ tự:
1. **Timers**: xử lý `setTimeout` và `setInterval` callback đã hết hạn.
2. **Pending callbacks**: I/O callbacks bị hoãn lại từ vòng trước.
3. **Idle/Prepare**: nội bộ libuv.
4. **Poll**: lấy I/O events mới và thực thi callback. Đây là phase "idle" — nếu không có gì, Event Loop có thể đợi ở đây.
5. **Check**: `setImmediate` callbacks.
6. **Close callbacks**: `socket.on('close', ...)` v.v.

Trước mỗi chuyển pha (và sau mỗi macrotask), **Microtask Queue bị drain hoàn toàn** (nextTick trước, Promise.then sau).

**Q3: Sự khác biệt giữa Microtask Queue và Macrotask Queue?**
A: Microtask Queue (gồm `process.nextTick`, `Promise.then`, `queueMicrotask`) có ưu tiên cao hơn Macrotask Queue (gồm `setTimeout`, `setInterval`, I/O, `setImmediate`). Sau mỗi macrotask (hoặc sau khi Call Stack trống), Event Loop drain toàn bộ Microtask Queue trước khi lấy macrotask tiếp theo. `process.nextTick` chạy trước cả `Promise.then` trong Microtask Queue.

**Q4: `process.nextTick` khác với `setImmediate` như thế nào?**
A: `process.nextTick` là ưu tiên cao nhất — chạy trước cả Promise.then, trước khi Event Loop chuyển sang phase tiếp theo. `setImmediate` chạy ở Check phase, sau I/O callbacks. Sử dụng `process.nextTick` khi cần chạy ngay sau call stack hiện tại (nhưng trước I/O). Sử dụng `setImmediate` khi muốn defer execution sau I/O events.

**Q5: Điều gì xảy ra nếu block Event Loop?**
A: Toàn bộ server "đóng băng" — không một request mới nào được xử lý, không một I/O callback nào được gọi, timeout không chạy đúng giờ. Tất cả client đều nhận timeout. Block Event Loop là lỗi nghiêm trọng nhất trong Node.js production, thường do: `*Sync` functions, vòng lặp `for` quá lớn, đệ quy sâu, `JSON.parse` với payload khổng lồ.

**Q6: Làm sao detect và debug Event Loop lag trong production?**
A: Sử dụng `perf_hooks.monitorEventLoopDelay` để đo lag. Công cụ: `clinic.js doctor` (nearForm) tự động phát hiện bottleneck và gợi ý nguyên nhân. `0x` để tạo flamegraph. Datadog/New Relic APM tracking `eventloop.utilization`. Alert khi lag > 100ms là ngưỡng warning, > 500ms là critical.

**Q7: Worker Threads khác gì Child Process?**
A: Worker Threads chạy trong cùng một process, chia sẻ memory qua `SharedArrayBuffer` và `Atomics`, overhead thấp hơn. Child Process là process riêng biệt với memory riêng, giao tiếp qua IPC (JSON serialization), nặng hơn nhưng an toàn hơn (crash child không ảnh hưởng process chính). Dùng Worker Threads cho CPU-intensive computation cần chia sẻ data; dùng Child Process cho isolation và security.

**Q8: Tại sao `setTimeout(fn, 0)` không đảm bảo chạy "ngay lập tức"?**
A: Hai lý do: (1) Node.js quy định minimum timer resolution là 1ms (thực tế có thể 4-10ms tùy OS). (2) Dù timeout = 0ms, callback vẫn chỉ chạy sau khi Call Stack trống VÀ Microtask Queue đã bị drain hoàn toàn. Nếu có nhiều Promise chain trước đó, `setTimeout(fn, 0)` có thể chạy sau đó rất lâu.

### Scenario

**S1: Server Node.js Express của bạn xử lý upload file lớn và bỗng nhiên tất cả request khác trả về 503. Nguyên nhân có thể là gì?**
A: Nhiều khả năng: (1) Code đang đọc file bằng `fs.readFileSync` hoặc `JSON.parse` trên payload quá lớn — block Event Loop. (2) Đang xử lý file bằng cách ghi hết vào memory rồi xử lý (thay vì stream). Fix: dùng `fs.createReadStream` + `readline` để stream file từng chunk, thêm `setImmediate` delay để nhường Event Loop thường xuyên. Monitor bằng `clinic.js doctor`.

**S2: Viết code in ra "A", "B", "C", "D" theo thứ tự đầy đủ: A = sync, B = nextTick, C = Promise, D = setTimeout.**
A:
```javascript
console.log('A');         // sync, Call Stack
process.nextTick(() => console.log('B')); // nextTick queue
Promise.resolve().then(() => console.log('C')); // microtask queue
setTimeout(() => console.log('D'), 0);  // macrotask queue
// Output: A -> B -> C -> D
```

**S3: Trong production, bạn thấy Event Loop lag đột biến lên 2000ms mỗi buổi tối lúc 2h sáng. Debug như thế nào?**
A: 2h sáng thường là thời điểm job chạy (cron job, analytics aggregation). Kiểm tra: (1) Có scheduled job nào chạy lúc đó không? (2) Có DB query nặng/full table scan nào được trigger? (3) Dùng `clinic.js flame` để chụp CPU flamegraph trong thời gian đó. (4) Check if any `*Sync` operation hoặc lớp JSON.parse trên data lớn. (5) Nếu là computation, chuyển sang Worker Thread.

**S4: làm sao xử lý 1 triệu record từ DB mà không block Event Loop và không out-of-memory?**
A: Dùng cursor/streaming query thay vì load hết:
```javascript
// Mongoose streaming
const cursor = User.find({ active: true }).cursor();
let processed = 0;
for await (const user of cursor) {
    await processUser(user);
    processed++;
    if (processed % 1000 === 0) {
        await new Promise(resolve => setImmediate(resolve)); // yield Event Loop
    }
}
// Mỗi 1000 record, Event Loop có cơ hội xử lý request khác
```

**S5: Tại sao trong I/O callback, `setImmediate` luôn chạy trước `setTimeout(fn, 0)`?**
A: Khi callback I/O đang chạy, Event Loop đang ở Poll phase. Sau khi Poll phase hoàn thành, nó chuyển sang **Check phase** (setImmediate) trước khi quay lại Timers phase (setTimeout). Do đó trong I/O callback, setImmediate luôn được ưu tiên. Bên ngoài I/O callback (ở top-level), thứ tự giữa setImmediate và setTimeout(0) là **không xác định** — phụ thuộc vào tốc độ khởi động process.

**S6: Nếu Microtask Queue mãi không trống (mỗi microtask thêm microtask mới), điều gì xảy ra?**
A: Event Loop bị "starved" — nó mãi mãi ở bước drain microtask queue, không bao giờ chạy đến macrotask (setTimeout, I/O callbacks). Toàn bộ server bị đóng băng giống như block Call Stack. Đây là anti-pattern nguy hiểm — đảm bảo microtask callback có điều kiện dừng rõ ràng.

## 14. References

- Node.js official docs — The Node.js Event Loop: https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick
- libuv design overview: https://docs.libuv.org/en/v1.x/design.html
- Node.js API — `perf_hooks.monitorEventLoopDelay`: https://nodejs.org/api/perf_hooks.html#perf_hooksmonitoreventloopdelayoptions
- Node.js Worker Threads: https://nodejs.org/api/worker_threads.html
- Bert Belder — "Everything you need to know about Node.js Event Loop" (JSConf Asia 2016): https://www.youtube.com/watch?v=PNa9OMajl9U

## 15. Real-world Code

- libuv source code (Event Loop core): https://github.com/libuv/libuv/blob/v1.x/src/unix/core.c
- Node.js internal Event Loop implementation: https://github.com/nodejs/node/blob/main/deps/uv/src/unix/core.c
- clinic.js — Event Loop profiling tool: https://github.com/clinicjs/node-clinic
- 0x — Flamegraph profiler: https://github.com/davidmarkclements/0x
- fastify — production HTTP framework aware of Event Loop: https://github.com/fastify/fastify

## 16. Community

- Stack Overflow — "What is the Event Loop?": https://stackoverflow.com/questions/21607692/understanding-the-event-loop
- Stack Overflow — "setImmediate vs process.nextTick": https://stackoverflow.com/questions/15349733/setimmediate-vs-nexttick
- Reddit r/node — Event Loop discussions: https://www.reddit.com/r/node/search/?q=event+loop
- Node.js blog — Don't Block the Event Loop: https://nodejs.org/en/learn/best-practices/dont-block-the-event-loop
- Loupe — visual tool to see Event Loop in action: http://latentflip.com/loupe/
- Deepal Jayasekara — "Event Loop and the Big Picture" series: https://blog.insiderattack.net/event-loop-and-the-big-picture-nodejs-event-loop-part-1-1cb67a182810
