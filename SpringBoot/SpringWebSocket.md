## WebSocket 说明

### 一. 概念

#### 1. Http 协议

	必须由Client向Server发送请求，Server返回结果，而无法由Server直接向Client发送消息。传统交互模式有 long poll 和 ajax轮询。

##### long pull

	客户端向服务器发送Ajax请求，服务器接到请求后hold住连接，直到有新消息才返回响应信息并关闭连接，客户端处理完响应信息后再向服务器发送新的请求。

##### ajax 轮询

	客户端定时向服务器发送Ajax请求，服务器接到请求后马上返回响应信息并关闭连接。

#### 2. Websocket协议

	一种在单个 TCP 连接上进行全双工通讯的协议。一旦WebSocket连接建立后，后续数据都以帧序列的形式传输。在客户端断开WebSocket连接或Server端中断连接前，不需要客户端和服务端重新发起连接请求。在海量并发及客户端与服务器交互负载流量大的情况下，极大的节省了网络带宽资源的消耗，有明显的性能优势，且客户端发送和接受消息是在同一个持久连接上发起，实时性优势明显。

##### 客户端

	new WebSocket实例化一个新的WebSocket客户端对象，请求类似 **ws://yourdomain:port/path** 的服务端WebSocket URL，客户端WebSocket对象会自动解析并识别为WebSocket请求，并连接服务端端口，执行双方握手过程，客户端发送数据格式类似

- Upgrade： websocket参数值表明这是WebSocket类型请求
- Sec-WebSocket-Key：客户端发送base64编码的密文，要求服务端必须返回一个对应加密的Sec-WebSocket-Accept应答，否则客户端会抛出Error during WebSocket handshake错误，并关闭连接。

```http
GET /webfin/websocket/ HTTP/1.1
Host: localhost
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: xqBt3ImNzJbYqRINxEFlkg==
Origin: http://localhost:8080
Sec-WebSocket-Version: 13
```

##### 服务端

返回的格式类型如下

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: K7DJLdLooIwIG/MOpvWFB3y3FE8=
```

- Sec-WebSocket-Accept：服务端采用与客户端一致的密钥计算出来后返回客户端
- HTTP/1.1 101 Switching Protocols：服务端接受WebSocket协议的客户端连接，经过这样的请求-响应处理后，两端的WebSocket连接握手成功, 后续就可以进行TCP通讯了。

![websocket](./websocket.png)

##### 优点

（1）建立在 TCP 协议之上，服务器端的实现比较容易。

（2）与 HTTP 协议有着良好的兼容性。默认端口也是80和443，并且握手阶段采用 HTTP 协议，因此握手时不容易屏蔽，能通过各种 HTTP 代理服务器。

（3）数据格式比较轻量，性能开销小，通信高效。

（4）可以发送文本，也可以发送二进制数据。

（5）没有同源限制，客户端可以与任意服务器通信。

（6）协议标识符是`ws`（如果加密，则为`wss`），服务器网址就是 URL。

##### STOMP

​	WebSocket协议定义了两种类型的消息，文本和二进制，但是它们的内容是未定义的。直接使用Websocket协议开发非常麻烦，Stomp协议是一个更高级的Websocket的子协议。

	STOMP即Simple (or Streaming) Text Orientated Messaging Protocol，简单(流)文本定向消息协议，它提供了一个可互操作的连接格式，允许STOMP客户端与任意STOMP消息代理（Broker）进行交互。STOMP基于帧（frame）的格式来定义消息。frame格式如下：

```http
COMMAND
header1:value1
header2:value2

Body^@
```

### 二. Springboot实现

添加依赖如下

```xml
<properties>
    <stomp.version>2.3.3</stomp.version>
    <sockjs.version>1.1.2</sockjs.version>
</properties>
<dependency>
  	<groupId>org.springframework.boot</groupId>
  	<artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
<dependency>
    <groupId>org.webjars</groupId>
    <artifactId>stomp-websocket</artifactId>
    <version>${stomp.version}</version>
</dependency>
<dependency>
    <groupId>org.webjars</groupId>
    <artifactId>sockjs-client</artifactId>
    <version>${sockjs.version}</version>
