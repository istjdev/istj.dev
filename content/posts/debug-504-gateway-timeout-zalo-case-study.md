---
title: "Chuyện tâm linh ngành Dev: Khi Server Zalo báo 'Phi như ngựa' nhưng Client lại ăn... 504 Timeout 🤡"
date: 2026-02-02T11:00:00+07:00
draft: false
tags: ["devops", "networking", "debug", "504-error", "zalo-api", "mtu-mismatch"]
categories: ["Tech Blog", "System Integration"]
description: "Phân tích case study thực tế: Tại sao đã ép MSS để fix connection timeout rồi mà vẫn dính chưởng 504 Gateway Timeout? Bí ẩn được giải mã bằng sơ đồ Sequence."
slug: "debug-504-gateway-timeout-zalo-case-study"
---

Chào các đồng code,

Hôm nay tôi xin phép kể lại một case debug "ảo ma Canada" mà tôi vừa trải qua với đội Zalo. Câu chuyện mang tên: **"Ông nói gà, bà nói vịt"** hay còn gọi là **"Bí ẩn chiếc gói tin mất tích"**.

Anh em làm tích hợp (Integration) chắc không lạ gì cảnh: Bên mình gọi API lỗi tùm lum, nhưng bên đối tác thì chìa ra cái biểu đồ xanh mướt và phán một câu xanh rờn: *"Bên anh vẫn mượt nhé chú em!"*.

Đây, mời anh em thị tẩm cái ảnh bằng chứng đầu tiên:

![Zalo Response Time](/images/zalo-response-time.jpg)
*(Ảnh minh họa: Zalo show hàng response time backend chỉ ~15ms-30ms. Nhanh như một cơn gió!)*

Nhưng thực tế bên tôi (Client Side) thì sao? **504 Gateway Timeout**. Chờ mòn mỏi, chờ đến hóa đá mà không thấy kết quả đâu.

Vậy thì... đứa nào nói dối? Xin thưa, **không ai nói dối cả**. Chỉ là chúng ta đang nhìn vào những "vũ trụ" khác nhau mà thôi.

## Tập 1: Cú lừa của "MSS 1350"

Trước đó, hệ thống bên tôi bị lỗi kết nối (Connection Timeout) - kiểu như gõ cửa mà nhà Zalo không thèm mở. Sau khi vò đầu bứt tai, tôi phát hiện ra vấn đề nằm ở **MTU/MSS** (Kích thước gói tin). Đường mạng nó hẹp (MTU thấp), mà gói tin thì to như cái container, nên bị kẹt cứng.

**Giải pháp thần thánh:** Tôi ép MSS xuống **1350**.

> *Tưởng tượng:* Thay vì đi xe container, tôi bắt server chia nhỏ dữ liệu ra nhét vào xe máy thôi.

**Kết quả:**
* Gõ cửa Zalo mở ngay! (Handshake TCP thành công).
* Gửi request đi OK.
* Zalo nhận được, xử lý vèo cái xong.

Tưởng đời nở hoa, ai ngờ **bế tắc tập 2**: Zalo xử lý xong, nhưng tôi **không nhận được hàng về**. Thay vì lỗi kết nối, giờ nó chuyển sang **504 Gateway Timeout**.

## Tập 2: Tại sao Backend nhanh mà tôi vẫn 504?

Ở đây có một sự "lệch pha" kinh điển về định nghĩa chữ **DONE (Xong)**. Để anh em đỡ phải tưởng tượng, tôi đã vẽ lại hiện trường vụ án bằng sơ đồ dưới đây. Nhìn phát hiểu luôn tại sao hai bên cãi nhau:

![Debug Timeout Diagram](/images/client-zalo-timeout.jpg)
*(Sơ đồ giải phẫu "cục máu đông" 504 Gateway Timeout giữa Client Server và Zalo)*

**Nhìn vào cái vùng màu đỏ (Giai đoạn 3) trong sơ đồ, anh em sẽ thấy chân tướng sự việc:**

