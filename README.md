# Xây dựng môi trường mô phỏng xử lý sự cố bảo mật sử dụng Wazuh, TheHive và Shuffle

## 1. Tổng quan
*  Xây dựng một môi trường SOC mô phỏng, tích hợp Wazuh, TheHive, VirusTotal và Shuffle SOAR để phát hiện, phân tích, và phản ứng tự động trước các mối đe dọa như Mimikatz và SSH brute-force.
*  Tuy còn đơn giản so với hệ thống SOC thực tế, nhưng hoàn toàn có thể mở rộng thêm các nguồn log, tích hợp thêm công cụ threat intel, hoặc bổ sung các kịch bản phản ứng phức tạp hơn. Đây là nền tảng vững chắc để phát triển kỹ năng vận hành và tự động hóa SOC trong môi trường thực tế.
*  Lợi ích:
    *   Hiểu rõ kiến trúc và luồng dữ liệu từ thu thập log, phân tích rule, đến phản hồi sự cố.
    *   Thực hành viết custom rules trong Wazuh để nâng cao khả năng phát hiện.
    *   Sử dụng Shuffle để tự động hóa enrichment, tạo case, gửi email và kích hoạt Active Response.
    *   Áp dụng Active Response để chặn IP độc hại trực tiếp từ agent.

## 2. Các thành phần và kiến trúc hệ thống
* Mô hình lab xây dựng bao gồm nhiều thành phần mã nguồn mở, phổ biến trong các tổ chức thực tế:
    *   `Wazuh` được triển khai như một SIEM (Security Information and Event Management) để thu thập log từ các máy chủ và endpoint, đồng thời tạo ra cảnh báo dựa trên rule.
    *   `TheHive` đóng vai trò một nền tảng quản lý sự cố (incident response platform), nơi các cảnh báo từ Wazuh được chuyển thành case để analyst xử lý.
    *   `VirusTotal` được sử dụng để thực hiện enrichment các IOC (như IP, hash, domain…) bằng cách quét và phân tích với hơn 70 công cụ antivirus và dịch vụ khác, giúp quá trình điều tra nhanh chóng và hiệu quả hơn.
    *   `Shuffle` một nền tảng SOAR (Security Orchestration, Automation, and Response) sử dụng để xây dựng các playbook nhằm tự động hóa các hành động như gửi cảnh báo, phản ứng như block IP,…
* Sơ đồ kiến trúc:

    ![Hình 1 sơ đồ kiến trúc](images/h1.png)

* Thiết bị:
    *   SIEM: máy ảo Wazuh OVA
    *   TheHive+Shuffle: chạy docker trên máy ảo ubuntu server
    *   Wazuh agent: máy ảo windows 10, máy ảo ubuntu
    *   SOC analyst: máy thật windows 11
 


## 3. Triển khai
### Kịch bản 1 (mimikatz)
Cấu hình đẩy log sysmon:

![Hình 2 Cấu hình đẩy log sysmon](images/h2.png)

Viết rule cho mimikatz:

![Hình 3 Viết rule cho mimikatz](images/h3.png)

Chạy `mimikatz.exe` và xem alert trên wazuh

Trường `win.eventdata.originalFileName` tham chiếu đến tên gốc được nhúng trong siêu dữ liệu (metadata) của tệp; tên này không thay đổi ngay cả khi tệp được đổi tên trên ổ đĩa. Do đó, nếu một kẻ tấn công chỉ đơn thuần đổi tên tệp `mimikatz.exe` thành một tên khác, quy tắc này vẫn sẽ kích hoạt cảnh báo vì tên tệp gốc vẫn là `mimikatz.exe`.

Chúng ta hãy thử đổi tên tệp `mimikatz.exe` thành một tên khác, thực thi tệp đó và kiểm tra xem Wazuh có tạo cảnh báo hay không.

![Hình 4 chạy mimikatz và đổi tên](images/h4.png)
![Hình 5 alert mimikatz1](images/h5_1.png)
![Hình 5 alert mimikatz2](images/h5_2.png)

Nếu dựa vào trường `win.eventdata.image` thì alert sẽ ko được kích hoạt

Thiết lập Shuffle và TheHive
*   Workflow:
    1. Nhận cảnh báo: Hệ thống phát hiện Mimikatz và gửi alert sang Shuffle.
    2. Xử lý ban đầu: Shuffle nhận alert, trích xuất SHA256 hash từ file bị nghi ngờ.
    3. Phân tích: Gửi hash lên VirusTotal để kiểm tra mức độ uy tín.
    4. Tạo hồ sơ điều tra: Gửi thông tin sang TheHive để tạo alert cho SOC.
    5. Thông báo nhân sự: Gửi telegram cho SOC analyst để bắt đầu điều tra.

    ![Hình 6 workflow mimikatz](images/h6.png)

