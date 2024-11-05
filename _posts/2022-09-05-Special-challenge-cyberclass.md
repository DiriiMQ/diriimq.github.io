---
title: Special challenge Cyberclass
category: ctf
tags:
    - cyberclass
    - bof
---

Một ngày đẹp trời lướt discord mình bắt gặp được challenge ~~siêu khó~~ của anh Phan Đức trong Cyberclass. Dù bị khá nhiều deadline dí nhưng mình vẫn quyết tâm làm bài đó. Sau đây sẽ là write-up của bài đó.

# Introduction

Khác với những đề ctf mình từng làm thì đề này rất ngắn, chỉ 3 dòng =))).

![problem detail](/assets/images/2022-09-05-Special-challenge-cyberclass/pic1.png)

Sau khi connect tới server thì mình thấy 1 executable file tên là `secret` với quyền `suid`, kết quả sau khi chạy thử như sau:

![secret demo](/assets/images/2022-09-05-Special-challenge-cyberclass/pic2.png)

Có vẻ vấn đề nằm ở chương trình này, nhập vào 1 chuỗi bất kì và in ngược ra terminal. Sau khi làm tới đây thì mình chả biết làm gì nữa =)))). 

<!-- ![con meo ngu](/assets/images/2022-09-05-Special-challenge-cyberclass/pic3.jpg) -->
<center><img src="/assets/images/2022-09-05-Special-challenge-cyberclass/pic3.jpg" alt="drawing" width="200"/></center>

Để ý đề bài thấy flag ở `/root` nên khả năng cao mình sẽ cần leo lên user `root` bằng 1 cách nào đó thông qua file này. Lúc này, mình đã xem lại video dạy về **OS Attack** của Cyberclass. Mình đã thấy kĩ thuật **buffer overflow** (bof) với file `/usr/bin/sudo`. Điểm đặc biệt ở đây là file `/usr/bin/sudo` và `~/secret` đều có điểm chung là được phân quyền `suid`, đồng thời đều nhận tham số vào là 1 chuỗi kí tự. Vì những đặc điểm này nên mình đã phân tích bài này theo hướng bof.

# BOF là cái quái gì =)))

Mình chưa từng làm bài nào về bof nên mình đã quyết định tra google =)))). Sau 7749 các bài blog, mình đã thông não được cơ bản về bof và tóm tắt về nó như sau:

- Bof xảy ra khi có 1 lượng dữ liệu được truyền vào vượt quá mức được khai báo. Trong chall lần này là nhập vào 1 xâu.
- Khi config hệ thống không đúng cách (cho phép execute trên phân vùng có thể write) sẽ khiến cho mình có thể chèn 1 đoạn `shellcode` (đơn giản là đoạn code biểu diễn thành mã hex) vào và khiến nó được execute. 
- Công cụ phổ biến được dùng là `gdb` (GNU Debugger).

Cách hoạt động của kỹ thuật này có thể mô tả như sau:

![overflow stack](https://2603957456-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LFEMnER3fywgFHoroYn%2F-MXwAmlrjE8Ejl_0OQQX%2F-MY1CM45VgNv_HpMZ6jc%2Fimage.png?alt=media&token=1c61c18a-d990-4cc6-a4d2-45244fd7ad4e)

Có thể hiểu phân vùng `local variable` dùng để lưu xâu chúng ta nhập vào. Khi size quá lớn vượt qua phân vùng này thì dữ liệu sẽ chèn lên các phân vùng `ebp`, `return address`, etc. Với chương trình bình thường, sau khi gặp lệnh `ret` (return), chương trình sẽ tìm tới `return address`, phân vùng này sẽ lưu địa chỉ dẫn đến câu lệnh tiếp theo tại `eip` để chương trình chạy. Nhưng điều gì sẽ xảy ra nếu ta control được địa chỉ của câu lệnh tiếp theo =))))). Đó là chương trình sẽ nhảy được đến địa chỉ mình mong muốn và tại đó mình sẽ chèn `shellcode` => chương trình sẽ run đoạn code này. Và thế là mình đã khiến chương trình chạy theo ý bản thân =)))). Giải thích thêm hình trên có thể thấy sau khi bị bof thì vùng `local variable` và `ebp` đã bị lấp đầy chữ **A** (pad). Còn vùng `return address` trỏ lên vị trí bắt đầu của `shellcode`.

