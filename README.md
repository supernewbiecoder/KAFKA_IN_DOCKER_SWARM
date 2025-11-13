# KAFKA_IN_DOCKER_SWARM
Tài liệu này là trải nghiệm của tôi về quá trình và cách dựng cụm kafka trong docker swarm

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [KAFKA_IN_DOCKER_SWARM](#kafka_in_docker_swarm)
  - [1. Docker swarm](#1-docker-swarm)
    - [1. Docker swarm là gì](#1-docker-swarm-là-gì)
    - [2. Đặc điểm nổi bật của Docker swarm](#2-đặc-điểm-nổi-bật-của-docker-swarm)
    - [3. Cách hoạt động của Docker Swarm](#3-cách-hoạt-động-của-docker-swarm)
      - [*1. Cách node hoạt động*](#1-cách-node-hoạt-động)
      - [*2.Cách service hoạt động*](#2cách-service-hoạt-động)
      - [*Bảo mật trong Docker Swarm bằng PKI (Public Key Infrastructure)*](#bảo-mật-trong-docker-swarm-bằng-pki-public-key-infrastructure)
      - [*Trạng thái của Task trong Docker Swarm*](#trạng-thái-của-task-trong-docker-swarm)
      - [*Tóm lại*](#tóm-lại)
  - [Kafka](#kafka)
    - [1. Vấn đề mà Kafka giải quyết](#1-vấn-đề-mà-kafka-giải-quyết)
      - [*1. Mớ hỗn độn của các Kết nối Trực tiếp*](#1-mớ-hỗn-độn-của-các-kết-nối-trực-tiếp)
      - [*2. Sự khác biệt về Tốc độ xử lý*](#2-sự-khác-biệt-về-tốc-độ-xử-lý)
      - [*3. Mất dữ liệu và Thiếu tin cậy*](#3-mất-dữ-liệu-và-thiếu-tin-cậy)
      - [*4. Dữ liệu bị cô lập trong các kho dữ liệu (Data Silos)*](#4-dữ-liệu-bị-cô-lập-trong-các-kho-dữ-liệu-data-silos)
    - [2. Các thành phần trong KAFKA](#2-các-thành-phần-trong-kafka)
  - [Kafka trong Docker Swarm.](#kafka-trong-docker-swarm)

<!-- /code_chunk_output -->


---
## 1. Docker swarm

### 1. Docker swarm là gì
>Docker Swarm là công cụ quản lý cụm (cluster management tool) tích hợp sẵn trong Docker, cho phép bạn chạy và điều phối nhiều container Docker trên nhiều máy (node) như thể chúng là một hệ thống duy nhất.

### 2. Đặc điểm nổi bật của Docker swarm
**1. Quản lý cụm tích hợp với Docker Engine**
Docker Swarm được tích hợp sẵn trong Docker Engine, cho phép bạn tạo một cụm (swarm) gồm nhiều Docker Engine để triển khai ứng dụng mà không cần cài thêm công cụ điều phối khác (như Kubernetes).

**2. Thiết kế phi tập trung (Decentralized design)**
Docker Engine tự xử lý việc phân biệt giữa manager và worker trong lúc chạy, không cần tách biệt khi cài đặt.
👉 Bạn có thể tạo cả cụm Swarm chỉ từ một image duy nhất.

**3. Mô hình dịch vụ khai báo (Declarative service model)**
Docker Swarm dùng mô hình khai báo trạng thái mong muốn (desired state).
Ví dụ: bạn mô tả ứng dụng gồm web frontend, message queue, và database backend, Docker sẽ tự triển khai đúng như mô tả đó.

**4. Tự động mở rộng (Scaling)**
Bạn có thể chỉ định số lượng task/container cần chạy cho mỗi service.
Khi bạn tăng hoặc giảm số lượng, Swarm sẽ tự động thêm hoặc xóa container để đảm bảo đúng trạng thái mong muốn.

**5. Tự động khôi phục trạng thái mong muốn (Desired state reconciliation)**
Swarm Manager liên tục giám sát trạng thái cụm, và nếu phát hiện lệch so với cấu hình, nó sẽ tự sửa.
Ví dụ: bạn muốn chạy 10 bản sao (replica) mà 1 máy hỏng mất 2 bản → Swarm sẽ tự khởi tạo lại 2 bản đó trên máy khác.

**6. Mạng nhiều host (Multi-host networking)**
Swarm hỗ trợ overlay network – cho phép container ở các node khác nhau giao tiếp như cùng một mạng nội bộ.
Khi khởi tạo hoặc cập nhật dịch vụ, Swarm tự cấp địa chỉ IP cho container.

**7. Khám phá dịch vụ (Service discovery)**
Swarm Manager gán DNS name duy nhất cho từng service.
Mọi container trong swarm đều có thể truy cập service khác qua DNS và tự động load balancing.

**8. Cân bằng tải (Load balancing)**
Có thể mở port ra ngoài cho các service hoặc điều phối nội bộ cách phân phối container giữa các node.
Swarm sẽ tự động chia tải giữa các container cùng service.

**9. Bảo mật mặc định (Secure by default)**
Tất cả các node trong Swarm giao tiếp an toàn bằng TLS, với xác thực hai chiều (mutual authentication).
Bạn có thể dùng chứng chỉ tự ký hoặc chứng chỉ từ CA riêng.

**10. Cập nhật luân phiên (Rolling updates)**
Khi triển khai bản cập nhật, Swarm có thể cập nhật dần từng nhóm node theo thời gian bạn quy định.
Nếu có lỗi, bạn có thể rollback (quay lại phiên bản cũ). 

### 3. Cách hoạt động của Docker Swarm
#### *1. Cách node hoạt động*
![](https://github.com/supernewbiecoder/KAFKA_IN_DOCKER_SWARM/blob/main/images/nodesInSwarm.png?raw=true)

**1. Manager Node**
- Chức năng chính: Quản lý cụm, bao gồm:
    - Duy trì trạng thái cluster bằng Raft
    - Lên lịch các service
    - Cung cấp Swarm mode HTTP API.
- Độ chịu lỗi:
    - Số lượng swarm manager nên là số lẻ để tận dụng khả năng chịu lỗi
    - Một số lượng lẻ N manager còn có thể hoạt động kể cả khi mất đi nhiều nhất là (N-1)/2 manager.
> Lưu ý: Thêm nhiều manager không tăng hiệu năng hay khả năng mở rộng; đôi khi còn giảm hiệu năng.

**2. Worker Node**
- Chức năng chính: Chỉ thực thi container, không tham gia Raft, không ra quyết định thêm lịch, không cung cấp API.
- Quy tắc:
    - Luôn cần ít nhất một manager mới có thể dùng worker.
    - Mặc định, manager cũng là worker.
- Drain mode:
    - Khi muốn ngăn manager nhận task, đặt manager ở chế độ Drain.
    - Scheduler sẽ dừng các task trên node này và chuyển sang node Active.

#### *2.Cách service hoạt động*
>Khi Docker Engine chạy ở Swarm mode, để triển khai một ứng dụng, bạn sẽ tạo một service.
Mỗi service thường đại diện cho một microservice trong một ứng dụng lớn hơn (ví dụ: HTTP server, cơ sở dữ liệu, hoặc bất kỳ chương trình thực thi nào bạn muốn chạy trong môi trường phân tán).

Khi tạo service, bạn cần chỉ định:
- Image sẽ sử dụng.
- Lệnh thực thi bên trong container.
- Các tùy chọn cấu hình như:
    - Port mà service sẽ mở ở ngoài Swarm.
    - Overlay network để kết nối với các service khác trong cụm.
    - Giới hạn CPU, bộ nhớ.
    - Chính sách cập nhật luân phiên (rolling update policy).
    - Số lượng bản sao (replicas) cần chạy trong swarm.

Khi triển khai service, Swarm Manager nhận định nghĩa service như một trạng thái mong muốn (desired state).
Nó sẽ phân bổ service đó lên các node trong Swarm dưới dạng các task replica — mỗi task là một instance độc lập chạy trong container.


![](https://github.com/supernewbiecoder/KAFKA_IN_DOCKER_SWARM/blob/main/images/service.png?raw=true)

    Container là một tiến trình độc lập (isolated process).
    Mỗi task chỉ tương ứng với một container duy nhất.
    Khi container đang hoạt động → task ở trạng thái running.
    Nếu container lỗi hoặc dừng → task cũng kết thúc.

---

**Task và cơ chế lập lịch (Scheduling)**

1. Task là đơn vị nhỏ nhất trong hệ thống lập lịch Swarm.
Khi bạn khai báo trạng thái mong muốn của service (ví dụ: chạy 3 bản sao HTTP listener), orchestrator sẽ tạo ra 3 task và gán cho các node thích hợp.
Nếu một task bị lỗi (do container crash hoặc fail health check), orchestrator sẽ:
    - Xóa task cũ
    - Tạo task mới để thay thế, đảm bảo đúng trạng thái mong muốn.

> Lưu ý: Một service có thể không thể khởi chạy nếu không có node nào đủ điều kiện chạy task của nó. Khi đó service sẽ ở trạng thái pending.

*Có 2 loại triển khai service, đó là replicated và global.*

>Replicated service: với replicated, bạn chỉ định số lượng bản sao của task mà bạn muốn chạy. Swarm sẽ phân phối nó tới các node trong cụm.

>Global service: bạn không cần chỉ định số lượng replicas, thay vào đó, swarm sẽ tự động chạy 1 task trên mỗi node trong cụm. Khi ta thêm node mới, Swarm sẽ tự tạo task mới trên mỗi node đó. 

#### *Bảo mật trong Docker Swarm bằng PKI (Public Key Infrastructure)*

1. Swarm dùng mutual TLS (Transport Layer Security) để xác thực, mã hóa, và ủy quyền giữa các node (manager ↔ worker).

2. Khi chạy ```docker swarm init```:

    - Manager node được tạo.
    - Docker tự sinh root CA (Certificate Authority) và cặp khóa.
    - Sinh 2 token để thêm node mới:
        - 1 cho worker.
        - 1 cho manager.
    - Token chứa hash của CA + chuỗi bí mật để xác minh tính hợp lệ.

3. Khi node mới join swarm:
    - Nó dùng digest trong token để kiểm tra CA.
    - Manager cấp chứng chỉ TLS mới chứa node ID (danh tính duy nhất) và vai trò (manager/worker).

4. Mỗi node tự động gia hạn chứng chỉ mỗi 3 tháng, có thể chỉnh lại bằng:

    ```docker swarm update --cert-expiry <THỜI_GIAN>```


5. Nếu CA hoặc manager bị lộ, có thể xoay vòng CA bằng:

    ```docker swarm ca --rotate```


- Docker tạo CA mới, ký tạm bằng CA cũ (cross-signed).
- Tất cả node tự động nhận chứng chỉ mới.
- Token join cũ bị vô hiệu hóa.

#### *Trạng thái của Task trong Docker Swarm*
1. Service là mô tả “trạng thái mong muốn” (desired state).
Task là đơn vị thực thi thực tế (do the work).
2. Quy trình thực thi:
    - Dùng lệnh docker service create để tạo service.
    - Yêu cầu được gửi đến manager node.
    - Manager phân bổ (schedule) service cho các node phù hợp.
    - Mỗi service có thể khởi chạy nhiều task.
3. Vòng đời của một task:
    - Task bắt đầu ở trạng thái NEW.
    - Sau đó chuyển dần qua các trạng thái như PENDING, RUNNING, rồi COMPLETE hoặc FAILED.
    - Task chỉ chạy một lần — khi kết thúc, không chạy lại, nhưng một task mới có thể được tạo để thay thế nó.
    - Trạng thái của task chỉ tiến về phía trước, không quay ngược lại (ví dụ: không thể từ COMPLETE → RUNNING).
4. View task state
chạy lệnh ```docker service ps <service-name>``` để xem trạng thái của task.

        docker service ps webserver
        ID             NAME              IMAGE    NODE        DESIRED STATE  CURRENT STATE            ERROR                              PORTS
        owsz0yp6z375   webserver.1       nginx    UbuntuVM    Running        Running 44 seconds ago
        j91iahr8s74p    \_ webserver.1   nginx    UbuntuVM    Shutdown       Failed 50 seconds ago    "No such container: webserver...¦"
        7dyaszg13mw2    \_ webserver.1   nginx    UbuntuVM    Shutdown       Failed 5 hours ago       "No such container: webserver...¦"
#### *Tóm lại*
**Luồng hoạt động của Docker Swarm**

1. Người dùng tạo một stack (bằng docker stack deploy)
→ Stack là tập hợp nhiều service, thường mô tả trong file docker-compose.yml.

2. Mỗi service trong stack định nghĩa:
    - Image container sẽ chạy,
    - Số lượng bản sao (replicas),
    - Các giới hạn tài nguyên, mạng, port, v.v.

3. Manager node nhận yêu cầu và:
    - Phân tích đặc tả service (desired state).
    - Sinh ra tasks, mỗi task tương ứng với một container cần chạy.
    - Dựa trên tình trạng cụm (cluster) → phân phối các task đó đến các node thích hợp (worker hoặc manager đang ở chế độ active).

4. Các node nhận nhiệm vụ (task) từ manager:
    - Tải image tương ứng (nếu chưa có).
    - Khởi chạy container theo cấu hình đã chỉ định.
    - Gửi thông tin trạng thái (health, running, fail, …) về cho manager.

5. Khi có request từ người dùng (client request):
    - Người dùng có thể gửi request tới bất kỳ node nào trong swarm, gọi là Swarm ingress load balancing.
    - Node đó (dù có hoặc không có container của service được yêu cầu) vẫn sẽ:
        - Tự động định tuyến request nội bộ đến node có container phù hợp đang chạy task đó.
:arrow_right: Cơ chế này được Docker đảm bảo bằng routing mesh — hệ thống cân bằng tải và định tuyến nội bộ của Swarm.

6. Cân bằng tải (Load Balancing):
    - Swarm tự động phân phối request giữa các replica container để chia đều tải.
    - Mọi container trong cùng một service đều có cùng DNS name và chia sẻ port công khai.

7. Duy trì “desired state”:
    - Nếu một container chết hoặc node bị mất, manager tự động khởi tạo task mới trên node khác để đảm bảo số lượng replica luôn đúng như yêu cầu.

[Tham khảo thêm tại đây](https://docs.docker.com/engine/swarm/)


## Kafka
### 1. Vấn đề mà Kafka giải quyết
#### *1. Mớ hỗn độn của các Kết nối Trực tiếp*

Các ứng dụng (ví dụ: dịch vụ người dùng, dịch vụ đơn hàng, dịch vụ thanh toán) cần trao đổi dữ liệu với nhau.

Chúng thường kết nối trực tiếp thông qua các API. Khi số lượng ứng dụng tăng lên, mạng lưới kết nối trở nên cực kỳ phức tạp, khó bảo trì và dễ gây lỗi dây chuyền.

Ví dụ: Nếu có 5 ứng dụng cần trao đổi với nhau, bạn cần tới 10 kết nối trực tiếp. Nếu một ứng dụng bị sập, các ứng dụng khác có thể bị ảnh hưởng.

![](https://github.com/supernewbiecoder/KAFKA_IN_DOCKER_SWARM/blob/main/images/problemThatKafkaSolved.png?raw=true)

:arrow_right: Cách kafka giải quyết: Mô hình Pub/Sub (Publish-Subscribe)

- Kafka đóng vai trò là một "trung tâm truyền thông" (central nervous system) hoặc "xương sống dữ liệu" (data backbone).
- Các ứng dụng nguồn (Producers) không cần biết ứng dụng đích là ai, chúng chỉ cần "publish" dữ liệu lên các Topic (chủ đề) trên Kafka.
- Các ứng dụng nhận (Consumers) chỉ cần "subscribe" vào các Topic mà chúng quan tâm để nhận dữ liệu.

![](https://github.com/supernewbiecoder/KAFKA_IN_DOCKER_SWARM/blob/main/images/problemThatKafkaSolved1.png?raw=true)
:arrow_right:Kết quả: Giảm độ phức tạp, tách biệt các hệ thống, dễ dàng mở rộng và bảo trì.


#### *2. Sự khác biệt về Tốc độ xử lý*

- Một ứng dụng có thể tạo ra dữ liệu rất nhanh (ví dụ: clickstream từ người dùng), nhưng ứng dụng xử lý phía sau (ví dụ: hệ thống phân tích) lại xử lý chậm hơn.
- Điều này dẫn đến tắc nghẽn, tràn bộ nhớ, hoặc thậm chí làm sập ứng dụng phía sau.

:arrow_right: Cách kafka giải quyết: Hệ thống đệm tin nhắn (Message Buffer)

- Kafka hoạt động như một bộ đệm có độ bền cao.

- Producers ghi dữ liệu vào Kafka, và Kafka lưu trữ tất cả dữ liệu đó trên đĩa cứng một cách có thứ tự.

- Consumers có thể lấy dữ liệu với tốc độ phù hợp với khả năng xử lý của chúng.

:arrow_right: Kết quả: Kafka hấp thụ các đợt tải dữ liệu lớn, ngăn chặn tình trạng quá tải cho hệ thống phía sau, và cho phép xử lý theo luồng (stream processing).

#### *3. Mất dữ liệu và Thiếu tin cậy*

- Các hệ thống hàng đợi tin nhắn (Message Queues) truyền thống thường xóa tin nhắn ngay sau khi consumer đọc xong.
- Nếu hệ thống xử lý gặp sự cố và cần đọc lại dữ liệu, hoặc nếu có nhiều consumer cần cùng một dữ liệu, thì dữ liệu đã bị mất.

:arrow_right: Cách kafka giải quyết: Khả năng lưu trữ dữ liệu bền vững

- Kafka lưu trữ tất cả dữ liệu trên ổ đĩa và giữ lại dữ liệu trong một khoảng thời gian xác định trước (ví dụ: 7 ngày, 1 tháng, hoặc cho đến khi hết dung lượng).
- Dữ liệu được nhân bản (replicated) trên nhiều máy chủ để đảm bảo không bị mất ngay cả khi một số máy chủ gặp sự cố.
- Consumers có toàn quyền kiểm soát: họ có thể đọc lại dữ liệu từ bất kỳ thời điểm nào (replay data) để xử lý lại hoặc cho mục đích testing.

:arrow_right: Kết quả: Đảm bảo tính toàn vẹn của dữ liệu, cho phép khôi phục sau sự cố và xây dựng các ứng dụng có độ tin cậy cao.

#### *4. Dữ liệu bị cô lập trong các kho dữ liệu (Data Silos)*
- Dữ liệu thường bị mắc kẹt trong các cơ sở dữ liệu và ứng dụng riêng lẻ. Rất khó để có một cái nhìn toàn cảnh, thống nhất về dữ liệu đang di chuyển trong toàn bộ hệ thống.

:arrow_right: Cách kafka giải quyết: 

- Kafka cung cấp một luồng dữ liệu trung tâm, duy nhất (a single, central source of truth) cho tất cả các sự kiện đang xảy ra trong hệ thống.

- Mọi sự kiện quan trọng (như: người dùng đăng ký, đặt hàng, thanh toán, log hệ thống...) đều được ghi vào Kafka.

- Bất kỳ hệ thống nào cần (như Data Warehouse, hệ thống monitoring, hệ thống recommendation, cơ sở dữ liệu tìm kiếm) đều có thể kết nối và tiêu thụ cùng một luồng dữ liệu này.

### 2. Các thành phần trong KAFKA

![](https://github.com/supernewbiecoder/KAFKA_IN_DOCKER_SWARM/blob/main/images/kafkaStructure.png?raw=true)

1. Kafka cluster: Đây là toàn bộ hệ thống Kafka của bạn, bao gồm một hoặc nhiều máy chủ (brokers) hoạt động cùng nhau.

2. Kafka broker: Một broker về cơ bản là một máy chủ Kafka. Nhiều broker kết hợp với nhau tạo thành một Kafka Cluster. Mỗi broker được xác định bằng một ID số nguyên duy nhất.
- Nhiệm vụ:
    - Lắng nghe các yêu cầu từ producers (ghi dữ liệu) và consumers (đọc dữ liệu).
    - Lưu trữ dữ liệu cho các topic.
    - Nhân bản dữ liệu từ các broker khác để đảm bảo tính sẵn sàng cao.

3. Topic: là một "chủ đề" hoặc một "danh mục" mà dữ liệu được publish vào. Bạn có thể hình dung nó giống như một tên bảng trong cơ sở dữ liệu hoặc một folder trong hệ thống file. Mọi message/event/sự kiện đều được ghi vào một topic cụ thể.

4. Partition: Một topic được chia nhỏ thành nhiều partition. Đây là cách Kafka đạt được khả năng mở rộng và xử lý song song.
    - Mỗi partition là một log file có thứ tự, bất biến (append-only log).
    - Các message trong một partition được gán một ID số tăng dần gọi là Offset.
- Tại sao quan trọng?
    - Tính song song: Producers và consumers có thể đọc/ghi song song trên nhiều partition của cùng một topic.
    - Khả năng mở rộng: Dữ liệu của một topic có thể được trải ra trên nhiều broker thông qua các partition.
    - Thứ tự: Thứ tự của message CHỈ được đảm bảo trong một partition, không phải toàn bộ topic.

5. Offset: Là một chỉ mục số (giống như số thứ tự trong danh sách) duy nhất cho mỗi message trong một partition. Một khi message đã được ghi vào partition, offset của nó sẽ không thay đổi.
    - Consumer "commit" (cam kết) offset mà nó đã xử lý xong. Điều này cho phép consumer có thể dừng và bắt đầu lại mà không bị mất message hoặc xử lý trùng lặp.

6. Producer: Ứng dụng gửi message/stream dữ liệu đến các topic của Kafka.
    - Producer có thể chọn partition để ghi dữ liệu (dựa trên key của message hoặc theo vòng round-robin).

7. Consumer: Ứng dụng đọc dữ liệu từ các topic của Kafka.
    - Consumer Group: Các consumer thường hoạt động theo nhóm. Mỗi consumer trong một nhóm sẽ đọc từ một tập partition cụ thể.
        - Quy tắc vàng: Một partition chỉ được đọc bởi một consumer duy nhất trong cùng một consumer group. Điều này đảm bảo việc xử lý theo thứ tự.

        - Nếu số consumer trong nhóm nhiều hơn số partition, những consumer dư thừa sẽ không nhận được message nào.

8. Replication và Leader/Follower
- Mỗi partition được nhân bản trên nhiều broker để chống mất mát dữ liệu.
    - Leader: Một trong các bản sao được chọn làm leader. Tất cả các hoạt động đọc và ghi đều diễn ra với leader.
    - Follower (ISR - In-Sync Replica): Các bản sao còn lại. Chúng sao chép dữ liệu từ leader một cách thụ động. Nếu leader bị lỗi, một follower sẽ được bầu lên làm leader mới.
9. Zookeeper (trong phiên bản Apache kafka mới, còn đc gọi là Kraft Mode, một broker có thể được bầu thành controller, controller này có thể thay thế Zookeeper).
    - Quản lý Cấu hình Cluster (Cluster Membership):
        - Theo dõi tất cả các broker nào đang hoạt động và available trong cluster.

        - Lưu trữ metadata của cluster (ví dụ: có những topic nào, mỗi topic có bao nhiêu partition...).
    - Bầu chọn Leader (Leader Election):
        - Khi leader của một partition bị lỗi, ZooKeeper sẽ điều phối việc bầu chọn một leader mới từ số các follower (ISR). Quá trình này diễn ra rất nhanh, đảm bảo tính sẵn sàng cao.
    - Dịch vụ Đồng bộ hóa và Phối hợp (Coordination Service):
        - Đảm bảo các thao tác cấu hình (như tạo topic, xóa topic) được thực hiện một cách tuần tự và đồng bộ.
        - Giúp các consumer trong cùng một group phối hợp với nhau để biết partition nào được gán cho consumer nào.
    - Lưu trữ Access Control Lists (ACLs):
        - Lưu trữ các quy tắc bảo mật, xác định ai được phép đọc/ghi vào topic nào.

[tham khảo thêm về kafka tại đây](https://viblo.asia/s/apache-kafka-tu-zero-den-one-aGK7jPbA5j2)

## Kafka trong Docker Swarm.
> Sau đây là cách tôi xây dựng cụm kafka trong Docker Swarm

**1. Giới thiệu**
Tôi đã thử nghiệm dựng cụm kafka trên docker swarm dựa trên 2 máy ảo mà mình tạo ra, do cấu hình máy thấp. Trong đó bao gồm 1 worker và 1 manager, tuy nhiên việc mở rộng, thêm các broker vào kafka cluster, hay thêm các máy để thêm vào swarm để quản lý thì việc đấy hoàn toàn dễ dàng.

Tôi cài đặt cấu hình các broker vào trong một file compose.yml để triển khai các dịch vụ. Và tôi tận dụng chế độ Apache Kafka mới (Kraft Mode) để đỡ khỏi congif Zookeeper.

**2. Config**
[Các bạn có thể tham khảo đầy đủ các config tại link này](https://kafka.apache.org/documentation/#brokerconfigs)

> Lưu ý: trong tài liệu tham khảo, với mỗi biến đều phải chuyển thành dạng chữ hoa, dấu ```.``` sẽ được chuyển thành dấu ```_```, dấu ```_``` sẽ được chuyển thành ```__```, dấu ```__``` sẽ được chuyển thành ```___```. và phải có ```KAFKA_``` làm tiền tố.
[xem thêm cái này để biết thêm thông tin](https://github.com/apache/kafka/blob/trunk/docker/examples/README.md) 

Ở đây mình sẽ chỉ liệt kê một số các config quan trọng
- **Thuộc tính buộc phải có **
    1. [KAFKA_PROCESS_ROLE](https://kafka.apache.org/documentation/#brokerconfigs_process.roles): Vai trò mà process này sẽ đóng, có thể là controller, broker, hoặc là cả 2.
    2. [KAFKA_NODE_ID](https://kafka.apache.org/documentation/#brokerconfigs_node.id): Cái này yêu cầu khi bạn muốn ở chế độ Kraft mode. Là id của process, id này phải là duy nhất.
    3. [KAFKA_CONTROLLER_QUORUM_VOTERS](https://kafka.apache.org/documentation/#brokerconfigs_controller.quorum.voters): Bản đồ (map) chứa thông tin id/endpoint cho tập hợp các voter, được viết dưới dạng danh sách các phần tử cách nhau bởi dấu phẩy, mỗi phần tử có cấu trúc: ```{id}@{host}:{port}```
    ví dụ: ```1@localhost:9092,2@localhost:9093,3@localhost:9094```
    4. [ KAFKA_CONTROLLER_LISTENER_NAMES](https://kafka.apache.org/documentation/#brokerconfigs_controller.listener.names): Một danh sách các tên listener được phân tách bằng dấu phẩy, dùng bởi controller.
    Trường này bắt buộc nếu đang chạy trong chế độ KRaft.
    Khi giao tiếp với controller quorum, broker sẽ luôn sử dụng listener đầu tiên trong danh sách này.
- **Một số thuộc tính khác**

    ***Network & Connectivity***
    1. [KAFKA_LISTENERS](https://kafka.apache.org/documentation/#brokerconfigs_listeners): Các endpoint mà Kafka lắng nghe kết nối 
    ```ví dụ: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093```

    2. [KAFKA_ADVERTISED_LISTENERS](https://kafka.apache.org/documentation/#brokerconfigs_advertised.listeners): Các endpoint mà client sử dụng để kết nối đến broker.

    3. [KAFKA_LISTENER_SECURITY_PROTOCOL_MAP](https://kafka.apache.org/documentation/#brokerconfigs_listener.security.protocol.map): Map các listener với security protocol 
    ```ví dụ: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT```

    ***Storage & Data Management***
    1. [KAFKA_LOG_DIRS](https://kafka.apache.org/documentation/#brokerconfigs_log.dirs): Thư mục lưu trữ log data 
    ```ví dụ: /tmp/kafka-logs```
    2. [KAFKA_NUM_PARTITIONS](https://kafka.apache.org/documentation/#brokerconfigs_num.partitions): Số partition mặc định khi tạo topic mới.
    3. [KAFKA_DEFAULT_REPLICATION_FACTOR](https://kafka.apache.org/documentation/#brokerconfigs_default.replication.factor): Replication factor mặc định cho topic.

    ***Performance & Memory***
    1. [KAFKA_MESSAGE_MAX_BYTES](https://kafka.apache.org/documentation/#brokerconfigs_message.max.bytes): Kích thước tối đa của message (tính bằng bytes).
    2. [KAFKA_NUM_NETWORK_THREADS](https://kafka.apache.org/documentation/#brokerconfigs_num.network.threads): Số thread xử lý network requests.
    3. [KAFKA_NUM_IO_THREADS](https://kafka.apache.org/documentation/#brokerconfigs_num.io.threads): Số thread xử lý disk I/O
    ***Security & Authentication***
    1. [KAFKA_AUTO_CREATE_TOPICS_ENABLE](https://kafka.apache.org/documentation/#brokerconfigs_auto.create.topics.enable): Tự động tạo topic khi chưa tồn tại (nên set false cho production).
    2. [KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR](https://kafka.apache.org/documentation/#brokerconfigs_offsets.topic.replication.factor): Replication factor cho internal __consumer_offsets topic.
    ***Docker Specific***
    1. [KAFKA_CLUSTER_ID](https://kafka.apache.org/documentation/#brokerconfigs_cluster.id): ID của Kafka cluster (bắt buộc trong KRaft mode).
    ***Logging & Monitoring***
    1. [KAFKA_LOG_RETENTION_HOURS](https://kafka.apache.org/documentation/#brokerconfigs_log.retention.hours): Thời gian lưu trữ log (hours).
    2. [KAFKA_LOG_RETENTION_BYTES](https://kafka.apache.org/documentation/#brokerconfigs_log.retention.bytes): Kích thước tối đa của log trước khi bị xóa.

dưới đây là code của t về cụm kafka trong docker swarm.
```
version: '3.8'

services:
  kafka-node-1:
    image: apache/kafka:latest
    deploy:
      replicas: 1
      restart_policy:
        condition: any
        delay: 10s
        max_attempts: 3
      placement:
        constraints:
          - node.labels.kafka==true
    environment:
      # Cluster & Node Configuration
      - KAFKA_PROCESS_ROLES=broker,controller
      - KAFKA_NODE_ID=1
      - KAFKA_CLUSTER_ID=ZkQJ7Sl1TJCmt1VFxIqJow
      - KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka-node-1:9093,2@kafka-node-2:9093,3@kafka-node-3:9093
      
      # Network & Listeners - FIXED for Swarm
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka-node-1:9092
      - KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT
      
      # Storage & Data Management
      - KAFKA_LOG_DIRS=/kafka/data
      
      # Cluster Management - SIMPLIFIED
      - KAFKA_AUTO_CREATE_TOPICS_ENABLE=false
      - KAFKA_NUM_PARTITIONS=3
      - KAFKA_DEFAULT_REPLICATION_FACTOR=3
      - KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=3
      - KAFKA_MIN_INSYNC_REPLICAS=2
      
    volumes:
      - kafka_data_1:/kafka/data
    networks:
      - kafka-net

  kafka-node-2:
    image: apache/kafka:latest
    deploy:
      replicas: 1
      restart_policy:
        condition: any
        delay: 10s
        max_attempts: 3
      placement:
        constraints:
          - node.labels.kafka==true
    environment:
      - KAFKA_PROCESS_ROLES=broker,controller
      - KAFKA_NODE_ID=2
      - KAFKA_CLUSTER_ID=ZkQJ7Sl1TJCmt1VFxIqJow
      - KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka-node-1:9093,2@kafka-node-2:9093,3@kafka-node-3:9093
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka-node-2:9092
      - KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT
      - KAFKA_LOG_DIRS=/kafka/data
    volumes:
      - kafka_data_2:/kafka/data
    networks:
      - kafka-net

  kafka-node-3:
    image: apache/kafka:latest
    deploy:
      replicas: 1
      restart_policy:
        condition: any
        delay: 10s
        max_attempts: 3
      placement:
        constraints:
          - node.labels.kafka==true
    environment:
      - KAFKA_PROCESS_ROLES=broker,controller
      - KAFKA_NODE_ID=3
      - KAFKA_CLUSTER_ID=ZkQJ7Sl1TJCmt1VFxIqJow
      - KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka-node-1:9093,2@kafka-node-2:9093,3@kafka-node-3:9093
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka-node-3:9092
      - KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT
      - KAFKA_LOG_DIRS=/kafka/data
    volumes:
      - kafka_data_3:/kafka/data
    networks:
      - kafka-net

volumes:
  kafka_data_1:
    driver: local
  kafka_data_2:
    driver: local  
  kafka_data_3:
    driver: local

networks:
  kafka-net:
    driver: overlay
    attachable: true
```
**Lệnh triển khai**
```
# Deploy stack
docker stack deploy -c compose.yml kafka-cluster

# Kiểm tra trạng thái
docker stack services kafka-cluster

# Kiểm tra logs
docker service logs kafka-cluster_kafka-node-1
```

Các lệnh cơ bản làm việc với kafka:
> Lưu ý: thay ```kafka-cluster_kafka-node-1_1``` thành tên của container chạy trong mỗi node. Lệnh này chạy trên termnal.

**Tạo topic**
```
# Tạo topic cơ bản
docker exec -it kafka-cluster_kafka-node-1_1 kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic test-topic \
  --partitions 3 \
  --replication-factor 3

# Tạo topic với config
docker exec -it kafka-cluster_kafka-node-1_1 kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic orders \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config cleanup.policy=compact
  ```
**Quản lý Topic**
```
# Liệt kê tất cả topics
docker exec -it kafka-cluster_kafka-node-1_1 kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list

# Xem chi tiết topic
docker exec -it kafka-cluster_kafka-node-1_1 kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic test-topic

# Xem tất cả topics chi tiết
docker exec -it kafka-cluster_kafka-node-1_1 kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe

# Xóa topic
docker exec -it kafka-cluster_kafka-node-1_1 kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --delete \
  --topic test-topic

```

**Producer (Gửi tin)**
```
# Producer cơ bản
docker exec -it kafka-cluster_kafka-node-1_1 kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic

# Producer với key
docker exec -it kafka-cluster_kafka-node-1_1 kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --property "parse.key=true" \
  --property "key.separator=:"

# Producer với batch và throughput
docker exec -it kafka-cluster_kafka-node-1_1 kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --property "batch.size=16384" \
  --property "linger.ms=100"

```
**Consumer (Nhận tin)**
```
# Consumer cơ bản (từ đầu)
docker exec -it kafka-cluster_kafka-node-1_1 kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --from-beginning

# Consumer từ offset hiện tại
docker exec -it kafka-cluster_kafka-node-1_1 kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic

# Consumer với group
docker exec -it kafka-cluster_kafka-node-1_1 kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --group my-consumer-group

# Consumer hiển thị key, timestamp
docker exec -it kafka-cluster_kafka-node-1_1 kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --from-beginning \
  --property "print.key=true" \
  --property "print.timestamp=true" \
  --property "key.separator= | " \
  --property "print.partition=true"
```

**Monitoring & Admin**
```
# Xem cluster metadata
docker exec -it kafka-cluster_kafka-node-1_1 kafka-cluster.sh \
  --bootstrap-server localhost:9092 \
  --describe

# Kiểm tra broker config
docker exec -it kafka-cluster_kafka-node-1_1 kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --entity-name 1 \
  --describe

# Kiểm tra topic config
docker exec -it kafka-cluster_kafka-node-1_1 kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name test-topic \
  --describe

```

***Ví dụ thực tế***
```
# 1. Tạo topic
docker exec -it kafka-cluster_kafka-node-1_1 kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic user-events \
  --partitions 3 \
  --replication-factor 3

# 2. Gửi message (mở terminal 1)
docker exec -it kafka-cluster_kafka-node-1_1 kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic user-events

# 3. Nhận message (mở terminal 2)
docker exec -it kafka-cluster_kafka-node-1_1 kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --from-beginning

# 4. Kiểm tra group (terminal 3)
docker exec -it kafka-cluster_kafka-node-1_1 kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --list
```

**Khuyến nghị config volume cho production:**

```
# Sử dụng persistent volume thay vì local
volumes:
  kafka_data_1:
    driver: local-persist
    driver_opts:
      mountpoint: /mnt/kafka/node1
  kafka_data_2:
    driver: local-persist  
    driver_opts:
      mountpoint: /mnt/kafka/node2
  kafka_data_3:
    driver: local-persist
    driver_opts:
      mountpoint: /mnt/kafka/node3
#hoặc sử dụng NFS volume, cloud volume..., nhưng trong phần này mình sẽ ko đề cập đến
```

##