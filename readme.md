# Nguồn:
https://developer.confluent.io/courses/kafka-connect/intro/?utm_source=youtube&utm_medium=video&utm_campaign=tm.devx_ch.cd-kafka-connect-101_content.connecting-to-apache-kafka

# Bài 1: Introduction to Kafka Connect

![Thu thập dữ liệu từ các hệ thống thượng nguồn (upstream system)](images/1.%20Ingest%20data%20%20from%20upstream%20system.png)

1. Giới thiệu.
    Là 1 component của hệ sinh thái apache kafka, dùng để **streaming data** từ các hệ thống khác vào kafka và từ kafka ra các hệ thống khác ví dụ như CDC.
    Bằng việc sử dụng kafka connect, ta có thể không cần phải viết code producer và consumer và tương tác với các hệ thống bằng tay. Thay vào đó ta chỉ cần upload 1 file json đơn giản lên kafka connect và dành thời gian tập trung code các đoạn code nghiệp vụ khác. 
    Tuy nhiên kafka connect không phải toàn năng, đôi lúc ta muốn customer producer và consumer có khả năng cao siêu hơn thì vẫn phải viết code. 
    Kafka connect có kiến trúc cluster (không có master), các node sử dụng 1 topic trong kafka làm nơi trao đổi dữ liệu. Nhiều node có thể có vai trò consumer và nhiều node có thể có vai trò producer tùy thuộc vào cấu hình file config bạn upload lên cụm kafka. Do đó dùng kafka connect thường cho hiệu suất tốt hơn so với code (khi code họ thường chỉ chạy 1 producer và 1 consumer và ít khi nghĩ đến nhân bản consumer).
    Ngoài việc nhập (ingest) và xuất (egress) dữ liệu, kafka connect còn có thể thực hiện transform nhẹ data.

2. Điểm yếu của code producer va consumer.
    Code CDC có khi không liên quan gì đến logic nghiệp vụ của ứng dụng.
    Phải duy trì mã nguồn, tính toán nhân bản service (scaling out and back).
    Fix Bug, logging, restart lại khi có sự cố như thế nào, ...

3. Với kafka connect bạn không cần lập trình mà chỉ cần viết chuỗi json config => Nhiều người có thể sử dụng kafka hơn không chỉ mỗi dev.

4. Cụm kafka connect nằm riêng biệt với cụm kafka broker/controller. Do đó nó linh hoạt trong khả năng mở rộng, tính toán phân tán, tăng khả năng chịu lỗi, ... mà không phụ thuộc vào bất kỳ thành phần nào khác.

5. Các trường hợp sử dụng:
    CDC data source (real time events stream) đến data sink nào đó phục vụ cho việc phân tích tạo báo cáo, train mô hình ai. Do CDC ảnh hưởng rất ít đến hiệu suất data source, data source không bị quá tải và tập trung vào phục vụ các ứng dụng. Chưa kể đến việc ứng dụng có thể tập trung vào việc code của mình mà không cần quan tâm đến các ứng dụng khác.
    
    ![Streaming pipeline](images/1.%20Stream%20data%20pipelines.png)

    Hình trên là ví dụ, data source là transactional database. Kafka connect có nhiệm vụ CDC các events làm thay đổi dữ liệu trong DB như CRUD của 1 table nào đó trong DB. Các event sẽ được chuyển đổi thành message bắn lên topic trong kafka. Do đó khi CDC thứ đầu tiên ta nghĩ đến là kafka broker + kafka connect.

    Ngoài ra nếu chỉ build kafka cluster nằm giữa source/sink thì hệ thống rất lỏng lẻo (có thể thay đổi source/sink bất cứ lúc nào mà bên kia còn lại không hề hay biết).

