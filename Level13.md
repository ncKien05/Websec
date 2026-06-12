# Đoạn code xử lý logic 
```PHP
<?php

// Defines $flag
include 'flag.php';

$db = new PDO('sqlite::memory:');
$db->exec('CREATE TABLE users (
  user_id   INTEGER PRIMARY KEY,
  user_name TEXT NOT NULL,
  user_privileges INTEGER NOT NULL,
  user_password TEXT NOT NULL
)');

$db->prepare("INSERT INTO users VALUES(0, 'admin', 0, '$flag');")->execute();

for($i=1; $i<25; $i++) {
  $pass = md5(uniqid());
  $user = "user_" . substr(crc32($pass), 0, 2);
  $db->prepare("INSERT INTO users VALUES($i, '$user', 1, '$pass');")->execute();
}
?>
```

```PHP
<?php
if (isset($_GET['ids'])) {
    if ( ! is_string($_GET['ids'])) {
        die("Don't be silly.");
    }

    if ( strlen($_GET['ids']) > 70) {
        die("Please don't check all the privileges at once.");
    }

  $tmp = explode(',',$_GET['ids']);
  for ($i = 0; $i < count($tmp); $i++ ) {
        $tmp[$i] = (int)$tmp[$i];
        if( $tmp[$i] < 1 ) {
            unset($tmp[$i]);
        }
  }

  $selector = implode(',', array_unique($tmp));

  $query = "SELECT user_id, user_privileges, user_name
  FROM users
  WHERE (user_id in (" . $selector . "));";

  $stmt = $db->query($query);

    echo '<br>';
    echo '<div class="well">';
  echo '<ul>';
  while ($row = $stmt->fetch(\PDO::FETCH_ASSOC)) {
        echo "<li>";
        echo "User <em>" . $row['user_name'] . "</em>";
      echo "    with id <code>" . $row['user_id'] . '</code>';
        echo " has <b>" . ($row['user_privileges'] == 0?"all":"no") . "</b> privileges.";
        echo "</li>\n";
  }
    echo "</ul>";
    echo "</div>";
}
?>
```

# Phân tích luồng hoạt động
```PHP
include 'flag.php';
```  
File flag.php được include trực tiếp vào hệ thống  

```PHP
$db->exec('CREATE TABLE users (
  user_id   INTEGER PRIMARY KEY,
  user_name TEXT NOT NULL,
  user_privileges INTEGER NOT NULL,
  user_password TEXT NOT NULL
)');
```
Khởi tạo bảng trong cơ sở dữ liệu

```PHP
$db->prepare("INSERT INTO users VALUES(0, 'admin', 0, '$flag');")->execute();
```  
Thêm người dùng `admin` vào cơ sở dữ liệu có `id=0`,`privileges=0`,`password=$flag`  

```PHP
for($i=1; $i<25; $i++) {
  $pass = md5(uniqid());
  $user = "user_" . substr(crc32($pass), 0, 2);
  $db->prepare("INSERT INTO users VALUES($i, '$user', 1, '$pass');")->execute();
}
```

Khởi tạo 24 người dùng ngẫu nhiên có `id` từ 1->24,`user_pass` ngẫu nhiên, `user_name` ngẫu nhiên dựa trên `user_pass` và đều có `privileges=1`  

```PHP
if ( ! is_string($_GET['ids'])) {
    die("Don't be silly.");
}
```  

Kiểm tra đầu vào:
* Nếu là chuỗi => True
* Khác chuỗi => False và die()  

```PHP
if ( strlen($_GET['ids']) > 70) {
    die("Please don't check all the privileges at once.");
}
```  
Chiều dài của chuỗi nhập vào phải <= 70  

```PHP
$tmp = explode(',',$_GET['ids']);
```  

Khởi tạo mảng `$tmp` được tạo ra bằng cách tách chuỗi nhập vào dựa trên dấu `,`.Ví dụ: `$_GET['ids'] = "1,2,3"` => `$tmp = ["1","2","3"]`  

