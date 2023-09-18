# WebSokcet

百度百科：**WebSocket**是一种在单个[TCP](https://baike.baidu.com/item/TCP?fromModule=lemma_inlink)连接上进行[全双工](https://baike.baidu.com/item/全双工?fromModule=lemma_inlink)通信的协议。

参考: [SpringBoot 集成 WebSocket，实现后台向前端推送信息](https://cloud.tencent.com/developer/article/1830998)



> 依赖

```xml
 <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```



> websocket配置类

```java
@Configuration
public class WebSocketConfig extends ServerEndpointConfig.Configurator {
    // 注入不进来
//    String token_header = "Authorization";

    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }

    /**
     * 注入service，因为WebSocket启动的时候优先于spring容器，从而导致在WebSocketServer中调用业务Service会报空指针异常
     * @param tokenService
     */
    @Autowired
    private void setTokenService(TokenService tokenService) {
        WebSocketServer.tokenService = tokenService;
    }

    @Override
    public void modifyHandshake(ServerEndpointConfig sec, HandshakeRequest request, HandshakeResponse response) {
//        List<String> authorizationList = request.getHeaders().get(token_header);
//        if (!CollUtil.isEmpty(authorizationList)) {
//            // 将auth信息传入session中
//            Map<String, Object> userProperties = sec.getUserProperties();
//            userProperties.put(token_header, authorizationList.get(0));
//        }
//        response.getHeaders().put(token_header, authorizationList);
        super.modifyHandshake(sec, request, response);
    }
}
```

继承ServerEndpointConfig.Configurator是为了从请求中获取认证信息（因为html中webSocket无法设置header所以最后放弃了）。需要注意的是，webSocket服务的启动会优先于spring容器，所以无法直接将service注入到WebSocketServer中，可以通过以上方式实现注入。



> webSocketServer

```java
@Component
@Slf4j
@ServerEndpoint(value = "/websocket/{userId}", configurator = WebSocketConfig.class)
public class WebSocketServer {
    //    static final String TOKEN_HEADER = "Authorization";
    // 与某个客户端的连接会话,需要通过它来给客户端发送数据
    private Session session;

    private String userId;
    public static TokenService tokenService;
    // 当前在线连接数
    private static final AtomicInteger onlineCount = new AtomicInteger(0);
    // 存储用户session, key: 用户ID, value: 用户在每个终端的session
    private static final ConcurrentHashMap<String, WebSocketServer> webSocketMap = new ConcurrentHashMap<>();

    /**
     * 连接建立成功调用的方法
     */
    @OnOpen
    public void onOpen(Session session, @PathParam("userId") String userId) throws IOException {
        this.session = session;
//        Map<String, Object> userProperties = session.getUserProperties();
//        String token = (String) userProperties.get(TOKEN_HEADER);
//        if (StringUtils.isEmpty(token)) {
//            session.close();
//            return;
//        }
//        LoginUser loginUser = tokenService.getLoginUser(token);
//        if (loginUser == null) {
//            session.close();
//            return;
//        }
//        this.userId = String.valueOf(loginUser.getUser().getUserId());
        this.userId = userId;
        // 若原session存在，将原session删除后再添加
        if (webSocketMap.containsKey(userId)) {
            webSocketMap.remove(userId);
            webSocketMap.put(userId, this);
        } else {
            // 直接添加session
            webSocketMap.put(userId, this);
            addOnlineCount();
        }
        // TODO 删除下面的测试代码
        log.info("有新窗口开始监听:" + userId + ",当前在线人数为:" + getOnlineCount());

    }

    /**
     * 连接关闭调用的方法
     */
    @OnClose
    public void onClose() {
        if (userId == null) return;
        WebSocketServer remove = webSocketMap.remove(this.userId);
        if (remove != null) {
            subOnlineCount();
        }
        // TODO 删除下面的测试代码
        //断开连接情况下，更新主板占用情况为释放
        log.info("释放的userId为：" + userId);
        //这里写你 释放的时候，要处理的业务
        log.info("有一连接关闭！当前在线人数为" + getOnlineCount());

    }

    /**
     * 收到客户端消息后调用的方法
     *
     * @ Param message 客户端发送过来的消息
     * @ Param session session
     */
    @OnMessage
    public void onMessage(String message, Session session) {
        log.info("收到来自窗口" + userId + "的信息:" + message);
    }

    /**
     * @ Param session
     * @ Param error
     */
    @OnError
    public void onError(Session session, Throwable error) {
        log.error("发生错误");
        error.printStackTrace();
    }

    /**
     * 实现服务器主动推送
     */
    public void sendMessage(WebSocketVO vo) throws IOException {
        this.session.getBasicRemote().sendText(new ObjectMapper().writeValueAsString(vo));
    }

    /**
     * 发送自定义消息
     */
    public static void sendInfo(WebSocketVO webSocketVO, String userId) {
        WebSocketServer server = webSocketMap.get(userId);
        if (server == null) return;
        try {
            server.sendMessage(webSocketVO);
        } catch (Exception e) {
            log.error("WebSocketServer:sendInfo:" + e.getMessage());
        }

    }

    public static synchronized int getOnlineCount() {
        return onlineCount.get();
    }

    public static synchronized void addOnlineCount() {
        WebSocketServer.onlineCount.incrementAndGet();
    }

    public static synchronized void subOnlineCount() {
        WebSocketServer.onlineCount.decrementAndGet();
    }
}
```

通过路径传递参数，类上`@ServerEndpoint("/api/websocket/{sid}")`，然后方法上`public void onOpen(Session session, @PathParam("sid") String sid)`

如果一个user可以登录多台设备则webSocketMap可以是`ConcurrentHashMap<String, ArrayList<WebSocketServer>>`，然后通过userId找到用户的所有设备的WebSocket连接，再找session.id相同的。

WebSocketVO是我封装的一个返回类型。

> 测试html

```html
<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <title>Java后端WebSocket的Tomcat实现</title>
  <script type="text/javascript" src="js/jquery.min.js"></script>
</head>

<body>
  <div id="main" style="width: 1200px;height:800px;"></div>
  Welcome<br /><input id="text" type="text" />
  <button onclick="send()">发送消息</button>
  <hr />
  <button onclick="closeWebSocket()">关闭WebSocket连接</button>
  <hr />
  <div id="message"></div>
</body>
<script type="text/javascript">
  var websocket = null;
  //判断当前浏览器是否支持WebSocket
  if ('WebSocket' in window) {
    //改成你的地址
    websocket = new WebSocket("ws://127.0.0.1:8099/admin-server/websocket/100");
  } else {
    alert('当前浏览器 Not support websocket')
  }

  //连接发生错误的回调方法
  websocket.onerror = function () {
    setMessageInnerHTML("WebSocket连接发生错误");
  };

  //连接成功建立的回调方法
  websocket.onopen = function () {
    setMessageInnerHTML("WebSocket连接成功");
  }
  var U01data, Uidata, Usdata
  //接收到消息的回调方法
  websocket.onmessage = function (event) {
    var msg = event.data;
    console.log(msg);
    setMessageInnerHTML(msg);
    setechart()
  }

  //连接关闭的回调方法
  websocket.onclose = function () {
    setMessageInnerHTML("WebSocket连接关闭");
  }

  //监听窗口关闭事件，当窗口关闭时，主动去关闭websocket连接，防止连接还没断开就关闭窗口，server端会抛异常。
  window.onbeforeunload = function () {
    closeWebSocket();
  }

  //将消息显示在网页上
  function setMessageInnerHTML(innerHTML) {
    document.getElementById('message').innerHTML += innerHTML + '<br/>';
  }

  //关闭WebSocket连接
  function closeWebSocket() {
    websocket.close();
  }

  //发送消息
  function send() {
    var message = document.getElementById('text').value;
    websocket.send('{"msg":"' + message + '"}');
    setMessageInnerHTML(message + "&#13;");
  }
</script>

</html>
```

