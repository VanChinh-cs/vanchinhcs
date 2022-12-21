# vanchinhcs
#A. Development
##I. Thư viện
- [Gradle Build Tool](https://gradle.org/)
- [Spring Boot](https://spring.io/projects/spring-boot)

Api được xây dựng trên framework Spring Boot và sử dụng build tool là Gradle.

#### Java version
- Api hoạt động tốt trên JDK 1.8
- JDK version >= 1.8_171
#### IDE sử dụng
- Khuyến nghị IntelliJ

##II. Cấu trúc
#### 1. Cấu trúc Project trong Tvan
- TvanUI        :   Project chứa code giao diện portal
- ApiPortal     :   Project chứa code api portal: đăng nhập, quản lý tài khoản, tra cứu...
- ApiGW         :   Project chứa code api gateway: cung cấp đầu api dịch vụ cho khách hàng
- TVanProcess   :   Project chưa code process xử lý: validate message gửi lên, build xml gửi thuế, nhận xml kết quả và build json phản hồi...
- TVanSender    :   Project chứa code xử lý gửi message sang thuế
- TVanReceiver  :   Project chứa code xử lý nhận message phản hồi từ thuế, lưu vào database và publish kafka
- MmCommon      :   Đây là project chứa code common, tạo các hàm sử dụng chung cho tvan: Kiểm tra authen api, load config db tự động, kết nối kafka, kết nối database...Lưu ý: MmCommon là lib chung, nếu có chỉnh sửa thì cần build lại và ghi đè lại vào thư mục lib của các project ứng dụng.
#### 2.Cấu trúc thư mục và ý nghĩa:

- etc: thư mục chứa file cấu hình
    |-- application.yml         :   File cấu hình chung
    |-- log.xml                 :   File cấu hình ghi log
- libs: thư mục lib local, lib dùng chung
    |-- MmCommon-1.0.jar        :   Lib chứa các hàm sử dung chung như kết nối db, kafka, config, kiểm tra quyền...
    |-- ojdbc8.jar              :   Lib sử dụng để kết nối db oracle (hiện không sử dụng)
- src
    |-- tvan
        |-- process
            |-- impl  
                |-- ConfigEnum.java     :   Enum config (map với bảng config)
            |-- service
                |-- responseinv
                    |-- build
                        |-- EinvoiceLoadCommon.java         :   Build chi tiết từ xml => json 
                    |-- handle
                        |-- BuildTaxResponse.java           :   Xử lý nghiệp vụ với từng message phản hồi từ thuế
                    |-- kafka
                        |-- KafkaReceiverTaxConsumer.java   :   consumer topic phản hồi từ thuế với topic name = config: TVAN_KAFKA_RECEIVER_TAX_TOPIC
                    |-- CreateEinvoinceRunnable.java        :   worker lấy các message phản hồi từ thuế đã được đẩy vào queue trên Ram và xử lý
                    |-- ScanEinvoiceDbRunnable.java         :   class nhận message phản hồi từ thuế thông qua quét db hoặc từ kafka rồi đẩy vào queue trên Ram,
                    xử lý tạo json phản hồi và lưu vào db
                |-- sendtax
                    |-- handle
                        |-- BuildRequest.java           :   Xử lý kiểm tra message tiếp nhận, build xml và lưu vào database để Sender gửi sang thuế
                    |-- kafka
                        |-- KafkaReceiverTaxConsumer.java   :   consumer topic tiếp nhận để gửi sang thuế với topic name = config: TVAN_KAFKA_VALIDATE_TOPIC
                    |-- ScanTaxDbRunnable.java          :   class nhận message yêu cầu gửi sang thuế thông qua quét db hoặc từ kafka rồi đẩy vào queue trên Ram
                    |--  SendTaxRunnable.java           :   worker lấy các message cần xử lý validate và build message gửi thuế đã được đẩy vào queue trên Ram để xử lý build message và lưu vào db
        |-- Start.java: Class main, start chương trình- File cấu hình và ý nghĩa
### 1.CONFIG: PORT, CONTEXT PATH ###
#server.servlet.context-path: /zuul+
server:
   port: ${PORT:8088} #Đây là cấu hình port của api sau khi start
   error:
      include-message: always
   enable-cors: true   #Đây là cấu hình cho phép nhận request không cùng domain
   route:
      jwt:
         secret: 7xC01tFcEnhIoUYf #jwt chuoi bi mat
spring:
   jndi:
      ignore: true #fix loi bao mat log4j
   main:
      web-application-type: none #Config tắt khởi tạo api
###2.DB CONFIG ###
datasource:
   db-app: #Cấu hình kết nới tới database posgresql
      driver-class-name: org.postgresql.Driver
      jdbc-url: jdbc:postgresql://10.14.185.14:5432/tvandb_test?prepareThreshold=0
      user-name: postgres
      pass-word: postgres#123
app:#B. BUILD chương trình
##I. Build server on premise

Các bước để tạo bản build và cài đặt trên server:

Build chương trình => Copy code build và cấu hình vào chung một thư mục => Chạy chương trình####B1: Build code
- Chạy lệnh sau để build api
 ./gradlew clean build
hoặc 
    Sử dụng gradle trên IntelliJ để build - Truy cấp thư mục build tại:
 ./build/distributions/- Giải nén file build zip và copy thư mục etc vào trong thư mục zip vừa giải nén
- Copy file run_systemctl.sh, shutdown.sh, start.sh vào thư mục vừa giải nén này (Cloud thì không cần thực hiện bước này)
- Thư mục build code sau các bước trên:
- TVanProcess-1.0-SNAPSHOT
    |-- lib
    |-- TVanProcess-1.0-SNAPSHOT-plain.jar
    |-- etc
    |-- run_systemctl.sh
    |-- shutdown.sh
    |-- start.sh####B2: Start
- Sau khi đã tạo được bản build, thực hiện cập nhật thông tin database trong file application.yml, ta sẽ thực hiện cấp quyền script start chương trình.
- Khởi tạo: chỉ chạy lần đầu
chmod 755 -R ./run_systemctl.sh
chmod 755 -R ./start.sh
chmod 755 -R ./shutdown.sh
sudo ./run_systemctl.sh createTrong đó:
|-- SERVICE_NAME:  trong file run_systemctl.sh sẽ là tên service trong systemctl.
- Start: 
sudo ./run_systemctl.sh start- Stop: 
sudo ./run_systemctl.sh stop- Restart: 
sudo ./run_systemctl.sh restart
#####Lưu ý:
- Nếu có lỗi xảy ra, thực hiện kiểm tra file log trong thư mục logger:

|--logger
    |-- db_error        :   Thư mục chứa log lỗi db
    |-- kafka_error     :   Thư mục chứa log lỗi kafka
    |-- full.log        :   File chứa log lỗi chung, bao gồm cả log lỗi db và log lỗi kafka
