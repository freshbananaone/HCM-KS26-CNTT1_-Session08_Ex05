# 🚀 Smart-Farm — Hệ thống tưới Lan thông minh

**Tài liệu bàn giao kỹ thuật | Trang trại Lan | Phiên bản 1.0 (bản mẫu học tập)**

> **Trước khi sử dụng:** Đây là cấu hình và mã minh họa cho bài tập IoT, **chưa phải hệ thống được chứng nhận để vận hành không giám sát**. Kiểm tra đấu dây, hiệu chuẩn cảm biến và thử ngắt bơm thực tế trước khi dùng trên giàn Lan.

## 1. Mô tả dự án

**Smart-Farm** là mô hình *giám sát vi khí hậu và hỗ trợ tưới Lan tự động*: cảm biến đo nhiệt độ, độ ẩm không khí, độ ẩm giá thể và mức nước; máy tính nhúng đọc dữ liệu rồi quyết định có tưới hay không. Người vận hành có thể xem nhật ký để biết *khi nào bơm chạy và vì sao hệ thống từ chối tưới*.

Mục đích là **tưới khi giá thể thực sự thiếu ẩm**, không chỉ vì nhiệt độ tăng. Trong bản mẫu này, nếu độ ẩm đất/giá thể dưới ngưỡng, bình còn nước và đã qua thời gian nghỉ, bơm chạy một nhịp ngắn rồi tự tắt. Khi dữ liệu không đáng tin cậy hoặc mức nước không an toàn, hệ thống **ưu tiên không bật bơm**.

**Cấu trúc thư mục:**

```text
Smart-Farm/
├── README.md                      # Tài liệu bàn giao
├── smart_farm.py                  # Chương trình điều khiển Python
├── config.example.json           # Ngưỡng và thời gian mẫu
├── requirements.txt              # Các thư viện cảm biến
├── tests/test_logic.py           # Kiểm thử quy tắc tưới (không bật bơm)
└── assets/
    ├── system-architecture.png   # Sơ đồ phần cứng ngoại tuyến
    ├── system-architecture.dot   # Mã nguồn sơ đồ Graphviz
    └── dikw-pyramid.png          # Hình kim tự tháp DIKW
```

## 2. Phần 1 — Phân tích hệ thống (IPO & SoC)

### 2.1. Xác định Input – Process – Output

| Nhóm | Thành phần | Dữ liệu / nhiệm vụ | Giao tiếp |
|:--|:--|:--|:--|
| **Input** | SHT31 | Nhiệt độ (°C) và độ ẩm không khí (%RH) | I²C |
| **Input** | Cảm biến ẩm điện dung | Điện áp tương ứng độ ẩm giá thể | Analog → ADS1115 |
| **Input** | Công tắc phao | Còn nước hay thiếu nước trong bình | GPIO27 |
| **Process** | Raspberry Pi 4 | Lọc/chuẩn hóa số đo, áp ngưỡng, kiểm tra thời gian nghỉ, ghi log | CPU + RAM trong SoC |
| **Output** | Module relay tương thích 3,3 V | Bật/tắt nguồn điều khiển bơm | GPIO17 |
| **Output** | Bơm DC 12 V | Cấp nước cho gốc Lan | Nguồn 12 V **riêng** |
| **Output** | Tệp nhật ký giới hạn dung lượng | Ghi trạng thái, thời điểm và lỗi | microSD |

### 2.2. Sơ đồ phối hợp phần cứng

