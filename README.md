# LAB 8 — Local DNS Attack (pfSense)

## 1. Thông tin sinh viên

- **Họ và tên:** Phạm Hữu Ân
- **MSSV:** 1150080084
- **Mã lớp:** CNPM1

---

## 2. Mục tiêu

- Dựng và kiểm tra môi trường DNS lab cô lập có pfSense, DNS server, máy nạn nhân và máy kiểm thử.
- Hiểu quá trình phân giải tên miền và vai trò của file `hosts`, DNS server và DNS cache.
- Quan sát ảnh hưởng của việc sửa file `hosts` trong phạm vi máy nạn nhân.
- Thực hành và phân tích DNS spoofing trong mạng lab được cho phép.
- Thực hành và phân tích DNS cache poisoning theo tài liệu bài lab.
- Sử dụng `dig`/`nslookup`, `curl`/trình duyệt và Wireshark để đối chiếu kết quả.
- Xóa cache, khôi phục cấu hình và xác minh kết quả phân giải trở về trạng thái ban đầu.

> **Phạm vi an toàn:** Chỉ thực hiện trong mạng ảo hóa cô lập được giảng viên cho phép. Không gửi các gói giả mạo đến DNS server hoặc thiết bị ngoài phạm vi lab.

---

## 3. Môi trường thực hành

### 3.1. Topology và địa chỉ IP

| Thiết bị | Vai trò | Địa chỉ theo tài liệu |
|---|---|---|
| pfSense — DNSLAB | Gateway/firewall cho mạng lab | `10.9.0.1/24` |
| DNS-SERVER | Ubuntu Server chạy BIND9/DNS resolver theo cấu hình lab | `10.9.0.10` |
| VICTIM | Máy kiểm thử phân giải tên và truy cập web | `10.9.0.100` |
| ATTACKER | Kali Linux dùng để quan sát/thực hiện phép thử trong lab | `10.9.0.200` |
| DMZ-WEB | Máy chủ web đích ban đầu | `172.16.6.10` |
| APP-TEST/AUTH-DNS | Máy ứng dụng hoặc DNS có thẩm quyền theo sơ đồ bài | `172.16.6.20` |

- Mạng lab chính: `10.9.0.0/24`.
- Tên miền kiểm thử theo tài liệu: `www.external.lab9.test`.
- Kết quả baseline mong đợi: `www.external.lab9.test` phân giải thành `172.16.6.10`.

**Lưu ý:** Tên miền có chuỗi `lab9` theo tài liệu được cung cấp. Nếu giảng viên yêu cầu tên miền khác cho LAB 8, hãy cập nhật nhất quán trong cấu hình DNS, lệnh kiểm tra, ảnh minh chứng và báo cáo.

### 3.2. Sơ đồ logic

```text
                   Mạng lab cô lập: 10.9.0.0/24
                                |
                         pfSense DNSLAB
                           10.9.0.1
                                |
          +---------------------+---------------------+
          |                     |                     |
     DNS-SERVER               VICTIM                ATTACKER
      10.9.0.10              10.9.0.100             10.9.0.200

       Các máy/segment đích theo sơ đồ bài:
       DMZ-WEB 172.16.6.10
       APP-TEST/AUTH-DNS 172.16.6.20
```

Tên interface, cách nối card mạng và cách định tuyến giữa các segment cần theo đúng sơ đồ/phiếu lab của giảng viên. Không tự thêm kết nối giữa các mạng nếu chưa được yêu cầu.

---

## 4. Chuẩn bị và kiểm tra kết nối

Trước khi thực hiện các task:

1. Kiểm tra từng VM đã gắn đúng mạng lab cô lập.
2. Kiểm tra địa chỉ IP trên DNS-SERVER, VICTIM và ATTACKER.
3. Kiểm tra interface DNSLAB và các firewall rule trên pfSense theo tài liệu.
4. Kiểm tra kết nối cần thiết giữa các máy trong phạm vi lab.
5. Chụp ảnh bằng chứng địa chỉ IP và cấu hình mạng thực tế.

Các lệnh kiểm tra IP tham khảo:

**Ubuntu / Kali Linux**
```bash
ip -br addr
ip route
```

**Windows**
```powershell
ipconfig /all
```

Không ghi trạng thái `PASS` nếu chưa chạy lệnh hoặc chưa xác minh kết quả thực tế.

---

## 5. Kiểm tra BIND9 và DNS baseline

Trên DNS-SERVER, kiểm tra cú pháp cấu hình BIND9 và zone theo tên/tệp thực tế trong cấu hình lab:

```bash
sudo named-checkconf
sudo named-checkzone <ten-zone> <tep-zone>
```

Thay `<ten-zone>` và `<tep-zone>` bằng giá trị thực tế. Lưu ảnh terminal cho thấy lệnh đã chạy và kết quả trả về.

Từ máy được chỉ định trong bài, truy vấn tên miền baseline:

```bash
dig www.external.lab9.test
```

Nếu cần chỉ định DNS server, dùng địa chỉ DNS server đúng theo cấu hình của lab, ví dụ:

