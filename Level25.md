# Đoạn code xử lý logic
```PHP
<?php
if (!isset($_GET['page'])) {
  header('Location: http://websec.fr/level25/index.php?page=main');
  die();
}
?>
<div class='container'>
    <div class='row'>
        <label for='user_id'>Enter the page you want to include:</label>
        <form name='username' method='get'>
            <div class='form-group col-md-2'>
                <input type='text' class='form-control' id='page' name='page' value='main' required>
            </div>
            <div class='col-md-2'>
                <input type='submit' class='form-control btn btn-default' name='send'>
            </div>
        </form>
    </div>
    <p class='well'>
        <?php
        parse_str(parse_url($_SERVER['REQUEST_URI'])['query'], $query);
        foreach ($query as $k => $v) {
            if (stripos($v, 'flag') !== false)
                die('You are not allowed to get the flag, sorry :/');
        }
        include $_GET['page'] . '.txt';
        ?>
    </p>
</div>
```  

# Phân tích luồng hoạt động
Dòng 4-7 thực hiện việc kiểm tra bằng `$_GET` xem tham số `page` có bị bỏ trống hay không. Nếu bỏ chống lập tức `die` và về trang `http://websec.fr/level25/index.php?page=main`  

Dòng 14 nhận dữ liệu thông qua ô text và truyền vào biến `page`  

Dòng 22-28 là đoạn xử lý biến đầu vào và cũng là vùng lỗi của bài lab này, quy trình xử lý như sau:
* `parse_url()` nhận vào 1 URI và thực hiện tách các phần tử trong path ra và trả về 1 array  
Ví dụ : nếu url nhập vào là `https://example.com/index.php?x=a` thì kết quả trả về của `parse_url($_SERVER['REQUEST_URI'])` sẽ là:  
```
array(
    'path' => 'index.php',
    'query' => 'x=a'
)
```  
* Lấy ra `query` và gán cho biến `$query` thông qua `parse_str()` phân tách, ta sẽ thu được 1 mảng:
```
$query = [
    "x" => "a"
];
```  

* Vòng lặp `foreach` sẽ lấy ra toàn bộ các cặp key-value trong `$query` và thực hiện so sánh cứng value với 'flag'. Nghĩa là chỉ cần trong value xuất hiện chữ flag thì lập tức sẽ `die()`  

* `$_GET` cũng làm tương tự như vậy,nếu ta nhập url:
  
```
https://example.com/index.php?x=a
```  
Thì `$_GET` sẽ trả về như sau:
```
$_GET = [
    "x" => "a"
];
```  

# Khai thác lỗi
Điều ta cần là làm thế nào để `$_GET` có thể hiểu là ta đang nhập vào value là 'flag' còn hàm `parse_str` và `parse_url` thì không.  

Điểm đặc biệt trong bài này là việc `$_GET` và parse_str(parse_url()) xử lý dấu `/` trong URI như thế nào. Hiểu đơn giản là nếu ta nhập `///index.php?page=flag` thì `$_GET['page']` sẽ là 'flag' còn `parse_str(parse_url())['query']` sẽ là chuỗi rỗng dẫn đến việc vòng lặp `foreach ($query as $k => $v)` sẽ không được thực thi (Do parse_url() kỳ vọng nhập vào 1 URI chuẩn)  

