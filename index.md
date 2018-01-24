# SCSt

##SCSt Kafka接入

####mvn依赖：
```
	<dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-stream-dependencies</artifactId>
                <version>${spring-cloud-stream.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
		
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-stream-kafka</artifactId>
        </dependency>
```
#### 生产者配置：
```
spring:
    cloud:
        stream:
	        # 实例数量
            instanceCount: 1
            # 实例索引
            instanceIndex: 0
            # 默认配置
            default:
	            # 生产者默认配置
                producer:
	                # 使用头部
                    headerMode: headers
                    # 开启错误通道
                    errorChannelEnabled: true
                    # 指定分区数量
                    partitionCount: 1
            # kafak配置
            kafka:
	            # binder配置
                binder:
	                # 代理集群地址，多个用逗号分隔
                    brokers: localhost
                    # 代理端口号
                    defaultBrokerPort: 9092
                    # zk集群地址，多个用逗号分隔
                    zkNodes: localhost
                    # zk默认端口号
                    defaultZkPort: 2181
                    # 开启自动创建主题 需要kafka配置为允许自动创建主题
                    autoCreateTopics: true
                    # 开启自动创建分区
                    autoAddPartitions: true
            # 显式指定默认的binder为kafka
            defaultBinder: kafka
            # 绑定配置
            bindings:
	            # 配置名为orgInfoOutput的输出通道
                orgInfoOutput:
		            # 配置绑定要绑定到的kafka topic
                    destination: organization-info
                    # 配置消息的内容类型格式
                    contentType: application/json
                # 配置名为userInfoOutput的输出通道
                userInfoOutput:
                    destination: user-info
                    contentType: application/json
```
#### 消费者配置：
```
spring:
    cloud:
        stream:
            instanceCount: 2
            instanceIndex: 0
            defaultBinder: kafka
            default:
	            # 消费者默认配置 写到这里 避免所有消费者都要写一遍
                consumer:
	                # 要进行分区 分区需要在生产消费两端配置
                    partitioned: true
                    # 使用spring.cloud.stream.instanceCount
                    instanceCount: -1
                    # 使用spring.cloud.stream.instanceIndex
                    instanceIndex: -1
            kafka:
                binder:
                    brokers: localhost
                    defaultBrokerPort: 9092
                    zkNodes: localhost
                    defaultZkPort: 2181
                    autoCreateTopics: true
                    autoAddPartitions: true
                bindings:
	                # 绑定的通道配置
                    orgInfoInput:
	                    # 消费者配置
                        consumer:
	                        # 重新设置偏移量
                            resetOffsets: true
                            # 新加入的消费者 从哪里开始消费
                            startOffset: latest
                            # 关闭均衡负载，才能为每个使用者使用固定的分区
                            autoRebalanceEnabled: false
                    userInfoInput:
                        consumer:
                            resetOffsets: true
                            startOffset: latest
                            autoRebalanceEnabled: false
            # 绑定配置
            bindings:
	            # 配置一个名为orgInfoInput的输入通道
                orgInfoInput:
	                # 这个通道订阅的kafka topic
                    destination: organization-info
                    # 消息格式
                    contentType: application/json
                    # WARNING:组名不能含有-，否则此绑定的属性将不生效
                    # 此通道所属的消费者组
                    group: xxfCarOrgInfoConsumer
                userInfoInput:
                    destination: user-info
                    contentType: application/json
                    # WARNING:组名不能含有-，否则此绑定的属性将不生效
                    group: xxfCarUserInfoConsumer
```
#### 开启SCSt
&emsp;&emsp;在配置类上使用```@EnableBinding```注解即可触发SCSt的下层架构。
```
@EnableBinding({Sink.class})
public class XXApplication{
	...
}
```

&emsp;&emsp;```@EnableBinding```注解接受一个或多个接口作为参数，接口中定义输入或者输出通道.
&emsp;&emsp;以下是SCSt提供的三个接口
###### Sink
&emsp;&emsp;定义了一个名为input的输入通道的接口
###### Source
&emsp;&emsp;定义了一个名为output的输出通道的接口
###### Processor
&emsp;&emsp;定义了一个名为input的输入通道和一个名为output的输出通道的接口
&emsp;&emsp;你也可以像这样定义你自己的接口和通道：
```
public interface OrgInfoProcessor{

    /**
     * 部门信息输入、输出通道名
     */
    String INPUT = "orgInfoInput";
    String OUTPUT = "orgInfoOutput";

    @Input(OrgInfoProcessor.INPUT)
    SubscribableChannel orgInfoInput();

    @Output(OrgInfoProcessor.OUTPUT)
    MessageChannel orgInfoOutput();
}
```
#### 发送消息
##### 直接注入SCSt生成的接口实现bean发送消息：
```
@Service
public class XXService{

	@Autowired
	private OrgInfoProcessor orgInfoProcessor;

	public void sendMessage(Object message){
		orgInfoPocessor.orgInfoOutput().send(MessageBuilder.withPayload(message).build());
	}
}
```
##### 注入通道发送消息：
```
@Service
public class XXService{
	
	@Resource
	private MessageChannel messageChannel;
		
	public void sendMessage(Object message){
		messageChannel.send(MessageBuilder.withPayload(message).build());
	}

}
```
##### 使用```@SendTo```注解发送消息：
```
@Service
public class XXService{
	
	@SendTo(OrgInfoProcessor.OUTPUT)
	public Object send(){
		return new Object();
	}
}
```
#### 接受消息：
##### 使用```@StreamListener```注解接收消息：
```
@Service
public class XXStreamListener {

	@StreamListener(OrgInfoSink.INPUT)
	public void orgInfoInput(OrgInfoDTO orgInfoDTO){
		// handle message
	}
}
```
&emsp;&emsp;```@StreamListener```带有消息类型转换的功能，如果消息有一个值为application/json的contentType头，注解会自动用json消息转换器将受到的消息转换后填充到方法的参数中。
