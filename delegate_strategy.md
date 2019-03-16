## 委派模式和策略模式作业

### 举例Spring源码中你见过的委派模式，并画出类关系图

* 以spring-mvc的DispatcherServlet为例，spring-mvc将所有请求分发给不同的handler处理



### 利用策略模式重构一段业务代码

*  重构前订阅配置：

```
/**
 *配置用户订阅地址
 * 
 * @param accessor
 */
private void configSubDes(StompHeaderAccessor accessor) {
	// 获取订阅路径
	String subDest = accessor.getDestination();
	// 校验订阅目的地
	String subPrefix = verifySubDest(subDest);
	// 排除不需要拦截的订阅前缀
	if(excludePrefixSet.contains(subPrefix)) {
		return;
	}
	// 获取用户标识
	String userId = accessor.getUser().getName();
	// 获取订阅后缀
	String subSuffix = subDest.substring(subPrefix.length());
	// 定义交换机、队列以及绑定
	String exchangeName, queueName, routingKey;
	Exchange exchange = null;
	Queue queue = null;
	Binding binding = null;
	// 每种订阅前缀需要配置不同的交换机、队列以及路由规则

	// 如果是广播消息订阅前缀
	if(TOPIC_SUB_PREFIX.equals(subPrefix)){
		queueName = "topic" + "-" + subPrefix.substring(1, subPrefix.length() - 1) + "-" + userId;
		routingKey = subSuffix;
		exchangeName = TOPIC_EXCHANGE;
		// 定义队列为非持久化队列	
		queue = new Queue(queueName, false);	
		// 定义交换机为topic类型
		exchange = new TopicExchange(exchangeName);
		// 创建绑定 -> 建立消息路由规则
		binding = BindingBuilder.bind(queue).to(exchange).with(routingKey).build();

	// 如果是运营消息订阅前缀
	}else if(OPER_SUB_PREFIX.equals(subPrefix)){
		queueName = "oper" + "-" + subPrefix.substring(1, subPrefix.length() - 1) + "-" + userId;
		routingKey = subSuffix;
		exchangeName = OPER_EXCHANGE;
		// 定义队列为持久化队列	
		queue = new Queue(queueName, true);	
		// 定义交换机为topic类型
		exchange = new TopicExchange(exchangeName);
		// 创建绑定 -> 建立消息路由规则
		binding = BindingBuilder.bind(queue).to(exchange).with(routingKey).build();

	// 如果是用户消息订阅前缀
	}else if(USER_SUB_PREFIX.equals(subPrefix)){
		queueName = "user" + "-" + subPrefix.substring(1, subPrefix.length() - 1) + "-" + userId;
		routingKey = subSuffix + "$" + userId;
		exchangeName = USER_EXCHANGE;
		// 定义队列为持久化队列	
		queue = new Queue(queueName, true);	
		// 定义交换机为direct类型
		exchange = new DirectExchange(exchangeName);
		// 创建绑定 -> 建立消息路由规则
		binding = BindingBuilder.bind(queue).to(exchange).with(routingKey).build();

	}else{
		// 如果订阅前缀非法 直接抛出异常
		throw new RuntimeException("Illegal subscription path '" + subDest + "'");
	}

	// ...
}
```

* 使用策略模式重构后订阅配置：

```
/**
 *配置用户订阅地址
 * 
 * @param accessor
 */
private void configSubDes(StompHeaderAccessor accessor) {
	// 获取订阅路径
	String subDest = accessor.getDestination();
	// 校验订阅目的地
	String subPrefix = verifySubDest(subDest);
	// 排除不需要拦截的订阅前缀
	if(excludePrefixSet.contains(subPrefix)) {
		return;
	}
	// 获取用户标识
	String userId = accessor.getUser().getName();
	// 获取订阅后缀
	String subSuffix = subDest.substring(subPrefix.length());

	// 根据订阅前缀获取订阅处理器
	SubscribeHandler handler = SubscribeHandlerHolder.getHandler(subPrefix);
	// 如果处理器不存在则抛出异常
	if(handler == null){
		throw new RuntimeException("Illegal subscription path '" + subDest + "'");
	}
	// 执行配置并初始化
	handler.config(subSuffix, userId);

	// ...
}
```

* 相关配置类

```
public abstract class SubscribeHandler {
	
	protected Exchange exchange;
	
	protected Queue queue;
	
	protected Binding binding;

	protected String subPrefix;
	
	
	/**
	 * 子类具体实现：根据用户ID和订阅后缀初始化队列以及绑定规则
	 */
	public abstract void config(String subSuffix, String userId);

}
```


```
public class SubscribeHandlerHolder {
	
	private static final Map<String, SubscribeHandler> HANDLER_MAP = new ConcurrentHashMap<>();
	
	static {
		HANDLER_MAP.put("/topic/", new TopicSubscribeHandler("amq.topic", "/topic/"));
		HANDLER_MAP.put("/oper/", new OperSubscribeHandler("amq.topic", "/oper/"));
		HANDLER_MAP.put("/user/", new UserSubscribeHandler("amq.direct", "/user/"));
	}
	
	private SubscribeHandlerHolder() {
		throw new RuntimeException("This class cannot be initialized");
	}
	
	/**
	 * 根据订阅前缀获取订阅处理器
	 * 
	 * @param subPrefix
	 * @return
	 */
	public static SubscribeHandler getHandler(String subPrefix) {
		return HANDLER_MAP.get(subPrefix);
	}
}
```


```
public class TopicSubscribeHandler extends SubscribeHandler {

	public TopicSubscribeHandler(String exchangeName, String subPrefix) {
		this.subPrefix = subPrefix;
		this.exchange = new TopicExchange(exchangeName);
	}
	
	@Override
	public void config(String subSuffix, String userId) {
		String queueName = subPrefix.substring(1, subPrefix.length() - 1) + "-" + userId;
		this.queue = new Queue(queueName, false, false, true, null);
		String routingKey = subSuffix;
		this.binding = BindingBuilder.bind(queue).to(exchange).with(routingKey).noargs();
	}

}
```

```
public class OperSubscribeHandler extends SubscribeHandler {

	public OperSubscribeHandler(String exchangeName, String subPrefix) {
		this.subPrefix = subPrefix;
		this.exchange = new TopicExchange(exchangeName);
	}
	
	@Override
	public void config(String subSuffix, String userId) {
		String queueName = subPrefix.substring(1, subPrefix.length() - 1) + "-" + userId;
		this.queue = new Queue(queueName, true, false, false, null);
		String routingKey = subSuffix;
		this.binding = BindingBuilder.bind(queue).to(exchange).with(routingKey).noargs();
	}

}
```


```
public class UserSubscribeHandler extends SubscribeHandler {

	public UserSubscribeHandler(String exchangeName, String subPrefix) {
		this.subPrefix = subPrefix;
		this.exchange = new DirectExchange(exchangeName);
	}
	
	@Override
	public void config(String subSuffix, String userId) {
		String queueName = subPrefix.substring(1, subPrefix.length() - 1) + "-" + userId;
		this.queue = new Queue(queueName, true, false, false, null);
		String routingKey = subSuffix + "$" + userId;
		this.binding = BindingBuilder.bind(queue).to(exchange).with(routingKey).noargs();
	}

}
```