```PHP
for ($i = 0; $i < count($tmp); $i++ ) {
    $tmp[$i] = (int)$tmp[$i];
    if( $tmp[$i] < 1 ) {
        unset($tmp[$i]);
    }
}
```  

Duyệt các phần từ trong mảng và thực hiện 2 việc:
* Biến các chuỗi trong mảng thành số nguyên
* So sánh số nguyên đó với 1:
  * `>= 1` => TRUE
  * `<1` => FALSE và thực hiện xóa phần tử đó khỏi mảng 

```PHP
$selector = implode(',', array_unique($tmp));
```  

Tạo 1 biến `$selector` mục đích là gộp lại các phần tử trong mảng `$tmp` lại thành 1 chuỗi và thực hiện xóa các phần tử trùng lặp  

```PHP
$query = "SELECT user_id, user_privileges, user_name
  FROM users
  WHERE (user_id in (" . $selector . "));";
```

Tạo biến `$query` truy vấn đến cơ sở dữ liệu  

# Analysis  

Ở ngay đoạn đầu ta đã có thể thấy, cơ sở dữ liệu sẽ có 1 bảng `users` chứa 4 cột là `user_id`,`user_name`,`user_privileges`,`user_password`.  
Sau đó hệ thống tạo 1 dòng dữ liệu với `id=0` là `admin` và `password` là giá trị flag mà chúng ta đang muốn tìm  

Điểm lỗi của bài lab này sẽ nằm ở đoạn này:
```PHP
$tmp = explode(',',$_GET['ids']);
for ($i = 0; $i < count($tmp); $i++ ) {
    $tmp[$i] = (int)$tmp[$i];
    if( $tmp[$i] < 1 ) {
        unset($tmp[$i]);
    }
}
$selector = implode(',', array_unique($tmp));
```  

Sai ở chỗ là,khi chúng ta nhập một số <1, nó chỉ thực hiện xóa phần từ đó khỏi mảng chứ không làm gì quá đặc biệt.  
Khi xóa phần tử khỏi mảng dẫn đến size của mảng cũng sẽ giảm làm sai lệch trong hàm for sẽ xảy ra. Hiểu đơn giản là khi ta nhập vào chuỗi `0,0,0`  
Mảng `$tmp` sẽ có dạng ["0","0","0"] và có size=3  
* Tại vòng lặp đầu tiên (i=0) => tmp[0]=(int)tmp[0]=0 => xóa phần tử khỏi mảng => size(tmp)=2  
* Tại vòng lặp thứ 2 (i=1) => tmp[1]=(int)tmp[1]=0 => xóa phần tử khỏi mảng => size(tmp)=1  
* Tại vòng lặp thứ 3 (i=3) và bùm!! lúc này i>count($tmp) không còn thỏa mãn vòng for nữa và nó sẽ không xử lý bất cứ diều gì  

# Payload exploit  

Như đã nói ở trên, việc của chúng ta chỉ là chèn `0,0,0` vào là đã khai thác được lỗi của bài lab.
```
Mảng `$tmp` lúc này ["0"] -> có 1 phần tử  
Biến `$selector = implode(',', array_unique($tmp));` => ["0"]
`0` được chèn trực tiếp vào `$query` và sẽ in ra `admin` 
```  

Nhưng mà chỉ có như vậy thì vẫn chưa lấy được flag do `$query` chỉ thực hiện `SELECT` 3 cột `id`,`user_privileges`,`user_name`. Mà flag nằm ở cột `user_password`. Chúng ta sẽ phải thực hiện khai thác thêm lỗi SQL Injection.

Payload: `0,0,0,)) UNION SELECT user_password,1,1 FROM users -- -`  

**Lưu ý: Bất cứ khi nào tồn tại dâu `,` thì size của $tmp đều tăng lên, do đó ta phải thực hiện thêm các số <1 vào đầu để làm sai lệch câu lệnh for**