*   Chi tiết:
    *   Tạo webhook để nhận alert từ wazuh

        ![Hình 7 webhook](images/h7.png)

    *   Cấu hình gửi alert qua cho webhook trên shuffle ở file `ossec.conf`

        ![Hình 8 intergration](images/h8.png)

    *   Tạo Execute python để trích xuất mã hash sha256 để đẩy lên virustotal

        ![Hình 9 python1](images/h9_1.png)
        ![Hình 9 python2](images/h9_2.png)

    *   Tạo node Virustotal để phân tích mã hash sha256

        ![Hình 10 virustotal](images/h10.png)

    *   Chuyển sang TheHive, ta sẽ tạo 1 tổ chức mới tên `SOC-lab` và tạo 1 người dùng mới `user1` để nhận hồ sơ điều tra là `user1@thehive.local`

        ![Hình 11 user1](images/h11.png)

    *   Tạo node TheHive 

        ![Hình 12 thehive1](images/h12_1.png)
        ![Hình 12 thehive2](images/h12_2.png)

    *   Tạo node telegram

        ![Hình 13 tele1](images/h13_1.png)
        ![Hình 13 tele2](images/h13_2.png)

    *   Test workflow và kiểm tra alert trên TheHive và telegram

        ![Hình 14 thehive alert](images/h14_1.png)
        ![Hình 14 tele alert](images/h14_2.png)


### Kịch bản 2 (SSH attack)
Cấu hình đẩy log linux:

![Hình 15 Cấu hình đẩy log linux](images/h15.png)

Ta sẽ sử dụng rule id `5763` có sẵn của wazuh cho ssh attack:

![Hình 16 rule cho ssh attack](images/h16.png)

Tấn công từ máy kali và xem alert trên wazuh

![Hình 17 kali attack](images/h17.png)
![Hình 18 alert ssh](images/h18.png)

Thiết lập Shuffle và TheHive
*   Workflow:
    1. Nhận cảnh báo: Hệ thống phát hiện SSH attacke và gửi alert sang Shuffle.
    2. Xử lý ban đầu: Shuffle nhận alert, trích xuất source ip.
    3. Phân tích: Gửi ip lên VirusTotal để phân tích.
    4. Tạo hồ sơ điều tra: Gửi thông tin sang TheHive để tạo alert cho SOC.
    5. Thông báo nhân sự: Gửi telegram cho SOC analyst để quyết định có chặn ip không.

    ![Hình 19 workflow ssh](images/h19.png)

*   Chi tiết:
    *   Tạo webhook để nhận alert từ wazuh

        ![Hình 20 webhook](images/h20.png)

    *   Cấu hình gửi alert qua cho webhook trên shuffle ở file `ossec.conf`

        ![Hình 21 intergration](images/h21.png)

    *   Tạo node Shuffle workflow để trích xuất source ip đẩy lên virustotal

        ![Hình 22 shuffle workflow](images/h22.png)

    *   Tạo node Virustotal để phân tích ip

        ![Hình 23 virustotal](images/h23.png)

    *   Tạo node user input nhằm tạo subflow để thông báo đến telegram giúp người phân tích đưa ra quyết định tức thì

        ![Hình 24 subflow1](images/h24_1.png)
        ![Hình 24 subflow2](images/h24_2.png)

    *   Vì ta sử dụng tính năng active reponse của wazuh để gọi đến firewall chặn ip nên ta cần lấy api key của wazuh bằng cách tạo node http và dùng curl

        ![Hình 25 apikeywazuh](images/h25.png)

    *   Ta cũng cần cấu hình file `ossec.conf` để kích hoạt active reponse

         ![Hình 26 conf active rep](images/h26.png)

    *   Tạo node TheHive

        ![Hình 27 thehive1](images/h27_1.png)
        ![Hình 27 thehive2](images/h27_2.png)

    *   Test workflow kiểm tra alert trên TheHive, telegram 

        ![Hình 28 thehive alert](images/h28_1.png)
        ![Hình 28 tele alert](images/h28_2.png)

    *   Dựa trên tin nhắn được gửi ở telegram, URL đầu tiên được sử dụng khi cần chặn IP nguồn còn URL thứ hai thì không. Do đó, chúng ta sẽ sử dụng URL đầu tiên để chặn IP.

        ![Hình 29 blockip](images/h29.png)

    *   Vì ta cấu hình đẩy cả log active reponse về wazuh nên ta có thể kiểm tra xem liệu có block thành công không

        ![Hình 30 log ar1](images/h30_1.png)
        ![Hình 30 log ar2](images/h30_2.png)





