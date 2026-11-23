## Đề bài: Bài tập 2 — Lập trình web

1. Sử dụng Node-RED với node `http in` và `http response` để tạo API đơn giản, thực hiện xử lý/thuật toán tự thiết kế và trả về dữ liệu dạng JSON (ví dụ tương tự cấu trúc `{"ok":1,"msg":"thành công","dssv":[{"name":"Cốp","money":123},{"name":"David","money":456}]}`).
2. Cấu hình Nginx để trang web có thể gọi tới API trên Node-RED.
3. Viết code JavaScript trong trang HTML để gọi API trên và hiển thị kết quả lên web[cite: 11].
   # Bài làm

   ## 1. Tạo API đơn giản trên Node-RED (`http in` + `http response`)

### Lý thuyết
- **Kiến trúc Request – Response trong HTTP API:** Khi trình duyệt hoặc ứng dụng gửi một yêu cầu HTTP (ví dụ `GET /api/dssv`) tới máy chủ, hệ thống sẽ tiếp nhận yêu cầu, chạy logic xử lý dữ liệu và phản hồi kèm mã trạng thái (`200 OK`) cùng nội dung JSON để bên gọi dễ dàng phân tích và hiển thị.
- **Node `http in`:** Là node đầu vào của luồng (flow) trong Node-RED, lắng nghe các yêu cầu HTTP theo phương thức (`GET`, `POST`, ...) và đường dẫn (`URL`) được cấu hình. Khi có request khớp, node này tạo ra đối tượng `msg` chứa thông tin yêu cầu (`msg.req`, `msg.payload`, `msg.query`) và đẩy sang node tiếp theo.
- **Node `function`:** Cho phép viết code JavaScript tùy chỉnh để xử lý nghiệp vụ hoặc thuật toán. Dữ liệu trả về của API thường được gán vào `msg.payload`, còn tiêu đề phản hồi (như kiểu dữ liệu `application/json` và header CORS) được cấu hình qua `msg.headers`.
- **Node `http response`:** Là node đầu ra có nhiệm vụ gửi `msg.payload` và `msg.headers` về cho client kèm mã trạng thái HTTP (mặc định `200`). Nếu thiếu node này hoặc luồng không đi tới node này, request sẽ bị treo hoặc trả lỗi timeout.
- **Thuật toán tự thiết kế trong bài:**
  1. Khởi tạo mảng dữ liệu đầu vào gồm các bản ghi có `name` và `money`.
  2. Duyệt qua từng phần tử bằng `.map()`, cộng thêm mức thưởng cố định (`+10`) cho mỗi bản ghi để tạo giá trị cập nhật.
  3. Cộng dồn giá trị sau cập nhật vào biến `tongTien` để tính tổng toàn danh sách.
  4. Đóng gói kết quả thành object JSON có cấu trúc `ok`, `msg`, `tong` và `dssv`, đồng thời gắn `Content-Type: application/json; charset=utf-8` để hiển thị tiếng Việt chuẩn xác.

### Code & Thực hiện
1. Truy cập giao diện Node-RED (`http://localhost:1880`), kéo 3 node vào vùng làm việc:
   - **`http in`**: chọn Method `GET`, nhập URL `/api/dssv`.
   - **`function`**: đặt tên (ví dụ `Xử lý danh sách & tính tổng`) và dán code bên dưới.
   - **`http response`**: để Status code `200`.
2. Nối dây theo thứ tự: `http in` $\rightarrow$ `function` $\rightarrow$ `http response`, sau đó bấm **Deploy** ở góc trên bên phải.
3. Kiểm tra nhanh API bằng cách mở trực tiếp `http://localhost:1880/api/dssv` trên trình duyệt hoặc chạy lệnh `curl` trong Terminal:

```bash
curl -i http://localhost:1880/api/dssv
```

Code xử lý trong node `function`:

