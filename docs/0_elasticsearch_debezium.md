Đồng bộ Database trong Postgres sang ElasticSearch

I. Analyze

Xây dựng một hệ thống để sync data từ Postgres sang ElasticSearch gồm Postgres, Zookeep, kafka, ElasticSearch, Kafka-Connect.

Các vấn đề cần tìm hiểu

1.Kafka, Zookeeper

Hiểu các khái niệm: Producer, consumer, Topic, broker

+ Producers: Một producer có thể là bất kì ứng dụng nào có chức năng publish message vào một topic.

+ Messages: Messages đơn thuần là byte array và developer có thể sử dụng chúng để lưu bất kì object với bất kì format nào - thông thường là String, JSON và Avro.

+ Topics: Một topic là một category hoặc feed name nơi mà record được publish.

+ Partitions: Các topic được chia nhỏ vào các đoạn khác nhau, các đoạn này được gọi là partition.

+ Consumers: Một consumer có thể là bất kì ứng dụng nào có chức năng subscribe vào một topic và nhận các messages

+ Brokers: Kafka cluster là một set các server, mỗi một set này được gọi là 1 broker.

+ Zookeeper: được dùng để quản lý và bố trí các broker.

Xem thêm tại: 

https://data-flair.training/blogs/apache-kafka-tutorial/

Cài đặt Zookeeper server dưới local để hiểu cách hoạt động

Lưu ý start Zookeeper trước khi start Kafka server.

Làm demo theo link: https://kafka.apache.org/quickstart

Elasticsearch

 

Một document của một index trong Elasticsearch sẽ có cấu trúc như hình. Trong đó

_index: tên index. Hệ thống sử dụng data lấy từ postgres nên index được kết hợp theo tên của database và tên table. Xem phần Index trong Elasticsearch bên dưới đặt tên đúng cách.

_type: Các data có liên quan với nhau vì mục đích truy vấn nào đó có thì để cùng type.

_id: id của document. Hiểu như khóa chính. Chọn 1 field trong phần _source để làm id cho document (nên chọn khóa chính của table tương ứng bên postgres). Phần này cần được config trong file-sink-connector

_score: Điểm thể hiện mức độ ưu tiên cho việc tìm kiếm document này.

_source: Chứa phần data cần được index. Đây là data của 1 record trong table tương ứng bên postgres.

 

Index trong ElasticSerach

Lưu ý một số quy ước về cách đặt tên index như sau

+ Tên chỉ chứa các ký tự chữ thường (Không được có chữ Hoa)

+ Không chứa ký tự đặc biệt: / \ * ? < > | , # và khoảng trắng

+ Không bắt đầu bằng ký tự: - + _

+ Giới hạn 255 ký tự

Query:

https://dzone.com/articles/23-useful-elasticsearch-example-queries

Kafka connector

Một famework đứng giữa để kết nối Kafka server với các External Systems như databases, key-value stores, search indexes, file systems.

Tham khảo: https://www.baeldung.com/kafka-connectors-guide

Việc tạo các connector thuần của Kafka khá phức tạp.

=> sử dụng connector có sẵn của Debezium connect giữa Kafka và postgres, connector của Confluent connect giữa kafka và Elasticsearch.

Các loại connector

: connector-source, connector-sink.

+ Debezium trực tiếp hỗ trợ connect tới Postgres server (connector-source).

+ Debezium không trực tiếp hỗ trợ cho ElasticSearch server => sử dụng Confluent-connect-Elasticsearch (connector-sink)

Các thành phần trong các file config cho các connector

+ Có 2 loại file config:

config file-source: Chi tiết xem phần Connector Properties trong link: 

https://debezium.io/docs/connectors/postgresql/

Chú ý: Xem kỹ phần 'topic names', 'Deploying a Connector'

config file-sink: Tham khảo: 

https://docs.confluent.io/current/connect/kafka-connect-elasticsearch/index.html

Transform là phần quan trọng trong cả 2 file. Config regex tốt trong phần này sẽ giảm được số lượng lớn connector

Logical decoding

Debezium’s PostgreSQL connector thực thi một trong các Logical decoding được Debezium hỗ trợ để encode những thay đổi về cả định dạng Protobuf và JSON.

Có 2 loại: decoderbuf và wal2jon. Config vấn đề này trong file postgresql.conf.sample

Xem thêm: link: https://debezium.io/docs/connectors/postgresql/#output-plugin Phần 'Installing the Logical Decoding Output Plug-in'.

II. Chuẩn bị

1.Tham khảo:

Làm theo ví dụ trong link trước để có cái nhìn rõ ràng hơn về những gì sắp bị làm

https://github.com/debezium/debezium-examples/tree/master/unwrap-smt

Tạo các image

Sử dụng các image có sẵn trên docker. Debezium hỗ trợ cả Kafka, zookeeper, connector

+ Zookeeper: debezium/zookeeper:0.8+ Kafka: debezium/kafka:0.8+ Connector: debezium/connect:0.9+ ElasticSearch: docker.elastic.co/elasticsearch/elasticsearch:5.5.2+ Postgres: Debezium cung cấp một docker image (debezium/postgres:9.6) dựa trên PostgreSQL server image và plugin sẵn logical decoding và được khuyên dùng sử dụng. Để cuộc đời trở nên đơn giản dễ dàng hơn thì copy dockerfile của image này trên dockerhub dùng cho postgres trong hệ thống 

Tạo các file config các connector:

Lưu ý:

+ Các table name và các database name nên đặt tên chữ thường bởi vì đánh index trên Elasticserach sẽ dựa vào chúng. Elasticsearch KHÔNG cho phép index CHỮ HOA.Nếu đã lỡ đặt tên chữ Hoa thì cần config file-connector-sink để mapping sang chữ thường => tốn thêm connector chỉ vì cái tên. + Tận dụng sức mạnh của regex và transform để config tốt hơn => giảm số lượng connector

Mỗi database có 2 connector: 1 connector-source + 1 connector-sink.

Đối với auth_service do có table chữ hoa nên cần 2 file sink (Để xử lý chuyện có kí tự viết hoa trong tên)

Event create insert update delete

:

Trong ví dụ ở phần 1 đã bắt được event create, insert, update. Nhưng không config để bắt được event delete nên khi delete data trên postgres thì Elasticsearch vẫn không delete.

=> sử dụng "transforms.unwrap.drop.tombstones" ở file-sinkXem kỹ tại đây: https://debezium.io/docs/configuration/event-flattening/

III. Apply vào hệ thống

Làm tương tự ví dụ ở mục II.1. Lưu ý một số điều sau:

Postgres:

+ Trong docker-file, chuyển context đến dockerfile trong folder app-db. Dockerfile này được coppy từ dockerhub với image: debezium/postgres:9.6.+ Thay đổi file postgresql.conf.sample: tăng số lượng max_replication_slots và max_wal_senders. (Số lượng relication tương ứng với số lượng tối database trong hệ thống)

Connector:

+ Tạo connector-service chứa các file source-connector và file source-connect

+ Tạo các file bash chứa các lệnh start các connector. (start connector-source trước)

Tìm hiểu thêm các API để xem kết quả

Có thể dùng postman để check kết quả bằng các API có sẵn

+ API của Kafka connector: list connector, delete connector.

https://docs.confluent.io/current/connect/references/restapi.html

+ API của Elasticsearch: list index, search

https://www.elastic.co/guide/en/elasticsearch/reference/current/docs.html