_Lưu ý_: Chương trình mình đang làm chạy trên **64-bit** nên các **32-bit** registers như `ebp`, `eip`, etc sẽ thành `rbp`, `rip`, etc. 

Cấu trúc payload có thể tạm hiểu như sau:

```
PAD + RIP + SHELLCODE
```

# GDB dùng như nào

Mình sẽ run `gdb ./secret` để tìm những thông tin hữu ích.

Đầu tiên mình sẽ run `set disassembly-flavor intel`. Sau đấy mình sẽ đọc hàm `main` bằng cách run `disassemble main`.

![disass main](/assets/images/2022-09-05-Special-challenge-cyberclass/pic4.png)

Mình sẽ đặt breakpoint trước `ret` để có thể đọc được các thông tin tại `return address`. Vì thế vị trí tốt nhất đặt breakpoint là ngay lệnh `leave` phía trên `ret` với địa chỉ là `0x0000000000401199`. Mình sẽ đặt breakpoint bằng cách run `break *0x0000000000401199`.

![set breakpoint](/assets/images/2022-09-05-Special-challenge-cyberclass/pic5.png)

Sau đấy, mình dùng lệnh `run` để chạy chương trình.

![run program](/assets/images/2022-09-05-Special-challenge-cyberclass/pic6.png)

# Tìm payload thôii

Mình sẽ tìm lần lượt các phần của payload là `PAD`, `RIP` và `shellcode`.

## Tìm PAD

PAD là phần gồm các ký tự bất kì mục đích để dữ liệu tràn đến ngay trước `return address`. Ở đây, mình dùng là 1 dãy ký tự **A**. Mỗi ký tự **A** sẽ chiếm 1 byte, vì vậy số lượng ký tự **A** cần chèn chính là số lượng `byte` từ `RSP` (vị trí đầu stack) đến `RIP`.

Mình tìm địa chỉ của `RIP` bằng cách gõ `info frame`.

![find rip](/assets/images/2022-09-05-Special-challenge-cyberclass/pic7.png)

Địa chỉ của `RIP` là `0x7fffffffe5c8`.

Tìm địa chỉ `RSP` bằng cách gõ `x/xg $rsp`.

![find rsp](/assets/images/2022-09-05-Special-challenge-cyberclass/pic8.png)

Địa chỉ của `RSP` là `0x7fffffffe1c0`.

Khoảng cách giữa 2 địa chỉ này chính là số lượng cần tìm `p/d 0x7fffffffe5c8 - 0x7fffffffe1c0`.

![calc bytes](/assets/images/2022-09-05-Special-challenge-cyberclass/pic9.png)

Như vậy PAD sẽ là 1032 ký tự **A**.

## Tìm RIP và shellcode

Theo như cấu trúc payload nêu bên trên thì phần `shellcode` nằm ngay sau `RIP`. Vì vậy, `RIP` sẽ lưu `0x7fffffffe5b8` + `0x8` (đây là địa chỉ tiếp theo ngay sau `return address`).

