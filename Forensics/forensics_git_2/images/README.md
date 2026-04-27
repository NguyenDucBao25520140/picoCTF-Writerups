# picoCTF Writeup - bytemancy 3

## Mục tiêu
Dưới đây là mô tả chi tiết từ đề bài:

![Description](./images/Description.png)

Tương tác với chương trình máy chủ, tra cứu và gửi chính xác địa chỉ bộ nhớ (dưới dạng 4-byte little-endian thô) của 3 hàm ngẫu nhiên trong file thực thi spellbook để lấy nội dung file flag.

## Phân tích
Dựa trên các dữ kiện thu thập được:
- **Dấu hiệu:** Chương trình yêu cầu kết nối qua netcat tới nc green-hill.picoctf.net 60107. Khi kết nối, máy chủ in ra lời chào và lần lượt yêu cầu người chơi cung cấp địa chỉ của 3 procedure ngẫu nhiên trong danh sách bao gồm: ember_sigil, glyph_conflux, astral_spark, binding_word.

- **Lỗ hổng:** Bài toán này không yêu cầu khai thác lỗi hỏng bộ nhớ (như buffer overflow), mà đánh giá kỹ năng tự động hóa và trích xuất thông tin từ file ELF. Máy chủ sử dụng hàm read_exact_bytes(4) để nhận chính xác 4 byte dữ liệu thô (raw bytes) từ người chơi và đối chiếu với địa chỉ thực của hàm tương ứng trong file spellbook.

- **Ý tưởng:** Không thể gõ trực tiếp các byte thô từ bàn phím terminal, do đó cần dùng script Python. Sử dụng thư viện pwntools để nạp file spellbook thành đối tượng ELF nhằm tự động lấy địa chỉ hàm thông qua elf.symbols. Script sẽ bóc tách tên hàm từ câu hỏi của máy chủ, chuyển đổi địa chỉ thành chuỗi 4 byte chuẩn little-endian qua hàm p32() và tự động trả lời bằng p.send().

## Khai thác

Các bước thực hiện chi tiét:
1. **Chuẩn bị mã khai thác tự động:**
Sử dụng đoạn script solve.py dưới đây để kết nối và giải quyết tự động 3 vòng thử thách của máy chủ.
```bash
from pwn import *

count = 3
# Khởi tạo đối tượng ELF để dễ dàng tra cứu bảng Symbol
elf = ELF('./spellbook')
# Kết nối tới dịch vụ
p = remote('green-hill.picoctf.net', 60107)

while count > 0:
    # Bỏ qua phần text phía trước và đọc tên hàm nằm trong dấu nháy đơn
    p.recvuntil(b"'")
    data = p.recvuntil(b"'", drop=True)
    symbol = data.decode('utf-8')
    
    # Lấy địa chỉ của hàm từ file ELF
    addr = elf.symbols[symbol]
    
    # Đóng gói địa chỉ thành 4-byte little-endian
    payload = p32(addr)
    
    # Chờ dấu nhắc nhập liệu rồi gửi chính xác 4 byte (không kèm \n)
    p.recvuntil(b"==> ")
    p.send(payload)
    
    count -= 1

# Trả lại quyền tương tác để đọc flag
p.interactive()
```

2. **Thực thi và nhận cờ:**
Chạy script solve.py trên terminal. Đoạn mã sẽ tự động trích xuất tên hàm (ví dụ: ember_sigil), tìm địa chỉ, gửi chuỗi byte tương ứng lên máy chủ 3 lần liên tiếp thành công và in ra flag.
Flag: picoCTF{0bjdump_m4g1c_ef585364}

Các bước được mô tả bằng hình ảnh chi tiết:

![Flag](./images/Flag.png)