```bash
dig @10.9.0.10 www.external.lab9.test
```

Kết quả baseline theo tài liệu:

```text
www.external.lab9.test -> 172.16.6.10
```

Lưu ý ghi lại toàn bộ kết quả thực tế. Nếu địa chỉ trả về khác, kiểm tra cấu hình zone, resolver đang được sử dụng và cache trước khi tiếp tục.

---

## 6. Task 1 — Kiểm tra ảnh hưởng của file hosts

Thực hiện đúng các bước trong tài liệu bài lab để thay đổi file `hosts` trên VICTIM.

Sau khi thay đổi:

- Truy cập tên miền kiểm thử bằng trình duyệt hoặc `curl`.
- Dùng `dig` để truy vấn DNS và so sánh kết quả.
- Ghi nhận liệu kết nối bị ảnh hưởng bởi mục `hosts` hay bởi câu trả lời từ DNS.
- Khôi phục file `hosts` về trạng thái ban đầu sau khi hoàn tất task.

**Bằng chứng cần có:**

- Nội dung mục `hosts` liên quan đến tên miền kiểm thử.
- Kết quả truy cập bằng trình duyệt hoặc `curl`.
- Kết quả `dig` để so sánh.
- Ảnh chụp phải thể hiện tên miền và kết quả rõ ràng.

Không khẳng định đã chuyển hướng thành công nếu kết quả thực tế không thể hiện điều đó.

---

## 7. Task 2 — DNS spoofing

Thực hiện phép thử DNS spoofing theo tài liệu bài lab và chỉ trong mạng DNS-LAB cô lập.

Quy trình ghi nhận kết quả:

1. Mở Wireshark trên interface phù hợp với topology lab.
2. Bắt các gói DNS liên quan đến tên miền kiểm thử.
3. Thực hiện phép thử theo đúng hướng dẫn của giảng viên.
4. Kiểm tra câu truy vấn DNS, phản hồi nhận được và địa chỉ IP mà VICTIM sử dụng.
5. So sánh kết quả với baseline `172.16.6.10`.

Bộ lọc Wireshark có thể dùng để thu hẹp gói DNS:

```text
dns
```

**Bằng chứng cần có:**

- Ảnh Wireshark có truy vấn và phản hồi liên quan.
- Ảnh kết quả phân giải hoặc truy cập từ VICTIM.
- Ghi chú địa chỉ IP thực tế quan sát được và kết luận dựa trên bằng chứng.

Tên interface bắt gói và nội dung gói có thể khác tùy môi trường; hãy ghi lại kết quả thật từ máy ảo.

---

## 8. Task 3 — DNS cache poisoning

Thực hiện phép thử cache poisoning theo đúng tài liệu bài lab trong môi trường được phép.

Ghi lại các trạng thái sau:

1. Kết quả truy vấn trước phép thử.
2. Các gói truy vấn/phản hồi liên quan trong Wireshark.
3. Trạng thái cache của DNS resolver sau phép thử, bằng phương pháp phù hợp với resolver đang dùng.
4. Kết quả truy vấn trong khi cache còn hiệu lực.
5. Kết quả sau khi flush/xóa cache theo hướng dẫn bài lab.

Kết quả khôi phục mong đợi theo tài liệu:

```text
www.external.lab9.test -> 172.16.6.10
```

Chỉ ghi nhận cache đã bị thay đổi hoặc đã được khôi phục khi có bằng chứng từ truy vấn/cache thực tế. Không tự điền địa chỉ IP giả định vào phần kết quả.

---

## 9. Khôi phục môi trường sau thực hành

Sau khi hoàn tất:

- Khôi phục file `hosts` của VICTIM.
- Xóa cache DNS theo hướng dẫn bài lab.
- Khôi phục các cấu hình tạm thời đã thay đổi.
- Kiểm tra lại kết quả baseline của tên miền.
- Khôi phục cấu hình mạng ảo theo yêu cầu, bao gồm Promiscuous Mode về `Deny` nếu bài lab yêu cầu.
- Tắt các công cụ hoặc tiến trình kiểm thử không còn cần thiết.

Chụp ảnh bằng chứng cho trạng thái sau khi khôi phục.

---

## 10. Danh sách ảnh minh chứng

Tạo thư mục `evidence/` và lưu ảnh chụp trực tiếp từ môi trường lab. Gợi ý tên file:

```text
evidence/
├── 01-pfsense-dnslab-interface.png
├── 02-pfsense-firewall-rules.png
├── 03-dns-server-ip.png
├── 04-victim-ip.png
├── 05-attacker-ip.png
├── 06-bind9-checkconf-checkzone.png
├── 07-dns-baseline-dig.png
├── 08-task1-hosts-file.png
├── 09-task1-browser-curl-and-dig.png
├── 10-task2-wireshark-dns-packets.png
├── 11-task2-victim-result.png
├── 12-task3-wireshark-response.png
├── 13-task3-cache-state.png
├── 14-task3-after-flush-baseline.png
└── 15-restored-settings-promiscuous-deny.png
```