```javascript
// Thuật toán: cộng thêm 10 cho từng bản ghi, tính tổng và trả về JSON
let danhSach = [
    { name: "Cốp", money: 123 },
    { name: "David", money: 456 },
    { name: "Hiền", money: 500 }
];

let tongTien = 0;

danhSach = danhSach.map(item => {
    const capNhat = Number(item.money || 0) + 10;
    tongTien += capNhat;
    return {
        name: item.name,
        money: capNhat
    };
});

msg.headers = {
    "Content-Type": "application/json; charset=utf-8",
    "Access-Control-Allow-Origin": "*"
};

msg.payload = {
    ok: 1,
    msg: "thành công",
    tong: tongTien,
    dssv: danhSach
};

return msg;
```

<img width="578" height="307" alt="image" src="https://github.com/user-attachments/assets/2b2b69dc-8f53-45b9-b0bd-75585ba4ba7d" />

---

## 2. Cấu hình Nginx để web gọi API trên Node-RED

### Lý thuyết
- **Reverse Proxy (`proxy_pass`):** Thay vì để trình duyệt gọi thẳng tới cổng nội bộ của Node-RED (`:1880`), Nginx đóng vai trò cổng trung gian tiếp nhận request tại đường dẫn `/api/...` trên domain của website và chuyển tiếp (proxy) sang dịch vụ Node-RED. Cách này giúp quản lý tập trung, dễ gắn SSL/domain và tránh xung đột cổng.
- **Các header chuyển tiếp (`proxy_set_header`):** Giúp dịch vụ phía sau (Node-RED) biết được thông tin gốc của request như `Host`, địa chỉ IP thật của client (`X-Real-IP`, `X-Forwarded-For`) và giao thức (`X-Forwarded-Proto`).
- **CORS (`Access-Control-Allow-Origin`):** Cơ chế bảo mật của trình duyệt khi trang web thực hiện yêu cầu tới nguồn khác (khác domain, giao thức hoặc cổng). Khi gọi qua cùng origin qua reverse proxy `/api/` hoặc có bật header `Access-Control-Allow-Origin *`, trình duyệt sẽ cho phép đọc dữ liệu JSON trả về.

### Code & Thực hiện
1. Mở file cấu hình site của Nginx (ví dụ `/etc/nginx/sites-available/truongnamtu.id.vn` hoặc file config mounted vào container `web_nginx`) và thêm khối `location /api/`:

```nginx
server {
    listen 80;
    server_name truongnamtu.id.vn;

    root /var/www/site1;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # Reverse proxy chuyển tiếp các request /api/ sang Node-RED
    location /api/ {
        proxy_pass [http://127.0.0.1:1880/api/](http://127.0.0.1:1880/api/);
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
        add_header Access-Control-Allow-Headers "Content-Type, Authorization";
    }
}
```

2. Kiểm tra lỗi cú pháp cấu hình và reload lại Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

3. Kiểm thử đường dẫn API đi qua Nginx:

```bash
curl -i [http://truongnamtu.id.vn/api/dssv](http://truongnamtu.id.vn/api/dssv)
```

<img width="575" height="331" alt="image" src="https://github.com/user-attachments/assets/69ad9d2f-782b-4163-bbbb-934bd7279db3" />

---

## 3. Code HTML + JavaScript gọi API hiển thị lên web

### Lý thuyết
- **Lập trình bất đồng bộ với `async / await` và `fetch()`:** Hàm `fetch(API_URL)` gửi request HTTP không đồng bộ mà không làm đơ giao diện trang web. Từ khóa `await` chờ phản hồi HTTP hoàn tất, sau đó gọi `await res.json()` để chuyển chuỗi JSON thành object JavaScript.
- **Kiểm tra trạng thái phản hồi (`res.ok`):** Giúp phát hiện sớm các lỗi HTTP như `404 Not Found` hoặc `502 Bad Gateway` (ví dụ khi Node-RED chưa bật hoặc `proxy_pass` sai địa chỉ) để đưa vào khối `catch` xử lý thông báo thân thiện cho người dùng.
- **Cập nhật DOM động:** Sau khi kiểm tra `data.ok === 1` và `Array.isArray(data.dssv)`, JavaScript dùng `.map()` kết hợp template string để tạo các hàng `<tr>` trong bảng `<tbody>` và in dữ liệu JSON gốc ra thẻ `<pre>` để dễ đối chiếu.

