
[Source](https://www.percona.com/blog/2016/10/12/mysql-5-7-performance-tuning-immediately-after-installation/ "Permalink to MySQL 5.7 Performance Tuning Immediately After Installation")

# MySQL 5.7 Điều chỉnh hiệu năng ngay sau khi cài đặt

Cập nhật của blog này [blog của Stephane Combaudon về  điều chỉnh hiêu năng MySQL][1], và bao gồm điều chỉnh hiệu năng MySQL 5.7 ngay sau khi cài đặt.

Một vài năm trước, Stephane Combaudon đã viết một blog trên [10 thiết lập điều chỉnh hiệu suất của MySQL sau khi cài đặt ][1] bao gồm (ngay bây giờ) phiên bản cũ của MySQL: 5.1, 5.5 và 5.6. Trong bài viết này, Tôi sẽ xem xét những điều chỉnh trong MySQL 5.7 (với a&nbsp;chủ yếu là InnoDB).

Tin tốt là MySQL 5.7 có giá trị mặc định tốt hơn đáng kể.&nbsp;Morgan Tocker đã tạo ra [trang với danh sách đầy đủ các tính năng trong MySQL 5.7][2], và là một tài liệu tham khảo tuyệt vời. Ví dụ, Biến sau đây được đặt _mặc đinh_ :

Trong MySQL 5.7, chỉ có 4 biến quan trong cần được thay đổi. Tuy nhiên, các biến khác của InnoDB và MySQL toàn cục có thể cần phải được điều chỉnh cho một khối lượng công việc và phần cứng cụ thể.

Đẻ bắt đầu, thêm cài đặt này vào trong my.cnf xuống dướt phần [mysqld]. bạn cần phải khởi động lại MySQL:

| ----- |
|   | 

[mysqld]

# Một vài biến khác ở đây

innodb_buffer_pool_size = 1G # (điều chỉnh tại đây, 50%-70% tổng dung lượng RAM)

innodb_log_file_size = 256M

innodb_flush_log_at_trx_commit = 1 # có thể thay đổi thành 2 hoặc 0

innodb_flush_method = O_DIRECT

 | 

Phần mô tả:

| ----- |
| **Biến** |  **giá trị** |  
| innodb_buffer_pool_size |  Bắt đầu với 50%-70% tổng dung lượng RAM.&nbsp;không cần lớn hơn kích thước của CSDL |  
| innodb_flush_log_at_trx_commit | 

* 1&nbsp;&nbsp; (Mặc định)
* 0/2 (hiệu năng tốt hơn nhưng bảo mật kém hơn)
 |  
| innodb_log_file_size |  128M – 2G (không cần phải lớn hơn vùng bộ nhớ đệm) |  
| innodb_flush_method |  O_DIRECT (tránh đệm đôi :|) | 

&nbsp;

_**Tiếp theo là gì?**_

Đó là điểm bắn đầy tốt cho bất kì cài đặt nào.&nbsp;Có một số biến khác có thể  làm tăng hiệu năng của MySQL cho một vài khối lượng công việc cụ thể. Thường thì tôi sẽ thiết lập công cụ theo dõi / đồ thị cho MySQL(ví dụ, the&nbsp;[Percona Monitoring and Management platform][3]) và sau đó kiểm tra bảng điều khiển MySQL để thực hiện điều chỉnh thêm.

_**Chúng ta có thể điều chỉnh gì thêm dựa trên biểu đồ (graph)?**_

_InnoDB buffer pool size_.&nbsp;Nhìn vào biểu đồ:

![MySQL 5.7 Performance Tuning][4]

![MySQL 5.7 Performance Tuning][5]

As we can see, we can probably benefit from increasing the InnoDB buffer pool size a bit to ~10G, as we have RAM available and the number of free pages is small compared to the total buffer pool.

_InnoDB log file size._ Look at the graph:

![MySQL 5.7 Performance Tuning][6]

Như chúng ta thấy ở đây, InnoDB thường ghi 2.26 GB dữ liệu trong một giờ, vượt quá tổng kích thước của các file log (2G). Bây giờ chúng ta có thể tăng biến innodb_log_file_size&nbsp; và khởi động lại MySQL. Ngoài ra, sử dụng "show engine InnoDB status" để [tínhs toán dung lượng file log tốt nhất cho InnoDB][7].

_**Một vìa biến khác**_

Có một số biến của InnoDB có thể điều chỉnh thêm:

_innodb_autoinc_lock_mode_

Thiết lập [innodb_autoinc_lock_mode][8] =2 (chế độ xen kẽ) có thể loại bỏ sự cần thiết cho cấp độ bảng khóa tự động tăng (table-level AUTO-INC lock) (và có thể tăng hiệu suất khi các câu lệnh chèn nhiều hàng được sử dụng để chèn các giá trị vào các bảng có khóa chính auto_increment). Điều này yêu cầu binlog_format=ROW&nbsp; hoặc MIXED&nbsp; (và mặc định là ROW trong MySQL 5.7).

_innodb_io_capacity _and_&nbsp;innodb_io_capacity_max_

Đây là một điều chỉnh nâng cao hơn, và chỉ có ý nghĩa khi bạn đang xử lí rất nhiều lượt ghi trong cùng thời gian (nó không áp dụng cho đọc dữ liệu, vĩ dụ. các câu SELECT). Nếu bạn thật sự cần điều chỉnh nó, cách tốt nhất là biết được có bao nhiêu hệ thống IOPS có thể làm được. Ví dụ, Nếu máy chỉ có một ổ SSD, bạn có thể thiết lập innodb_io_capacity_max=6000&nbsp;và innodb_io_capacity=3000&nbsp;(50% tối đa). Đó là một ý tưởng tốt để chạy sysbench hoặc bất kỳ công cụ benchmark nào khác để chuẩn hóa thông lượng đĩa.

Nhưng chúng ta có cần phải lo lắng về thiết lập này không? Nhìn vào biểu đồ của vùng bộ nhớ đệm "[dirty pages][9]":

![screen-shot-2016-10-03-at-7-19-47-pm][10]

Trong trường hợp này, tổng lượng dirty page là khá cao, và dường như InnoDB không thể flush chúng. Nếu chúng ta có một hệ thống ổ đĩa nhanh (ví dụ., SSD), chúng ta có lợi trong việc tăng innodb_io_capacity&nbsp;và innodb_io_capacity_max.

_**Kết luận hoặc phiên bản TL;DR**_

Các mặc định trong MySQL 5.7 tốt hơn nhiều cho khối lượng công việc chung. Cùng một lúc, chugns ta vẫn cần cấu hình các biến của InnoDB để tận dụng số lượng RAM trên box. Sau khi cài đặt thì làm theo cái bước sau:

1. Thêm các biến InnoDB vào my.cnf (như mô tả ở trên) và khởi động lại MySQL
2. Cài đặt hệ thống giám sát, (ví dụ., Percona Monitoring and Management platform)
3. Nhìn vào biểu đồ và xác định xem liệu MySQL có cần được điều chỉnh thêm hay không

### More resources:

#### Posts

#### Webinars

#### Presentations

#### Free eBooks

#### Tools

### _Related_

![][11]

[Alexander Rubin][12]

Alexander joined Percona in 2013. Alexander worked with MySQL since 2000 as DBA and Application Developer. Before joining Percona he was doing MySQL consulting as a principal consultant for over 7 years (started with MySQL AB in 2006, then Sun Microsystems and then Oracle). He helped many customers design large, scalable and highly available MySQL systems and optimize MySQL performance. Alexander also helped customers design Big Data stores with Apache Hadoop and related technologies.

[1]: https://www.percona.com/blog/2014/01/28/10-mysql-performance-tuning-settings-after-installation/
[2]: http://www.thecompletelistoffeatures.com/
[3]: http://pmmdemo.percona.com
[4]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.49.22-PM.png
[5]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.48.13-PM.png
[6]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.43.52-PM.png
[7]: https://www.percona.com/blog/2008/11/21/how-to-calculate-a-good-innodb-log-file-size/
[8]: http://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html
[9]: http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_page
[10]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-7.19.47-PM.png
[11]: https://secure.gravatar.com/avatar/79877aeedbd68531a30468cd771d5d07?s=84&amp;d=mm&amp;r=g
[12]: https://www.percona.com/blog/author/alexanderrubin/

  