</dependency>
```

#### 1. 广播式

Spring WebSocket应用程序充当客户端的STOMP代理，消息被路由到@Controller的处理方法@MessageMapping("/hello")。当服务端有消息时，会将消息发送给所有连接了当前 endpoint 的浏览器。具体实现如下：

- 开启 WebSocket STOMP 支持

```java
/**
 * @EnableWebSocketMessageBroker 开启使用 STOMP 协议来处理 Message broker 的消息（@MessageMapping）
 * 
 * @author dante
 */
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer {

	/**
	 * STOMP 协议节点，指定客户端使用 SockJS 协议进行连接 
	 */
	@Override
	public void registerStompEndpoints(StompEndpointRegistry registry) {
		registry.addEndpoint("/hello_websocket").withSockJS(); 
	}

	/**
	 * 配置广播式 Topic 的消息代理
	 */
	@Override
	public void configureMessageBroker(MessageBrokerRegistry config) {
		config.enableSimpleBroker("/topic");
	}
```

- 控制器

```java
/**
* 客户端发送请求的URL，通过 @MessageMapping 映射
* @SendTo, 即destination。消息订阅的路径，服务器向订阅了此路径的客户端发送消息
* 
* @param request
* @return
*/
@MessageMapping("/hello")
@SendTo("/topic/get_msg_resp")
public MessageResponse sendMsgResp(MessageRequest request) {
  	log.info("收到消息 {}.", request);
  	return new MessageResponse("消息[" + request.getName() + "]，序号" + new Random().nextInt(9) + 1);
}
```

- 客户端（浏览器）

```html
<!DOCTYPE html>
<html>
<head>
    <title>Hello WebSocket, 广播式</title>
    <script src="/jquery.min.js"></script>
    <script src="/webjars/sockjs-client/1.1.2/sockjs.min.js"></script>
    <script src="/webjars/stomp-websocket/2.3.3/stomp.min.js"></script>
</head>
<body>
	
	<div style="margin: 20px;padding: 15px;">
		<button id="conn">连接</button>
		<button id="disconn" disabled="disabled">断开</button>
		<br/>
		<input type="text" id="msg" style="width: 120px" />
		<button id="sendBtn">发送消息</button>
	</div>
	<div id="retMsg"></div>
	<div style="margin-top: 15px">
		<label>状态的变化：</label>
		<span id="curStatus"></span>
	</div>
	
	<script type="text/javascript">
		var retryCount = 0;
		var TopicPage = {
			stompClient: null,
			selfDisconn: false,
			connect: function() {
				this.selfDisconn = false;
				// 通过 SocketJS 注册 StompEndpoint
				var socket = new SockJS('/hello_websocket');
              	 // 使用 STOMP 子协议的 WebSocket 客户端
				this.stompClient = Stomp.over(socket);
				// 连接服务器，Stomp 消息基于帧（frame）
				this.stompClient.connect({}, function (frame) {
					TopicPage.connectStatusChange(true);
					console.log('Connected: ' + frame);
                  	  // 消息订阅，地址即服务器 @SendTo 的值
					TopicPage.stompClient.subscribe('/topic/get_msg_resp', function (returnMsg) {
			            $('#retMsg').text(JSON.parse(returnMsg.body).respMsg);
			        });
					
					TopicPage.stompClient.subscribe('/topic/status_change', function (msg) {
						console.log('=============> ' + msg.body);
						var status = msg.body;
			            $('#curStatus').text('当前状态：' + status).css('color' , (status == 1 ? 'red' : (status == 2 ? 'blue' : 'green')));
			        });
				});
			},
			connectStatusChange: function(connected) {
				$('#conn').prop('disabled', connected);
				$('#disconn').prop('disabled', !connected);
			},
			disconnect: function() {
				this.selfDisconn = true;
				if (this.stompClient != null) {
			        this.stompClient.disconnect();
			    }
			    this.connectStatusChange(false);
			    console.log("Disconnected");
			},
			checkConn: function() {
				setInterval(function() {
					if(TopicPage.selfDisconn) {
						return;
					}
					// 重连 10 次不成功，将不再尝试
					if(retryCount == 10) {
						return;
					}
					if(TopicPage.stompClient) {
						if(!TopicPage.stompClient.connected) {
							console.log('连接丢失，重新连接！' + retryCount)
							TopicPage.connect();
							retryCount++;
						}
					}
				}, 3000);
			},
			init: function() {
				this.checkConn();
				$('#conn').click(function(){
					TopicPage.connect();
				});
				$('#disconn').click(function(){
					TopicPage.disconnect();
				});
				$('#sendBtn').click(function() {
                  	  // 向 /hello destination 发送消息，在 @MessageMapping 中定义
					TopicPage.stompClient.send("/hello", {}, JSON.stringify({
                        'name': $('#msg').val()
                    }));
				});
			}
		};
		$(function() {
			TopicPage.init();
		});
	</script>
</body>
</html>
```

#### 2. 点对点

	应用程序可以发送针对特定用户的消息，Spring的STOMP支持可以识别以“/user/”为前缀的目标。每个用户必须有自己的 Session 信息。

- 添加 Spring-Security依赖

```xml
<dependency>
  	<groupId>org.springframework.boot</groupId>
  	<artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

- 服务端实现

```java
// 添加 /chat_room Endpoint
registry.addEndpoint("/chat_room").withSockJS(); 

// 配置点对点 queue 的消息代理
config.enableSimpleBroker("/queue");

// 配置页面映射
registry.addViewController("/login").setViewName("login");
registry.addViewController("/chat").setViewName("queue");

// 消息处理
/**
* 模拟单聊
*/
@MessageMapping("/p2p")
public void handleChat(Principal principal, String msg) {
  if (principal.getName().equals("snake")) {
    simpMessagingTemplate.convertAndSendToUser("dante", "/queue/chat", "snake -> " + msg);
  } else {
    simpMessagingTemplate.convertAndSendToUser("snake", "/queue/chat", "dante -> " + msg);
  }
}
```

- Spring-Security 配置

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
	/**
     * 配置访问路径
     *
     * @param http
     * @throws Exception
     */
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                //设置Spring Security对"/"、"/login"路径不拦截
                .antMatchers("/","/login","/topic").permitAll()
                .anyRequest().authenticated()
                .and()
                .formLogin()
                .loginPage("/login")//指定登录页面
                .defaultSuccessUrl("/chat")//登录成功后跳转的页面
                .permitAll()
                .and()
                .logout()
                .permitAll();
    }
    
    /**
     * 指定用户信息
     *
     * @param auth
     * @throws Exception
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        //指定用户名和密码及其角色
        auth.inMemoryAuthentication()
                .withUser("dante").password("123456").roles("USER")
                .and()
                .withUser("snake").password("123456").roles("USER");
    }
}
```

- 客户端（浏览器）

```html
<!DOCTYPE html>
<html>
<head>
    <title>Hello WebSocket, 点对点</title>
    <script src="/jquery.min.js"></script>
    <script src="/webjars/sockjs-client/1.1.2/sockjs.min.js"></script>
    <script src="/webjars/stomp-websocket/2.3.3/stomp.min.js"></script>
</head>
<body>
	
	<div style="margin: 20px;padding: 15px;">
		<button id="conn">连接</button>
		<button id="disconn" disabled="disabled">断开</button>
		<br/>
		<input type="text" id="msg" style="width: 120px" />
		<button id="sendBtn">发送消息</button>
	</div>
	<div id="retMsg"></div>
	
	<script type="text/javascript">
		var retryCount = 0;
		var TopicPage = {
			stompClient: null,
			selfDisconn: false,
			connect: function() {
				this.selfDisconn = false;
				// 通过 SocketJS 注册 StompEndpoint
				var socket = new SockJS('/chat_room');
				this.stompClient = Stomp.over(socket);
				// Stomp 消息基于帧（frame）
				this.stompClient.connect({}, function (frame) {
					TopicPage.connectStatusChange(true);
					console.log('Connected: ' + frame);
					TopicPage.stompClient.subscribe('/user/queue/chat', function (returnMsg) {
			            $('#retMsg').append('<div>'+returnMsg.body+'</div>');
			        });
				});
			},
			connectStatusChange: function(connected) {
				$('#conn').prop('disabled', connected);
				$('#disconn').prop('disabled', !connected);
			},
			disconnect: function() {
				this.selfDisconn = true;
				if (this.stompClient != null) {
			        this.stompClient.disconnect();
			    }
			    this.connectStatusChange(false);
			    console.log("Disconnected");
			},
			checkConn: function() {
				setInterval(function() {
					if(TopicPage.selfDisconn) {
						return;
					}
					// 重连 10 次不成功，将不再尝试
					if(retryCount == 10) {
						return;
					}
					if(TopicPage.stompClient) {
						if(!TopicPage.stompClient.connected) {
							console.log('连接丢失，重新连接！' + retryCount)
							TopicPage.connect();
							retryCount++;
						}
					}
				}, 3000);
			},
			registerEvt: function() {
				this.checkConn();
				$('#conn').click(function(){
					TopicPage.connect();
				});
				$('#disconn').click(function(){
					TopicPage.disconnect();
				});
				$('#sendBtn').click(function() {
					TopicPage.stompClient.send("/p2p", {}, $('#msg').val());
				});
			}
		};
		$(function() {
			TopicPage.registerEvt();
		});
	</script>
</body>
</html>
```

### 四. 参考资料

- https://docs.spring.io/spring/docs/5.0.2.RELEASE/spring-framework-reference/web.html#websocket

- https://www.zhihu.com/question/20215561