Phần `shellcode` mình sử dụng [ở đây](http://shell-storm.org/shellcode/files/shellcode-77.php).

## Build payload và lên root thôiii

Để build payload như trên thì mình viết code python và xuất payload ra file text. Từ đó, mình trỏ file text vào input của file `secret`. 

Dưới đây là code mình đã viết ra và chạy lên server.

![python code](/assets/images/2022-09-05-Special-challenge-cyberclass/pic10.png)

Hmm thử test trong `gdb` xem chương trình có chạy được không.

![gdb test](/assets/images/2022-09-05-Special-challenge-cyberclass/pic11.png)

Lúc này chương trình gọi được `usr/bin/dash` nhưng có vẻ không nhập được gì hết nên mình sẽ chạy ở ngoài `gdb` xem. Ở ngoài mình chạy `./secret < t.txt` nhưng lại bị segmention fault, còn không gọi được `/bin/sh`. :<

# Khám phá những cái ngu 

Sau khi xem xong video [này](https://www.youtube.com/watch?v=HSlhY4Uy8SA) mình đã nhận ra thay vì dùng `./secret < t.txt` thì mình sẽ dùng `(cat t.txt; cat) | ./secret`. Bởi vì `/bin/sh` không cho mình nhập lệnh thông qua chương trình `secret` nên mình phải dùng `cat` tạo ra 1 pipe để dẫn `stdin` tới `/bin/sh`.

Thử áp dụng cách trên xem sao.

![test](/assets/images/2022-09-05-Special-challenge-cyberclass/pic12.png)

Hmmm vẫn thế :<. Lúc này mình khá là bí nên có đi tìm hiểu, hỏi các anh thì được biết chương trình khi chạy ngoài `gdb` thì địa chỉ sẽ khác so với khi chạy trong `gdb`. Nguyên nhân là 1 số biến môi trường được khởi tạo khác nhau. Để ý khi đọc vài blog về BOF, mình thấy người ta có chèn thêm `NOP` trước `shellcode`. `NOP` được hiểu là 1 dãy ký tự `\x90`, khi chương trình gặp ký tự này nó sẽ không làm gì cả và tiếp tục chạy tiếp. Chính vì điều này, mình đã tạo ra 1 chuỗi `NOP` khá dài trước `shellcode` để nếu địa chỉ có lệch thì chương trình vẫn chạy vào được vùng `NOP`, từ đó chạy đến `shellcode` ở sau. Đồng thời, mình để `RIP` trỏ vào giữa `NOP` để xác xuất lệch ra ngoài `NOP` là thấp nhất.

Mình đã thêm `NOP` bằng cách chỉnh sửa code python như sau.

![adding NOP](/assets/images/2022-09-05-Special-challenge-cyberclass/pic13.png)

Sau đấy mình chạy lại file python để build ra payload. Tiếp đó chạy `gdb`, đặt `breakpoint` và chạy lại chương trình kèm file payload `t.txt` mới tạo. Cuối cùng, mình dùng lệnh `x/100xg $rbp` để chọn địa chỉ thích hợp cho `RIP`.

![find new RIP value](/assets/images/2022-09-05-Special-challenge-cyberclass/pic14.png)

Địa chỉ mình đã chọn là `0x7fffffffe6b0`. Chỉnh sửa lại code python và chạy lại xem nào.

![final python](/assets/images/2022-09-05-Special-challenge-cyberclass/pic15.png)

Chạy `secret` với payload mới và lên `root` thoiii.

![run final payload](/assets/images/2022-09-05-Special-challenge-cyberclass/pic16.png)

Vậy là mình đã lên được `root`, việc đọc flag trong /root bây giờ đã vô cùng dễ :))).

![get flag](/assets/images/2022-09-05-Special-challenge-cyberclass/pic17.png)

Đây là bài [blog](https://0xrick.github.io/binary-exploitation/bof5/) được mình tham khảo rất nhiều trong lúc làm và cả lúc viết wu.

# Vì sao shellcode phải có `setuid(0)`

Có thể các bạn để ý link shellcode mình dùng là `setuid(0) + /bin/sh`. Ban đầu, mình cũng không rõ tại sao phải có `setuid(0)` nhưng sau 1 hồi mò google cuối cùng mình cũng hiểu =))).

Khi một process thực thi sẽ có 3 loại user IDs: _real_ user ID (RUID), _effective_ user ID (EUID), _saved_ user ID (SUID). Khi chương trình C gọi hàm `system`, mặc định, chương trình này sẽ gọi `/bin/sh` để chạy command. Khi gọi `/bin/sh`, chương trình sẽ kiểm tra nếu `RUID` != `EUID` thì `/bin/sh` sẽ chạy với `UID` là `RUID`. Vì vậy, trong trường hợp này, process mình đang chạy có `RUID=1000(cyberclass1337)` và `EUID=0(root)` nên `/bin/sh` gọi ra sẽ có `UID=1000(cyberclass1337)`. Do đó, khi thực hiện `setuid(0)`, đồng thời câu lệnh này được chạy bởi process có `EUID=0` nên `setuid` sẽ thay đổi tất cả các IDs thành **0(root)**. Lúc này `RUID=EUID=0` nên `/bin/sh` gọi ra sẽ được chạy dưới `root`.

Tham khảo từ [đây](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/euid-ruid-suid#sh-and-bash-suid)

_Đây là wu đầu tiên của mình được viết trong thời gian deadline dí rất căng nên có nhiều sai sót mong các bạn thông cảm :<_ 