![Sơ đồ Smart-Farm: cảm biến nhiệt độ và độ ẩm, cảm biến giá thể qua ADC, phao nước đưa tín hiệu vào Raspberry Pi 4; CPU điều khiển relay, bơm và ghi nhật ký](https://quickchart.io/graphviz?format=png&width=1250&height=550&graph=digraph%20G%20%7Brankdir%3DLR%3Bbgcolor%3Dwhite%3Bnode%5Bshape%3Dbox%2Cstyle%3D%22rounded%2Cfilled%22%2Cfillcolor%3D%22%23EDF5FA%22%2Ccolor%3D%22%23AABDC9%22%2Cfontname%3D%22DejaVu%20Sans%22%2Cfontsize%3D12%5D%3BA%5Blabel%3D%22SHT31%5CnTemp%2FRH%22%5D%3BB%5Blabel%3D%22Soil%20moisture%5CnAnalog%22%5D%3BC%5Blabel%3D%22ADS1115%5CnADC%22%5D%3BD%5Blabel%3D%22Phao%20muc%20nuoc%22%5D%3BE%5Blabel%3D%22Raspberry%20Pi%204%5CnSoC%20CPU%20%2B%20RAM%22%5D%3BF%5Blabel%3D%22Relay%203.3V%22%5D%3BH%5Blabel%3D%22Bom%2012V%22%5D%3BL%5Blabel%3D%22Nhat%20ky%20gioi%20han%22%5D%3BA-%3EE%3BB-%3EC-%3EE%3BD-%3EE%3BE-%3EF-%3EH%3BE-%3EL%3B%7D)

*Hình trên phục vụ xem nhanh trong StackEdit khi có Internet. Bản PNG để dùng ngoại tuyến: [assets/system-architecture.png](assets/system-architecture.png).*

**Đường đi của tín hiệu:** cảm biến SHT31 và bộ ADC ADS1115 dùng chung bus `I2C`; tín hiệu analog từ cảm biến ẩm đất được chuyển sang số tại ADS1115. Công tắc phao gửi trạng thái tới `GPIO27`. SoC xử lý và điều khiển relay qua `GPIO17` để cấp/ngắt nguồn cho bơm.

**Sơ đồ đấu nối tối thiểu (chân BCM của Raspberry Pi):**

| Thiết bị | Kết nối tại Pi | Lưu ý |
|:--|:--|:--|
| SHT31 (`0x44`) | SDA `GPIO2`, SCL `GPIO3`, 3,3 V, GND | Không đưa 5 V vào chân GPIO/I²C 3,3 V |
| ADS1115 (`0x48`) | Chung SDA/SCL, 3,3 V, GND; `A0` nhận tín hiệu ẩm đất | Chỉ dùng cảm biến analog có mức tín hiệu phù hợp 0–3,3 V |
| Phao nước | `GPIO27` và GND | **Giả định:** khi còn nước, phao nối GPIO27 xuống GND; đứt dây = báo không an toàn |
| Relay điều khiển | `GPIO17`, GND, nguồn đúng theo module | Chọn loại nhận mức logic 3,3 V, **active-HIGH**, relay thường hở (NO) |
| Bơm 12 V | Nguồn 12 V qua tiếp điểm/driver phù hợp | Không cấp nguồn bơm từ chân Pi; cần bảo vệ tải cảm ứng |

**SoC (System on a Chip) là gì?** Raspberry Pi 4 sử dụng SoC `BCM2711`, tích hợp **CPU ARM Cortex-A72** và các bộ điều khiển giao tiếp. **RAM là bộ nhớ làm việc bên ngoài SoC**, nơi chương trình và dữ liệu tạm được nạp vào; mã nguồn và nhật ký lưu trên **thẻ microSD**. Nói ngắn gọn: SoC là nơi điều phối, RAM giữ trạng thái đang chạy, còn microSD lưu dữ liệu sau khi tắt điện.

> **An toàn điện:** Chỉ sử dụng module driver/relay được thiết kế cho mức logic GPIO 3,3 V, có mạch bảo vệ phù hợp với dòng khởi động bơm. Dùng tiếp điểm thường hở để bơm tắt khi mất nguồn điều khiển; bổ sung **watchdog phần cứng hoặc mạch giới hạn thời gian chạy độc lập** để phòng CPU treo hay tiến trình bị hệ điều hành kết thúc. Luôn có công tắc cắt nguồn bơm bằng tay.

## 3. Phần 2 — Hướng dẫn cài đặt

**Thiết bị áp dụng:** Raspberry Pi 4, Raspberry Pi OS, SHT31, ADS1115, cảm biến ẩm analog 3,3 V, công tắc phao và module relay/driver phù hợp. Các thông số trong `config.example.json` chỉ là **giá trị giả định để thử nghiệm**.

### Bước 1 — Chuẩn bị Pi

Đảm bảo đã giải nén bộ tài liệu vào thư mục `Smart-Farm`, **chưa kết nối nguồn 12 V cho bơm**. Mở Terminal và chạy:

```bash
cd Smart-Farm
sudo apt update
sudo apt install -y python3-venv python3-gpiozero python3-lgpio i2c-tools
sudo raspi-config
```

Trong `raspi-config`, chọn **Interface Options → I2C → Enable**, sau đó thoát ra. Nếu người dùng hiện tại chưa thuộc nhóm `i2c` và `gpio`, cấp quyền phù hợp rồi **đăng xuất/đăng nhập**:

```bash
sudo usermod -aG i2c,gpio "$USER"
```

Kiểm tra các địa chỉ I²C dự kiến: `0x44` (SHT31) và `0x48` (ADS1115). Nếu chưa thấy, kiểm tra dây và nguồn cảm biến, **không chạy bơm**:

```bash
sudo i2cdetect -y 1
```

### Bước 2 — Cài Python và cấu hình

```bash
python3 -m venv --system-site-packages .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
cp config.example.json config.json
```

Mở `config.json` để sửa ngưỡng sau khi đo thực tế. `soil_dry_voltage` và `soil_wet_voltage` là điện áp ở hai mốc giá thể **khô/ướt đã hiệu chuẩn**; không mặc định áp dụng mọi loại cảm biến.

### Bước 3 — Chạy thử trước khi đấu nguồn bơm

**Mô phỏng trên cả máy tính thông thường**, không đọc chân GPIO và không bật bơm:

```bash
python smart_farm.py --demo
```

Nếu đã đấu cảm biến đúng và vẫn **cắt riêng nguồn 12 V của bơm**, thử đọc/chạy một chu kỳ trên Pi:

```bash
python smart_farm.py --once
```

Sau khi đo chính xác chân kích relay, thử nút ngắt khẩn cấp, kiểm tra phao và bổ sung watchdog phần cứng, mới cấp nguồn bơm để chạy liên tục dưới giám sát:

```bash
python smart_farm.py
```

Nhấn `Ctrl+C` để dừng chương trình. Nhật ký nằm trong thư mục `logs/`. **Không dùng chế độ chạy thực tế trên máy không có GPIO hoặc trước khi xác nhận loại relay đang lắp.**

### Bước 4 — Đọc kết quả và hiểu ngưỡng tưới

Ví dụ, khi `T = 34°C`, độ ẩm giá thể tính được `22%`, phao báo còn nước và lần tưới trước cách đây hơn `600` giây, chương trình cho chạy bơm `10` giây. Nếu một điều kiện không thỏa, hệ thống không bật bơm.

Để biết vì sao chưa tưới, xem vài dòng cuối nhật ký:

```bash
tail -n 20 logs/smart-farm.log
```

## 4. Tính năng

- [x] Đọc nhiệt độ và độ ẩm không khí từ SHT31.
- [x] Đọc điện áp giá thể qua ADC và quy đổi sang phần trăm theo 2 mốc hiệu chuẩn.
- [x] Chỉ tưới khi giá thể khô, còn nước và hết thời gian nghỉ.
- [x] Chặn tưới nếu mất tín hiệu phao, dữ liệu cảm biến lỗi hoặc đang trong thời gian nghỉ.
- [x] Tự tắt bơm sau thời lượng tối đa; kiểm tra phao trong lúc bơm chạy.
- [x] Ghi log xoay vòng; có chế độ `--demo` không tác động phần cứng.
- [ ] Gửi cảnh báo đến điện thoại (hướng phát triển, **chưa có trong mã mẫu**).
- [ ] Giám sát watchdog độc lập (phải lắp thêm trước khi vận hành không giám sát).

## 5. Bảng thông số kỹ thuật tham khảo

| Thông số | Giá trị mẫu | Ghi chú |
|:--|:--|:--|
| Bộ xử lý | Raspberry Pi 4, SoC BCM2711 | CPU ARM Cortex-A72; RAM tùy phiên bản Pi |
| Nhiệt độ cảnh báo | `32°C` | Đánh dấu nóng; **không tự kích hoạt tưới một mình** |
| Ngưỡng ẩm giá thể | `< 30%` | Phải hiệu chuẩn theo loại giá thể trồng Lan |
| Điện áp khô / ướt | `2.55 V` / `1.15 V` | Ví dụ, chiều điện áp tùy cảm biến |
| Thời gian bơm một lượt | `10 s` | Giới hạn cứng trong cấu hình mẫu: `≤ 20 s` |
| Thời gian nghỉ giữa hai lượt | `600 s` | Tránh lặp tưới quá nhanh |
| Chu kỳ lấy mẫu | `30 s` | Không liên tục đóng/ngắt relay |
| Nguồn điều khiển / bơm | `3,3 V logic` / `12 V DC riêng` | Không đấu bơm trực tiếp vào Pi |
| Dung lượng log ứng dụng | `1 MiB × (1 hiện hành + 3 dự phòng)` | Giới hạn gần 4 MiB chưa tính phần phụ |

## 6. Chu kỳ Fetch – Decode – Execute của CPU

> **Fetch (Nạp lệnh):** CPU trong SoC lấy *lệnh máy* của chương trình từ vùng nhớ (sau khi chương trình đã được nạp từ microSD vào RAM). Ví dụ: các lệnh đọc giá trị cảm biến và kiểm tra điều kiện tưới.
>
> **Decode (Giải mã):** CPU giải mã lệnh thành thao tác cần thực hiện; chương trình đồng thời sử dụng giá trị cảm biến đã đọc để so sánh với ngưỡng `30%`, trạng thái phao và thời gian nghỉ.
>
> **Execute (Thực thi):** CPU thực thi các phép so sánh và lệnh ghi GPIO. Nếu các điều kiện hợp lệ, tín hiệu `GPIO17` kích relay, bơm chạy tối đa `10 s` rồi tắt; nếu không, chân điều khiển duy trì trạng thái tắt. Chu kỳ lệnh CPU lặp lại liên tục trong suốt quá trình này, **không phải một chu kỳ CPU kéo dài 10 giây**.

## 7. DIKW — Từ dữ liệu thành quyết định tưới

DIKW gồm **Data (Dữ liệu) → Information (Thông tin) → Knowledge (Tri thức) → Wisdom (Quyết định/hành động hợp lý)**. Có thể hình dung từ đáy đến đỉnh kim tự tháp:

```text
                    /-------------------------\
                   /  WISDOM — HÀNH ĐỘNG      \
                  /  Tưới 10 giây, có giám sát  \
                 /-----------------------------\
                /  KNOWLEDGE — TRI THỨC        \
               /  Đất khô + nước đủ + hết nghỉ \
              /---------------------------------\
             /  INFORMATION — THÔNG TIN         \
            /  34°C > 32°C; ẩm đất 22% < 30%    \
           /-------------------------------------\
          /  DATA — DỮ LIỆU                      \
         /  SHT31 = 34°C; ADC = 2,24 V; phao OK   \
        /_________________________________________\
```

[Bản minh họa kim tự tháp DIKW bằng hình ảnh (PNG)](assets/dikw-pyramid.png).

| Tầng DIKW | Dữ liệu điển hình | Ý nghĩa với người chăm Lan |
|:--|:--|:--|
| **Data** | Nhiệt độ `34°C`, điện áp ẩm đất `2,24 V`, phao = còn nước | Các giá trị đọc từ thiết bị |
| **Information** | Nhiệt độ vượt mốc `32°C`; ẩm đất sau hiệu chuẩn xấp xỉ `22% < 30%` | Xác định môi trường nóng và giá thể thiếu ẩm |
| **Knowledge** | Đất đang khô; có nước; đã đủ thời gian nghỉ | Biết rằng tưới một lượt ngắn có thể phù hợp |
| **Wisdom** | **Cho tưới 10 giây**, theo dõi phao; giữ khả năng ngắt thủ công | Hành động có giới hạn và điều kiện an toàn |

**Lưu ý nghiệp vụ:** nhiệt độ chỉ cung cấp *bối cảnh*; quyết định tưới dựa trước hết vào độ ẩm giá thể và các điều kiện an toàn. Với Lan trồng trong vỏ thông, dớn hoặc giá thể khác, phải xác nhận lại cách hiệu chuẩn thực tế.

## 8. Phần 3 — Kiểm thử trước bàn giao

Chạy bộ kiểm thử logic (không cần lắp Pi và **không bật bơm**):

```bash
python -m pip install pytest
python -m pytest -q tests/
```

| Tình huống kiểm thử | Kết quả mong đợi |
|:--|:--|
| Nhiệt độ cao `37°C`, nhưng ẩm đất `66%` | **Không tưới** vì đất đủ ẩm |
| Đất khô `22%`, bình hết nước / đứt dây phao | **Không tưới** và cảnh báo |
| Đất khô, vừa tưới cách đây `20 s` | **Không tưới**, chờ hết `600 s` |
| Đất khô, còn nước, hết thời gian nghỉ | Tưới đúng một nhịp, sau đó **tắt** |
| Ngắt chương trình bằng `Ctrl+C` | Chương trình cố gắng đưa GPIO về tắt |
| Cảm biến không trả dữ liệu | Ghi lỗi và **không tưới** |
| RAM đầy vì nhật ký hoặc vùng `/run` đầy | Ngắt bơm bằng phần cứng, điều tra và phục hồi có kiểm soát theo mục **Lưu ý** |

## 9. Lưu ý — Xử lý an toàn khi RAM đầy dữ liệu nhật ký

**Giả lập sự cố:** dịch vụ ghi quá nhiều nhật ký vào RAM (ví dụ `journald` lưu log tạm dưới `/run/log/journal`), khiến RAM khả dụng giảm mạnh hoặc tiến trình tưới bị dừng đột ngột. Cũng cần phân biệt với trường hợp **thẻ nhớ đầy** vì log lưu trên ổ đĩa; hai lỗi này khác nhau.

**Quy trình xử lý cho người vận hành:**

1. **Ưu tiên an toàn:** ngắt nguồn riêng cấp cho **bơm** bằng công tắc/cầu dao được bố trí sẵn; tiếp tục giữ Pi hoạt động nếu còn an toàn để kiểm tra. Nếu chạy trực tiếp trong Terminal, bấm `Ctrl+C`. **Đừng tin rằng lỗi RAM sẽ tự tắt bơm**: GPIO có thể giữ trạng thái sau khi tiến trình chết nếu không có watchdog phần cứng.
2. **Kiểm tra nguyên nhân:** sau khi bơm đã an toàn, xem lượng RAM còn lại và dung lượng vùng chứa log. Các lệnh sau **chỉ đọc**, không xóa dữ liệu:

   ```bash
   free -h
   df -h /run /var/log
   ps -eo pid,rss,comm --sort=-rss | head
   sudo journalctl --disk-usage
   ```

3. **Giữ lại bằng chứng:** lưu các đoạn log quan trọng và thời điểm xảy ra lỗi ra thiết bị/lưu trữ còn đủ dung lượng; thông báo người phụ trách kỹ thuật. Không dùng lệnh xóa hàng loạt như `rm -rf /var/log/*`, không xóa bừa `/run` hoặc ép dọn cache khi chưa biết nguyên nhân.
4. **Khắc phục có giới hạn:** ứng dụng mẫu dùng `RotatingFileHandler` nên mỗi file chỉ khoảng `1 MiB`, giữ tối đa 3 bản cũ; không gom toàn bộ log vào danh sách trong RAM. Nếu vấn đề đến từ nhật ký hệ thống, quản trị viên có thể cấu hình giới hạn `RuntimeMaxUse` cho `systemd-journald`, sau khi xác nhận vị trí log và sao lưu phần cần giữ. Nếu lỗi do RAM tiến trình tăng liên tục, kiểm tra rò rỉ bộ nhớ thay vì chỉ xóa log.
5. **Khởi động lại có kiểm soát:** chỉ chạy lại khi RAM/dung lượng đã ổn, dữ liệu cảm biến hợp lệ, phao hoạt động và relay thử ở trạng thái **OFF**. Cấp lại nguồn bơm cuối cùng, theo dõi ít nhất một chu kỳ và đảm bảo watchdog/giới hạn thời gian phần cứng hoạt động.

**Phòng ngừa:** giới hạn kích thước nhật ký ứng dụng và nhật ký hệ điều hành; giám sát RAM/ổ đĩa; lưu bản sao log quan trọng sang nơi lưu trữ khác; lắp bộ **ngắt bơm độc lập khi treo máy**. Hàm `finally: pump.off()` trong Python chỉ hỗ trợ dừng thông thường, **không thay thế được mạch bảo vệ khi mất điện, OOM hoặc chương trình bị kill cưỡng bức**.

---

**Bàn giao cho trang trại Lan:** Người sử dụng chỉ cần theo dõi độ ẩm, mức nước và nhật ký; khi có cảnh báo hoặc bơm hoạt động bất thường, **cắt nguồn bơm và liên hệ kỹ thuật**, không tự thay dây điện khi hệ thống còn cấp nguồn. Các thông số là ví dụ phục vụ bài tập và phải được kiểm chứng trên giàn Lan thật trước khi vận hành.