6. Cách hoạt động kafka connect

    Bản chất Kafka connect chỉ là các tiến trình JVM và có thể đóng gói lại thành container.
    Để thiết lập 1 cụm kafka connect bàn cần 1 kafka schema registry (nếu bạn muốn sử dụng định dạng Avro). Schema Registry sẽ lưu trữ và quản lý các schema của các bản ghi, giúp Kafka Connect biết cách serialize/deserialize dữ liệu. (Nếu bạn không sử dụng Avro hoặc các định dạng dữ liệu yêu cầu schema, thì Schema Registry không bắt buộc.)

    Standarlone mode: chỉ có 1 node kafka connect. Chỉ được dùng trong các hệ thống nhỏ lẻ và môi trường phát triển
    Distributed mode: dùng trong môi trường production. Mọi node đều có vai trò worker node (không có master). Các worker sẽ sử dụng 1 hoặc nhiều topic của kafka để giao tiếp với nhau (do đó nó không cần zookeeper hay kraft để quản lý câu hình cụm). 
        Khi tạo 1 connector, nó sẽ ghi vào topic tên "config", các worker đọc từ topic này và tự phân chia các task. 
        Các worker cũng sử dụng "status" topic để theo dõi tình trạng của các connector và task đang chạy.

    Thay vì producer/consumer, trong kafka connect chia làm 2 loại task sink connector (Lấy dữ liệu từ bên ngoài đẩy và kafka topic - Như producer) và source connector (lấy dữ liệu từ kafka topic đẩy ra bên ngoài - Như consumer). Bất kỳ worker nào cũng đều có thể đảm nhiệm 1 trong 2 vai trò sinh/source connector.

7. Ví dụ:
    Source connector sử dụng JDBC để lấy CDC data từ mysql.

    ![source connector](images/1.%20source%20connector.png)

    "task.max" chỉ định số lượng task tối đa và sẽ được chia đều cho các worker. Nếu CDC nhiều table thì nên dùng nhiều workers.

    Sink connector, tương tự task.max chỉ định số task có thể chia cho các worker node.

    ![sink connector](images/1.%20sink%20connector.png).

    Nếu chỉ CDC 1 table thì không nên config task.max = 3

# Bài 2 (Hands On) Getting started with Kafka Connect

Mục tiêu: Đẩy dữ liệu từ 1 data source thông qua kafka connect vào kafka topic.

Do không có datasource, bài thực hành này sử dụng 1 connector đặc biệt của kafka connect là Datagen. Nó sẽ làm kafka connect cluster connect đến 1 data source ảo, connector này sẽ gen ra 1 lượng dữ liệu fake.

Connector này chỉ phục vụ cho mục đích học tập.

B1: Tạo topic "orders"

B2: Tạo connector "datagen" và chọn loại dữ liệu mà nó sẽ fake là "orders". Làm theo các bước trên giao diện confluent sẽ được kết quả sau, nếu không có confluent ui thì upload file json sau.

![Datagen json](images/2.%20Datagen%20json.png)

# Bài 3: Running Kafka Connect

Khi cài cụm kafka connect thủ công, ta sẽ phải tự search và cài plugin tương ứng với source/sink về.
    Có những plugins có thể chạy trong Confluent cloud nhưng không chạy được ở kafka connect cluster tự dựng và ngược lại.
    Không phải plugins nào cũng có thể triển khai và có config giống nhau trên mọi cloud providers.
    STMs (single message transform) có thể triển khai với cụm kafka connect tự dựng nhưng không chạy được trên cloud.