1.  **Giai đoạn 1 & 2 (Màu vàng):** Request đi rất trơn tru vì gói tin nhỏ (< 1400 bytes). Zalo Backend xử lý cực nhanh (20ms) và trả về "Success 200". Đây là lý do Zalo khẳng định họ không lỗi.
2.  **Sự cố (Vùng đỏ):** Khi trả hàng về, Zalo đóng gói một cục to đùng (Response Body 1500 bytes) và gắn cái cờ **DF=1 (Don't Fragment - Cấm cắt nhỏ)**.
3.  **Cái kết:** Cái "thùng hàng" 1500 bytes này lao vào đường mạng (Internet/VPN) vốn chỉ chịu được tải 1400 bytes.
    * Router mạng bảo: *"To quá không qua được, mà lại cấm tao xẻ nhỏ ra, thôi tao vứt!"* -> **PACKET DROPPED**.
    * Load Balancer Zalo thấy bên mình chưa nhận được thì gửi lại (Retransmission), nhưng vẫn gửi cái thùng to đấy -> **Lại DROP tiếp**.

👉 **Hậu quả:**
* **Zalo:** *"Tao gửi rồi nhé, gửi tận mấy lần, lỗi tại mạng mày!"* (Log Success).
* **Client (Bên mình):** *"Tao đứng chờ 60 giây không thấy gì cả!"* -> Cắt kết nối, báo **504 Gateway Timeout**.

## Bài học xương máu (Giải pháp)

Đừng tin 100% vào cái dashboard Backend của đối tác. Nó chỉ chứng minh là code của họ chạy ổn, chứ không chứng minh là **hạ tầng mạng** (Network Layer) đang ổn.

Để tránh bị "bóng chuyền trách nhiệm", anh em cần làm gì?

### 1. Show cái sơ đồ này cho đối tác

Đôi khi một hình ảnh bằng ngàn lời nói. Chỉ cho họ thấy điểm Drop packet nằm ở lớp Network/LB chứ không phải App.

**Lý tưởng nhất** là đội Zalo sẽ fix ở phía họ bằng cách:
* Tắt DF flag hoặc config Path MTU Discovery (PMTUD) đúng cách
* Giảm response size hoặc enable compression
* Config MSS Clamping ngay tại Load Balancer của họ

Nhưng thực tế đời không như là mơ...

### 2. Dùng "Kính chiếu yêu" TCPDump - Bằng chứng thép

Bắt gói tin ngay tại server mình. Nếu thấy request đi mà không thấy response về (hoặc thấy response về nhưng toàn bị Retransmission đỏ lòm) => Bằng chứng không thể chối cãi.

```bash
# Bắt packet với Zalo (thay <ZALO_IP> bằng IP thực tế)
tcpdump -i eth0 -w zalo-debug.pcap host <ZALO_IP> and port 443

# Sau khi có file .pcap, mở bằng Wireshark và filter:
tcp.analysis.retransmission || tcp.analysis.lost_segment
```

Cái này quan trọng để **chứng minh với sếp** rằng không phải do code mình viết tệ, mà là vấn đề network thực sự. Screenshot gửi kèm email cho Zalo là xong!

### 3. Khi Zalo "đá bóng" - Tự lực cánh sinh bằng iptables

Đây là trường hợp **90% anh em sẽ gặp**: Zalo không fix, hoặc fix mãi không xong (vài tuần/tháng), mà sếp đang hối deadline. Phải tự cứu lấy mình thôi!

#### Solution A: Ép MSS ngay từ lúc bắt tay TCP

Khi TCP handshake (SYN-SYNACK-ACK), hai bên sẽ "thương lượng" MSS. Nếu mình advertise MSS thấp, Zalo **bắt buộc** phải tuân theo!

```bash
# Ép MSS cho traffic ĐẾN Zalo (POSTROUTING)
iptables -t mangle -A POSTROUTING -p tcp --tcp-flags SYN,RST SYN \
  -d <ZALO_IP_RANGE> -j TCPMSS --set-mss 1300

# Ép MSS cho traffic TỪ Zalo về (PREROUTING)
iptables -t mangle -A PREROUTING -p tcp --tcp-flags SYN,RST SYN \
  -s <ZALO_IP_RANGE> -j TCPMSS --clamp-mss-to-pmtu
```

**Giải thích:**
* `--set-mss 1300`: Cứng rắn, bắt MSS = 1300 bytes (an toàn cho hầu hết VPN)
* `--clamp-mss-to-pmtu`: Tự động điều chỉnh MSS theo MTU của path (linh hoạt hơn)

**Tips:** Dùng `1300` thay vì `1350` nếu đi qua VPN/IPsec vì còn phải trừ overhead của VPN header nữa!

#### Solution B: Giảm MTU của interface (hardcore hơn)

```bash
# Temporary (mất khi reboot)
ip link set dev eth0 mtu 1400

# Permanent - thêm vào /etc/network/interfaces:
# iface eth0 inet static
#   ...
#   mtu 1400

# Hoặc với netplan (Ubuntu 18.04+) trong /etc/netplan/*.yaml:
# ethernets:
#   eth0:
#     mtu: 1400
```

**⚠️ Lưu ý:** Cách này ảnh hưởng **TẤT CẢ** traffic qua interface đó. Chỉ nên dùng nếu server chỉ giao tiếp với Zalo, hoặc toàn bộ mạng của bạn đều có MTU thấp.

## Túm cái váy lại

Làm Dev tích hợp giống như yêu xa vậy.
Mình nhắn tin (Request), bên kia đã xem và soạn tin (Processing), nhưng mạng lag nên tin nhắn trả lời (Response) mãi không tới. Mình dỗi (Timeout), còn bên kia thì thề thốt *"Em trả lời anh ngay lập tức rồi mà!"*.

Cuối cùng, lỗi tại thằng **Shipper** (Network)! 😂

Chúc anh em debug vui vẻ và nhớ: **Check MTU trước khi check code!** 🚀

---
*P/S: Blog này được viết trong lúc đang chờ `tcpdump` chạy.* ☕️