### Code & Thực hiện
Tạo hoặc cập nhật file `index.html` trong thư mục gốc của website với nội dung sau:

```html
<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Bài tập 2 - Gọi API Node-RED</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      max-width: 760px;
      margin: 40px auto;
      padding: 0 16px;
      line-height: 1.5;
    }
    button {
      padding: 8px 16px;
      font-size: 15px;
      cursor: pointer;
    }
    #status {
      margin-top: 12px;
      font-weight: bold;
    }
    pre {
      background: #f4f4f4;
      padding: 12px;
      border-radius: 6px;
      overflow-x: auto;
    }
    table {
      width: 100%;
      border-collapse: collapse;
      margin-top: 16px;
    }
    th, td {
      border: 1px solid #ccc;
      padding: 8px 12px;
      text-align: left;
    }
    th {
      background: #f0f0f0;
    }
  </style>
</head>
<body>
  <h1>Danh sách từ API Node-RED</h1>
  <p>Dữ liệu được lấy qua Nginx reverse proxy và hiển thị bằng JavaScript.</p>

  <button id="btnLoad">Gọi lại API</button>
  <p id="status"></p>

  <table>
    <thead>
      <tr>
        <th>STT</th>
        <th>Tên</th>
        <th>Money (đã cộng thưởng)</th>
      </tr>
    </thead>
    <tbody id="ketQua"></tbody>
  </table>

  <h3>Dữ liệu JSON gốc:</h3>
  <pre id="rawJson">Chưa có dữ liệu</pre>

  <script>
    const API_URL = "/api/dssv";

    async function taiDuLieu() {
      const statusEl = document.getElementById("status");
      const tbodyEl = document.getElementById("ketQua");
      const rawEl = document.getElementById("rawJson");

      statusEl.textContent = "Đang tải dữ liệu từ API...";
      try {
        const res = await fetch(API_URL);
        if (!res.ok) {
          throw new Error("Mã trạng thái HTTP: " + res.status);
        }

        const data = await res.json();
        rawEl.textContent = JSON.stringify(data, null, 2);

        if (data.ok === 1 && Array.isArray(data.dssv)) {
          const tongText = data.tong !== undefined ? ` | Tổng tiền: ${data.tong}` : "";
          statusEl.textContent = `Trạng thái: ${data.msg}${tongText}`;

          tbodyEl.innerHTML = data.dssv
            .map((sv, index) => `
              <tr>
                <td>${index + 1}</td>
                <td>${sv.name}</td>
                <td>${sv.money}</td>
              </tr>
            `)
            .join("");
        } else {
          statusEl.textContent = "Dữ liệu JSON trả về không đúng cấu trúc mong đợi.";
          tbodyEl.innerHTML = "";
        }
      } catch (err) {
        statusEl.textContent = "Lỗi gọi API: " + err.message;
        tbodyEl.innerHTML = "";
      }
    }

    document.getElementById("btnLoad").addEventListener("click", taiDuLieu);
    window.addEventListener("DOMContentLoaded", taiDuLieu);
  </script>
</body>
</html>
```

<img width="583" height="305" alt="image" src="https://github.com/user-attachments/assets/195cc7ec-7a23-4c24-94c6-1dfee92e08f8" />


---

## 4. Tổng kết & Checklist nộp GitHub
- Đã tạo flow API trên Node-RED bằng `http in` + `function` + `http response` trả về JSON có thuật toán xử lý riêng.
- Đã cấu hình reverse proxy trên Nginx để chuyển tiếp `/api/` tới Node-RED và kiểm tra cấu hình hợp lệ.
- Đã viết mã HTML + JavaScript dùng `fetch()` gọi API và render bảng dữ liệu cùng JSON gốc lên giao diện web.
- Đã tải đủ ảnh minh chứng (`nodered-api.png`, `nginx-api.png`, `web-api.png`) và file báo cáo lên repository GitHub ở chế độ public.