Download các connector plugins tại [Confluent Hub](https://www.confluent.io/hub/?utm_medium=video&utm_source=youtube&utm_campaign=tm.devx_ch.cd-kafka-connect-101_content.connecting-to-apache-kafka&session_ref=https://www.youtube.com/&_gl=1*rw9w87*_gcl_aw*R0NMLjE3MjY5OTM5NDEuQ2owS0NRandnTC0zQmhEbkFSSXNBTDZLWjY4ZDFRR1M5b0gwSEhVeDhBbXNKdkJjLUhZZTl0TUxrR3MtZS1reDhFblhteWNYdkZ4VV95Y2FBcjNnRUFMd193Y0I.*_gcl_au*MTA5MjkxMjY4NS4xNzI2OTkzOTQx*_ga*MjcwOTYwNTAzLjE3MjY5OTM5NDE.*_ga_D2D3EGKSGD*MTcyNjk5Mzk0MS4xLjEuMTcyNjk5Njg5Ni42MC4wLjA.)

Nếu dùng Confluent cloud, có 1 thành phần là Managed Connectors và có sẵn các plugins kết nối đến các source/sink phổ biến. Chỉ cần tương tác với giao diện là được.

![Confluent Cloud Managed Connectors](images/3.%20Confluent%20Cloud%20Managed%20Connectors.png).

Nếu tự dựng kafka connect thì hiểu nhiên phải tự cấu hình,

![Self-Managed Connect Cluster](images/3.%20Self-Managed%20Kafka%20Connect.png)

# Bài 4: Connectors, Configuration, Converters and Transforms

Kiến trúc bên trong 1 worker kafka connect.

![Inside Kafka Connect](images/4.%20Inside%20Kafka%20Connect.png)

1. Connector instance: chịu trách nhiệm kết nối tương tác giữa kafka connect và các hệ thống bên ngoài (key-comppnent). Được tạo thành khi bạn cài các connectors plugins thích hợp.
    Là 1 logical jobs xử lý data được viết bởi cộng đồng, các nhà cung cấp hoặc người dùng tự viết và có thể tái sử dụng. Sau khi tải plugins trên mỗi worker, người dùng phải setup 1 số config cơ bản để tạo thành connectors instance. Hiêu đơn giản nó là 1 interface để kafka connect có thể đọc ghi dữ liệu.
    C1: Sử dụng REST API để upload chuỗi json config lên các workers hoặc dùng giao diện.

    ![Elasticsearch sink connector rest api](images/4.%20Elasticsearch%20sink%20connector%20rest%20api.png)

    "connector.class": Mỗi 1 plugin sẽ chỉ định 1 class riêng, đọc kỹ doc của họ.
    "topics": nơi mà kafka connect lấy dữ liệu đẩy vào sink.
    "connection.url": địa chỉ để kafka connect kết nối đến sink.
    Mỗi 1 plugins có 1 tập các properties khác nhau, đọc kỹ doc của họ.

    **C2: Sử dụng KsqlDB để quản lý connector: Thay vì sử dụng Rest API để tương tác trực tiếp với Kafka connect, ta có thể sử dụng KsqlDB để tạo, quản lý, xóa các Kafka Connectors bằng câu lệnh sql. Ví dụ**

    ![Elasticsearch sink connector ksql](images/4.%20Elasticseach%20sink%20connector%20ksql.png)

    C3: Sử dụng giao diện để upload config file.

2. Converters: Chịu trách nhiệm serialization/deserialization data.
    Có rất nhiều converters khác nhau có sẵn có thể kể đến như:
        Avro – io.confluent.connect.avro.AvroConverter (khi dùng avro bắt buộc phải có Schema Registry)
        Protobuf – io.confluent.connect.protobuf.ProtobufConverter (khi dùng định dạng Protobuf bắt buộc phải có schema registry)
        String – org.apache.kafka.connect.storage.StringConverter (dùng khi dữ liệu là 1 chuỗi văn bản, hiểu nhiên không có schema)
        JSON – org.apache.kafka.connect.json.JsonConverter (dùng khi dữ liệu có dạng json, không nhất thiết phải có schema chung cho tất cả các message trong 1 topic nhưng nếu tất cả các message đều có chung 1 schema json thì nên tạo 1 schema trong Schema Registry)
        JSON Schema – io.confluent.connect.json.JsonSchemaConverter
        ByteArray – org.apache.kafka.connect.converters.ByteArrayConverter
    Mỗi message được gửi đến kafka luôn có dạng Key-Value và ta có thể config sao cho mỗi cái chuyển đổi theo 1 converters khác nhau.

    ![Key-Value Converter](images/4.%20Key-Value%20converter.png)

    ![Key-Value Converter schema](images/4.%20Key-Value%20converter%20schema.png)

    Kafka chỉ lưu dữ liệu dưới dạng byte. Để serialization/deserialization dữ liệu, cần phải lưu schema của dữ liệu. Đó là lý do ta cần Schema Registry. Nhờ có schema registry, ta không cần phải lưu schema ở mỗi message khi nó đổ vào kafka. Thay vào đó các message sẽ chứa các key trỏ đến schema của nó trong Schema Registry. Ngoài ra schema registry còn giúp ta quản lý, đảm bảo tất cả các message trong 1 topic có cùng 1 schema.

    ![Schem Registry](images/4.%20Schema%20Registry.png)
        
3. Transformations: có thể transform nhẹ dữ liệu trước khi lưu vào kafka topic hay đẩy vào sinks. Đây là thành phần không bắt buộc trong kafka connect có thể config vào nếu thích. Transforms này được thực hiện trên từng message trước khi data được lưu vào kafka/sink/source do đó gọi nó là Single Message Transform.

    ![SMT](images/4.%20SMT.png)

    Thường dùng SMT trong 1 số trường hợp như drop 1 field trước khi lưu dữ liệu vào, thêm metadata, rename field, đổi data type field, ...
    Đối với các transform phức tạp như join, aggregations với các topic khác, ... thì nên dùng ksqldb hoặc kafka stream.

# Bài 5: (Hands On) Use SMTs with a Managed Connector

Mục tiêu: Tiếp tục sử dụng Datagen connector ở bài 2. Nhưng trước khi dữ liệu được đưa vào kafka topic, thực hiện transforms: orderid sẽ được lưu dưới dạng string, orderunits được lưu dưới dạng int32 (2 SMTs)

Sử dụng giao diện confluent cloud: Nhấn vào "Show advanced configuration" -> "Add a single message transforms" -> Đặt tên SMT là "castValues" cho dễ phân biệt.

![Fitst SMT Confluent](images/5.%20First%20SMT%20Confluent.png)

![Second SMT Confluent](images/5.%20Second%20SMT%20Confluent.png)

Cách khác là sử dụng Json file

![Json SMT](images/5.%20Json%20SMT.png)

# Bài 6: (Hand On) Confluent Cloud Managed Connector API

Mục tiêu: Giới thiệu 1 số API của Confluent Cloud để bạn có thể gọi đến nó. Bằng cách naỳ có thể tích hợp các project với các sản phẩm của confluent.

Tham khảo nếu dùng confluent cloud [đây](https://developer.confluent.io/courses/kafka-connect/connect-api-hands-on/).

# Bài 7: (Hands On) Confluent Cloud Managed Connector CLI

Mục tiêu: Giới thiệu các lệnh CLI để tương tác với Confluent managed conectors

Tham khảo nến dùng confluent cloud [đây](https://developer.confluent.io/courses/kafka-connect/confluent-cli-hands-on/)

# Bài 8: Deploying Kafka Connect

Mục tiêu: hiểu về distributed và standarlone mode (architech) của kafka connect. Cách 1 connector hoạt động, tạo 1 connector instance trên kafka connect.

![Deploying Kafka Connect](images/8.%20Deploying%20Kafka%20Connect.png)

Connector instance về mặt vật lý là 1 thread hay là 1 task. Ở hình trên ta có thể thấy 2 logic connectors, mỗi cái được coi là 1 task.

Task là đơn vị nhỏ nhất để có thể nhân bản, chạy song song, mở rộng. Hình dưới đây cho thấy task JDBC được nhân bản lên làm 2.

![Task Parallelism](images/8%20.Task%20Parallelism.png)

Các tasks sẽ được run trong các kafka connect worker. Kafka connect là 1 tiến trình daemon JVM tên là worker. Tiến trình này sẽ tạo ra các luồng, mỗi luồng có thể chạy 1 task.

![Worker daemon](images/8.%20Worker%20daemon.png)

1 node Kafka có thể ở 2 chế độ standarlone (1 node) hoặc distributed mode. Ở chế độ distributed, các worker sử dụng topic của kafka để lưu cấu hình state pertaining to connector configuration, connector status, and more. The topics are configured to retain this information indefinitely, known as compacted topics. Connector instances are created and managed via the REST API that Kafka Connect offers.

Do các thông tin đã được lưu trong kafka topic nên khi 1 worker mới được thêm vào, task sẽ tự động được cần bằng tải giữa các worker.

![rebalanced task](images/8.%20Rebalance%20Task.png)

Lưu ý: có thể cấu hình nhiều cluster kafka connect tương tự nhiều kafka cluster.

![Multi cluster](images/8.%20Multi%20cluster.png)

Standarlone mode: Khác với ở distributed mode. Kafka connect worker sẽ lưu cấu hình vào file của nó chứ không qua topic kafka. Connector được tạo ra bằng local files của nó không phải thông qua Rest API.

![Standarlone mode](images/8.%20Standarlone%20mode.png)

Kafka Connect ở chế độ distributed mode hay standarlone phụ thuộc vào giá trị CONNECT_GROUP_ID

# Bài 9: Running Kafka Connect in Docker

Confluent public image cp-kafka-connect để có thể dựng cụm kafka connect với 1 vài tác vụ như có sẵn 1 số connector cơ bản, thêm file jar sủa sink/source connector, SMT, ...

Có 2 cách thêm file jar plugin connector là thêm vào lúc tạo images hoặc lúc run container rồi thêm vào. Hiển nhiên cách thêm vào lúc tạo image thích hơn, sử dụng trường CONNECT_PLUGIN_PATH để chỉ định file jar (trong môi trường production sẽ còn nhiều trường hơn thế này)
C1:
```sh
kafka-connect:
  image: confluentinc/cp-kafka-connect:7.1.0-1-ubi8
  environment:
    CONNECT_PLUGIN_PATH: /usr/share/java,/usr/share/confluent-hub-components

  command:
    - bash
    - -c
    - |
      confluent-hub install --no-prompt neo4j/kafka-connect-neo4j:2.0.2
      /etc/confluent/docker/run
```
C2: Build image mới
```sh
FROM confluentinc/cp-kafka-connect:7.1.0-1-ubi8

ENV CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"

RUN confluent-hub install --no-prompt neo4j/kafka-connect-neo4j:2.0.2
```

Tuy nhiên đa số th là thêm connector instance trong khi container đã được run. Chả ai dại mà shutdown kafka connect trong môi trường production cả.
```sh
# Launch Kafka Connect
/etc/confluent/docker/run &
#
# Wait for Kafka Connect listener
echo "Waiting for Kafka Connect to start listening on localhost ⏳"
while : ; do
  curl_status=$$(curl -s -o /dev/null -w %{http_code} http://localhost:8083/connectors)
  echo -e $$(date) " Kafka Connect listener HTTP state: " $$curl_status " (waiting for 200)"
  if [ $$curl_status -eq 200 ] ; then
    break
  fi
  sleep 5 
done

echo -e "\n--\n+> Creating Data Generator source"
curl -s -X PUT -H  "Content-Type:application/json" http://localhost:8083/connectors/source-datagen-01/config \
    -d '{
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "kafka.topic": "ratings",
    "max.interval":750,
    "quickstart": "ratings",
    "tasks.max": 1
}'
sleep infinity
```

# Bài 10: (Hands On) Run a Self-Managed Connector in Docker

Đọc file docker-compose cụm kafka connect tại [đây](https://github.com/confluentinc/learn-kafka-courses/blob/main/kafka-connect-101/self-managed-connect.yml) và hướng dẫn hiểu file từ mục "[Running a Self-Managed Connector with Confluent Cloud exercise steps"](https://developer.confluent.io/courses/.kafka-connect/self-managed-connect-hands-on/) trở đi.

Mục tiêu: Bài này thực hiện CDC bằng debezium-mysql connector từ Mysql DB vào kafka cluster. Sau đó sử dụng elasticsearch và neo4j sink connector để đẩy dữ liệu từ kafka topic vào các sink này.

Ý nghĩa các config:

1. Mỗi kafka connect worker khai báo cụm kafka mà nó sẽ giao tiếp và group id của cluster mà nó thuộc về.

    ![Kafka Broker Env](images/10.%20Kafka%20Broker%20Env.png)

    Các kafka connect cùng 1 group id sẽ cùng 1 cluster và ở chế độ distributed mode.
    Tên các topic mặc định mà kafka connect cluster dùng để giao tiếp nội bộ với nhau.

2. Cấu hình key-value converter mà worker này sẽ sử dụng

    ![Converter Env](images/10.%20Converter%20Env.png)

3. Load các connector plugins

    ![Connector Plugin](images/10.%20Connector%20Plugin.png)

4. Chạy lệnh sau để chắc chắn cụm kafka connect đã sẵn sàng. 8083 là cổng của worker 1, do đó khi tương tác với cụm kafka connect, ta chỉ cần quan tâm 1 worker.

    ```sh
    bash -c ' \
    echo -e "\n\n=============\nWaiting for Kafka Connect to start listening on localhost ⏳\n=============\n"
    while [ $(curl -s -o /dev/null -w %{http_code} http://localhost:8083/connectors) -ne 200 ] ; do
    echo -e "\t" $(date) " Kafka Connect listener HTTP state: " $(curl -s -o /dev/null -w %{http_code} http://localhost:8083/connectors) " (waiting for 200)"
    sleep 15
    done
    echo -e $(date) "\n\n--------------\n\o/ Kafka Connect is ready! Listener HTTP state: " $(curl -s -o /dev/null -w %{http_code} http://localhost:8083/connectors) "\n--------------\n"
    '
    ```

5. Xác nhận các connector plugin đã cài đặt có những plugin cần thiết

    ```sh
    curl -s localhost:8083/connector-plugins | jq '.[].class'|egrep 'Neo4jSinkConnector|MySqlConnector|ElasticsearchSinkConnector'
    # Đầu ra mong đợi sẽ như sau
    "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector"
    "io.debezium.connector.mysql.MySqlConnector"
    "streams.kafka.connect.sink.Neo4jSinkConnector"
    ```

6. Run source connector instance là Stream CDC mysql đến kafka topic

    ```sh
    curl -i -X PUT -H  "Content-Type:application/json" \
        http://localhost:8083/connectors/source-debezium-orders-01/config \
        -d '{
                "connector.class": "io.debezium.connector.mysql.MySqlConnector",
                "value.converter": "io.confluent.connect.json.JsonSchemaConverter",
                "value.converter.schemas.enable": "true",
                "value.converter.schema.registry.url": "'$SCHEMA_REGISTRY_URL'",
                "value.converter.basic.auth.credentials.source": "'$BASIC_AUTH_CREDENTIALS_SOURCE'",
                "value.converter.basic.auth.user.info": "'$SCHEMA_REGISTRY_BASIC_AUTH_USER_INFO'",
                "database.hostname": "mysql",
                "database.port": "3306",
                "database.user": "kc101user",
                "database.password": "kc101pw",
                "database.server.id": "42",
                "database.server.name": "asgard",
                "table.whitelist": "demo.orders",
                "database.history.kafka.bootstrap.servers": "'$BOOTSTRAP_SERVERS'",
                "database.history.consumer.security.protocol": "SASL_SSL",
                "database.history.consumer.sasl.mechanism": "PLAIN",
                "database.history.consumer.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"'$CLOUD_KEY'\" password=\"'$CLOUD_SECRET'\";",
                "database.history.producer.security.protocol": "SASL_SSL",
                "database.history.producer.sasl.mechanism": "PLAIN",
                "database.history.producer.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"'$CLOUD_KEY'\" password=\"'$CLOUD_SECRET'\";",
                "database.history.kafka.topic": "dbhistory.demo",
                "topic.creation.default.replication.factor": "3",
                "topic.creation.default.partitions": "3",
                "decimal.handling.mode": "double",
                "include.schema.changes": "true",
                "transforms": "unwrap,addTopicPrefix",
                "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
                "transforms.addTopicPrefix.type":"org.apache.kafka.connect.transforms.RegexRouter",
                "transforms.addTopicPrefix.regex":"(.*)",
                "transforms.addTopicPrefix.replacement":"mysql-debezium-$1"
        }'
    ```

    Để kiểm tra connector instance vừa tạo chạy lệnh
    ```sh
    curl -s "http://localhost:8083/connectors?expand=info&expand=status" | \
        jq '. | to_entries[] | [ .value.info.type, .key, .value.status.connector.state,.value.status.tasks[].state,.value.info.config."connector.class"]|join(":|:")' | \
        column -s : -t| sed 's/\"//g'| sort
    # Đầu ra mong đợi sẽ như sau
    source  |  source-debezium-orders-01  |  RUNNING  |  RUNNING  |  io.debezium.connector.mysql.MySqlConnector
    ```

7. Stream data từ kafka topic đến elasticsearch
    ```sh
    curl -i -X PUT -H  "Content-Type:application/json" \
        http://localhost:8083/connectors/sink-elastic-orders-01/config \
        -d '{
                "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
                "value.converter": "io.confluent.connect.json.JsonSchemaConverter",
                "value.converter.schemas.enable": "true",
                "value.converter.schema.registry.url": "'$SCHEMA_REGISTRY_URL'",
                "value.converter.basic.auth.credentials.source": "'$BASIC_AUTH_CREDENTIALS_SOURCE'",
                "value.converter.basic.auth.user.info": "'$SCHEMA_REGISTRY_BASIC_AUTH_USER_INFO'",
                "topics": "mysql-debezium-asgard.demo.ORDERS",
                "connection.url": "http://elasticsearch:9200",
                "key.ignore": "true",
                "schema.ignore": "true",
                "tasks.max": "2"
        }'
    ```
    Kiếm tra connector instance vừa chạy
    ```sh
    curl -s "http://localhost:8083/connectors?expand=info&expand=status" | \
        jq '. | to_entries[] | [ .value.info.type, .key, .value.status.connector.state,.value.status.tasks[].state,.value.info.config."connector.class"]|join(":|:")' | \
        column -s : -t| sed 's/\"//g'| sort | grep ElasticsearchSinkConnector
    # Đầu ra mong đợi
    source  |  source-debezium-orders-01  |  RUNNING  |  RUNNING  | RUNNING  |  io.confluent.connect.elasticsearch.ElasticsearchSinkConnector
    ```

8. Stream data từ kafka topic đến Neo4j (tương tự)
    ```sh
    curl -i -X PUT -H  "Content-Type:application/json" \
        http://localhost:8083/connectors/sink-neo4j-orders-01/config \
        -d '{
                "connector.class": "streams.kafka.connect.sink.Neo4jSinkConnector",
            "value.converter": "io.confluent.connect.json.JsonSchemaConverter",
                "value.converter.schemas.enable": "true",
                "value.converter.schema.registry.url": "'$SCHEMA_REGISTRY_URL'",
                "value.converter.basic.auth.credentials.source": "'$BASIC_AUTH_CREDENTIALS_SOURCE'",
                "value.converter.basic.auth.user.info": "'$SCHEMA_REGISTRY_BASIC_AUTH_USER_INFO'",
                "topics": "mysql-debezium-asgard.demo.ORDERS",
                "tasks.max": "2",
                "neo4j.server.uri": "bolt://neo4j:7687",
                "neo4j.authentication.basic.username": "neo4j",
                "neo4j.authentication.basic.password": "connect",
                "neo4j.topic.cypher.mysql-debezium-asgard.demo.ORDERS": "MERGE (city:city{city: event.delivery_city}) MERGE (customer:customer{id: event.customer_id, delivery_address: event.delivery_address, delivery_city: event.delivery_city, delivery_company: event.delivery_company}) MERGE (vehicle:vehicle{make: event.make, model:event.model}) MERGE (city)<-[:LIVES_IN]-(customer)-[:BOUGHT{order_total_usd:event.order_total_usd,order_id:event.id}]->(vehicle)"
            } '
    ```
    Kiểm  tra connector instance vừa chạy
    ```sh
    curl -s "http://localhost:8083/connectors?expand=info&expand=status" | \
        jq '. | to_entries[] | [ .value.info.type, .key, .value.status.connector.state,.value.status.tasks[].state,.value.info.config."connector.class"]|join(":|:")' | \
        column -s : -t| sed 's/\"//g'| sort | grep Neo4jSinkConnector
    # Đầu ra mong đợi
    source  |  source-debezium-orders-01  |  RUNNING  |  RUNNING  | RUNNING  |  io.confluent.connect.elasticsearch.Neo4jSinkConnector
    ```

# Bài 11: Kafka Connect’s REST API

Ở bài thực hành 10, bản chất chúng ta đã tương tác với REST API của cụm kafka connector bằng lệnh curl đến http://localhost:8083 (là địa chỉ của kafka connect 1).
Kafka Connect được thiết kế để chạy như 1 service nên nó cũng hỗ trợ REST API để quản lý các connector instance. Bạn có thể thực hiện request lên bất cứ worker nào thuộc cụm, request sẽ tự động được chuyển tiếp nếu cần.
Dưới đây là 1 số API cơ bản

1. Lấy thông tin của cụm kafka connect: các worker, commit, kafka cluster id

    ```sh
    curl http://localhost:8083/
    ```

2. Kiểm tra các plugins đã cài vào trong cụm.

    ```sh
    curl -s localhost:8083/connector-plugins
    ```

    Có thể dùng jq để kiểm soát đầu ra dễ đọc hơn

    ```sh
    curl -s localhost:8083/connector-plugins | jq '.'
    ```

3. Tạo 1 connector instance: Để tạo 1 connector instance, ta thực hiện PUT/POST 1 JSON file để config connector vào api sau.

    ![Create a connector instance](images/11.%20Create%20a%20connector%20instance.png)

4. List tất cả các connectors

    ```sh
        curl -s -X GET "http://localhost:8083/connectors/"
    ```

5. Xem thông tin config và status của connector instance. (inspect) Dùng GET api để lấy dữ liệu.

    ```sh
        # sink-elastic-orders-00 là tên của connector instance.
        curl -i -X GET -H  "Content-Type:application/json" \
        http://localhost:8083/connectors/sink-elastic-orders-00/config

        # Sử dụng tiến trình jq để dữ liệu dễ nhìn hơn
        curl -s "http://localhost:8083/connectors?expand=info&expand=status" | \
        jq '. | to_entries[] | [ .value.info.type, .key, .value.status.connector.state,.value.status.tasks[].state, .value.info.config."connector.class"] |join(":|:")' | \
        column -s : -t| sed 's/\"//g'| sort
        
        # Ý nghĩa câu lệnh get
        ![](images/11.%20Ý%20nghĩa%20câu%20lệnh%20get.png)
    ```

6. Xóa connector instance,

    ```sh
        curl -s -X DELETE "http://localhost:8083/connectors/sink-elastic-orders-00"
    ```

7. Inspect các task của connector instance.

    ```sh
        curl -s -X GET "http://localhost:8083/connectors/source-debezium-orders-00/status" | jq '.'
        # Các task của connector được xếp trong 1 stack. Nếu như connector bị lỗi, cần check trang thái của các task để tìm ra lỗi.
        curl -s -X GET "http://localhost:8083/connectors/source-debezium-orders-00/tasks/0/status" | jq '.'
    ```

8. Restart connector và tasks

    ```sh
        # restart cả connector
        curl -s -X POST "http://localhost:8083/connectors/source-debezium-orders-00/restart"
        # restart 1 task trong connector
        curl -s -X POST "http://localhost:8083/connectors/source-debezium-orders-00/tasks/0/restart"
    ```
9. Pause, resume a connector

    ```sh
        # Bản chất các connector bao gồm các task, mỗi task chạy trên 1 luồng nên không có gì chắc chắn khi thực hiện pause connector thì các task pause cùng 1 lúc.
        curl -s -X PUT "http://localhost:8083/connectors/source-debezium-orders-00/pause"
        # Resume:
        curl -s -X PUT "http://localhost:8083/connectors/source-debezium-orders-00/resume"
    ```

10. List các task của 1 connector
    ```sh
    curl -s -X GET "http://localhost:8083/connectors/sink-neo4j-orders-00/tasks" | jq '.'
    ```

# Bài 12: Monitoring Kafka Connect

Nếu sử dụng confluent cloud, có thể sử dụng các công nghệ của confluent như: Confluent Cloud Console và Confluent Platform Control Centrer để monitoring các connector instance. Ngoài ra cũng có thể sử dụng Confluent Metrics API để thu thập các metrics và tích hợp nó vào các công cụ mornitoring khác như DataDog, Dynatrace, Grafana và Prometheus.

Cách khác là sử dụng dữ liệu được kafka connect cung cấp trực tiếp như JMX và REST
    JMX (java management extensions) là 1 chuẩn của java để quản lý và giám sát các ứng dụng. Nó cho phspe bạn theo dõi và quản lý các ứng dụng Java bằng cách cung cấp thông tin dưới dạng các MBean (management beans) đại diện cho các tài nguyên quản lý như cấu hình, thống kê và trạng thái của ứng dụng.
    Trong kafka connect, JMX cung cấp các thông tin như số lượng task, các tiến trình, trạng thái của connector và các chỉ số hiệu suất khác.

# Bài 13: Errors and Dead Letter Queues

Kafka connect support 1 vài patterns để errors handling: fail fast, silently ignore và dead letter queues. Bài này xem xét khả năng sử dụng 3 pattern này để handle error.
Kịch bản 1: 1 topic chứa các message được mã hóa dưới dạng json. Nhưng khi cấu hình, ta cấu hình nhầm value converter thành avro nên không thể deserializtion và gặp exception.

![kịch bản 1](images/13.%20Kịch%20bản%201.png)

Kịch bản 2: 1 topic chứa nhiều message được serialization bởi các converter khác nhau.

![Kịch bản 2](images/13.%20Kịch%20bản%202.png)

Bạn có thể giải quyết 2 kịch bản này bằng việc sử dụng error tolerances và dead letter queues.

1. Error Tolerances - Fail Fast (default): Theo mặc định nếu connector không thể deserialization, task sẽ stop và bạn phải tìm lỗi bật lại bằng tay.

![Fail Fast](images/13.%20Fail%20Fast.png)

2. Dead Letter Queues: Là 1 kafka topic khác, các message mà gặp error sẽ được ném vào đó. Và phải bật chức năng này bằng tay vì Fail Fast là default.

![Dead Letter Queues](images/13.%20Dead%20Leatter%20Queue.png)

![Solve](images/13.%20Solve.png)

# Bài 14: Troubleshooting Confluent Managed Connectors

Dùng cloud thì quan tâm, next sang bài 15.

# Bài 15: Troubleshooting Self-Managed Kafka Connect

Kafka Connect là 1 framework tích hợp dữ liệu, lỗi thường gặp phải là connect đến các external system ở chế độ security mode (ví dụ secret key, username, password, table name, ...)

Kịch bản: Connect worker vẫn running, source connector vẫn running nhưng không có data nào được insert vào.

![Scenario 1](images/15.%20Scenario%201.png)

Giải pháp xem các task trong connector instance xem có task nào bị lỗi không bằng cách sử dụng Rest API của kafka connect.

![Solve 1](images/15.%20Solve%201.png)

Nếu có task lỗi, đọc stack trace của task và nhìn exception.

![Exception](images/15.%20Exception.png)