Danh sách này là gợi ý tổ chức minh chứng; điều chỉnh theo các mục bắt buộc trong phiếu lab của giảng viên.

### Quy tắc bằng chứng

- Ảnh phải được chụp từ VM/môi trường lab thực tế.
- Ảnh phải đọc được lệnh, địa chỉ IP, tên miền hoặc kết quả cần chứng minh.
- Không dùng ảnh minh họa do AI tạo thay cho bằng chứng thực hành.
- Nếu bước nào chưa hoàn thành, ghi `FAIL` hoặc `NOT COMPLETED` và nêu lỗi; không tạo ảnh giả.
- Không đưa mật khẩu, token, cookie hoặc dữ liệu cá nhân vào repository.

---

## 11. Bảng kết quả PASS / FAIL

Điền cột trạng thái dựa trên kết quả thực tế trước khi nộp.

| Hạng mục | Trạng thái | Ghi chú / đường dẫn evidence |
|---|---|---|
| Các VM kết nối đúng mạng lab | Chưa xác nhận | |
| IP pfSense DNSLAB đúng | Chưa xác nhận | |
| IP DNS-SERVER, VICTIM, ATTACKER đúng | Chưa xác nhận | |
| `named-checkconf` thành công | Chưa xác nhận | |
| `named-checkzone` thành công | Chưa xác nhận | |
| Baseline phân giải về `172.16.6.10` | Chưa xác nhận | |
| Task 1 — ảnh hưởng file `hosts` | Chưa xác nhận | |
| Task 2 — ghi nhận DNS spoofing bằng Wireshark | Chưa xác nhận | |
| Task 3 — ghi nhận cache poisoning | Chưa xác nhận | |
| Flush cache và baseline được khôi phục | Chưa xác nhận | |
| Promiscuous Mode được khôi phục về `Deny` (nếu áp dụng) | Chưa xác nhận | |

> Để trạng thái `Chưa xác nhận` cho đến khi đã kiểm tra. Chỉ đổi thành `PASS` hoặc `FAIL` dựa trên kết quả quan sát được.

---

## 12. Câu hỏi và phân tích

Trả lời các câu hỏi theo đúng phiếu bài lab. Khi giải thích, phân biệt rõ:

- **File hosts:** cơ chế phân giải cục bộ trên một máy; không đồng nghĩa DNS server đã bị thay đổi.
- **DNS spoofing:** phản hồi DNS giả mạo được quan sát trong điều kiện kiểm thử; cần dùng packet capture và kết quả trên máy nạn nhân để chứng minh.
- **DNS cache poisoning:** dữ liệu không đúng được lưu vào cache resolver; cần kiểm tra cache và truy vấn sau khi flush để đánh giá kết quả.
- **Wireshark:** dùng làm bằng chứng gói tin, nhưng riêng một gói tin chưa đủ để chứng minh mọi máy đều bị ảnh hưởng.
- **Khôi phục:** kiểm tra lại cache, file `hosts` và kết quả baseline thay vì chỉ dựa vào việc đã chạy lệnh.

Nếu câu hỏi yêu cầu kết quả cụ thể từ lần thực hành, hãy dùng log/ảnh của chính mình thay vì điền kết quả dự kiến.

---

## 13. Lỗi thường gặp và cách xử lý

### DNS trả về địa chỉ không như mong đợi

- Kiểm tra DNS server mà client đang truy vấn.
- Kiểm tra zone và bản ghi của BIND9.
- Kiểm tra file `hosts`.
- Kiểm tra cache resolver và thực hiện flush theo hướng dẫn.
- Chạy lại truy vấn baseline và lưu kết quả.

### Không thấy gói DNS trong Wireshark

- Xác nhận đã chọn đúng interface.
- Xác nhận VICTIM thực sự phát sinh truy vấn DNS.
- Dùng display filter `dns`.
- Kiểm tra topology và đường đi của traffic theo sơ đồ lab.

### BIND9 không nạp cấu hình

- Chạy `named-checkconf`.
- Chạy `named-checkzone` với đúng tên zone và tệp zone.
- Đọc thông báo lỗi, sửa cấu hình rồi kiểm tra lại trước khi tiếp tục.

Ghi lại lỗi thực tế và cách xử lý đã áp dụng; không ghi một lỗi giả định là lỗi đã xảy ra.

---

## 14. Kết luận

Trong LAB 8, mục tiêu là quan sát và phân tích cách file `hosts`, phản hồi DNS và cache DNS ảnh hưởng đến việc phân giải tên miền trong một mạng lab cô lập. Kết luận cuối cùng cần dựa trên các bằng chứng đã thu thập: cấu hình mạng, kiểm tra BIND9, truy vấn baseline, packet capture, kết quả trên VICTIM và kết quả sau khi xóa cache/khôi phục cấu hình.

**Trước khi nộp:** kiểm tra lại tên bài, tên miền theo phiếu lab, bảng PASS/FAIL và thư mục `evidence/`. Không đánh dấu một hạng mục là hoàn thành khi chưa có bằng chứng tương ứng.
