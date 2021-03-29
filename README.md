## Spring Boot Client

# 1. Introdução
HTTP (Hypertext Transfer Protocol) é um protocolo de solicitação-resposta sem estado. Seu design simples o torna muito escalável, mas inadequado e ineficiente para aplicativos da web em tempo real altamente interativos devido à quantidade de sobrecarga que precisa ser transmitida junto com cada solicitação / resposta.

Como o HTTP é síncrono e os aplicativos em tempo real precisam ser assíncronos, quaisquer soluções como polling ou long polling (Comet) tendem a ser complicadas e ineficientes.

Para resolver o problema especificado acima, precisamos de um protocolo bidirecional e full-duplex baseado em padrões que possa ser usado por servidores e clientes, e isso levou à introdução da API JSR 356 - neste artigo, nós ' vou mostrar um exemplo de uso dele.

# 2. Configuração
Vamos incluir as dependências do Spring WebSocket em nosso projeto:

```
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-websocket</artifactId>
    <version>5.2.2.RELEASE</version>
 </dependency>
 <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-messaging</artifactId>
    <version>5.2.2.RELEASE</version>
 </dependency>
```

Sempre podemos obter as versões mais recentes das dependências do Maven Central para spring-websocket e spring-messaging.

# 3. STOMP
O STOMP (Stream Text-Oriented Messaging Protocol) é um formato de conexão simples e interoperável que permite que o cliente e os servidores se comuniquem com quase todos os agentes de mensagens. É uma alternativa ao AMQP (Advanced Message Queuing Protocol) e JMS (Java Messaging Service).

STOMP define um protocolo para cliente / servidor se comunicar usando semântica de mensagens. A semântica está no topo dos WebSockets e define quadros que são mapeados em quadros WebSockets.

Usar o STOMP nos dá flexibilidade para desenvolver clientes e servidores em diferentes linguagens de programação. Neste exemplo atual, usaremos STOMP para mensagens entre cliente e servidor.

# 4. Servidor WebSocket
Você pode ler mais sobre a construção de servidores WebSocket neste artigo.

# 5. Cliente WebSocket

Para se comunicar com o servidor WebSocket, o cliente deve iniciar a conexão WebSocket enviando uma solicitação HTTP a um servidor com um cabeçalho de atualização definido corretamente:

```
GET ws://websocket.example.com/ HTTP/1.1
Origin: http://example.com
Connection: Upgrade
Host: websocket.example.com
Upgrade: websocket
```

Observe que os URLs do WebSocket usam esquemas ws e wss, o segundo significa WebSockets seguros.

O servidor responde enviando o cabeçalho Upgrade na resposta se o suporte a WebSockets estiver habilitado.

```
HTTP/1.1 101 WebSocket Protocol Handshake
Date: Wed, 16 Oct 2013 10:07:34 GMT
Connection: Upgrade
Upgrade: WebSocket
```

Depois que esse processo (também conhecido como handshake WebSocket) for concluído, a conexão HTTP inicial será substituída pela conexão WebSocket na mesma conexão TCP / IP, após a qual qualquer uma das partes poderá compartilhar dados.

Esta conexão do lado do cliente é iniciada pela instância WebSocketStompClient.

# 5.1. O WebSocketStompClient
Conforme descrito na seção 3, primeiro precisamos estabelecer uma conexão WebSocket, e isso é feito usando a classe WebSocketClient.

O WebSocketClient pode ser configurado usando:

- StandardWebSocketClient fornecido por qualquer implementação JSR-356 como Tyrus;
- JettyWebSocketClient fornecido pela API WebSocket nativa Jetty 9+;
- Qualquer implementação do WebSocketClient da Spring.

Usaremos StandardWebSocketClient, uma implementação de WebSocketClient em nosso exemplo:

```
WebSocketClient client = new StandardWebSocketClient();

WebSocketStompClient stompClient = new WebSocketStompClient(client);
stompClient.setMessageConverter(new MappingJackson2MessageConverter());

StompSessionHandler sessionHandler = new MyStompSessionHandler();
stompClient.connect(URL, sessionHandler);

new Scanner(System.in).nextLine(); // Don't close immediately.
```

Por padrão, WebSocketStompClient oferece suporte a SimpleMessageConverter. Como estamos lidando com mensagens JSON, configuramos o conversor de mensagem para MappingJackson2MessageConverter para converter a carga útil JSON em objeto.

Durante a conexão com um ponto de extremidade, passamos uma instância de StompSessionHandler, que lida com eventos como afterConnected e handleFrame.

Se nosso servidor tiver suporte para SockJs, podemos modificar o cliente para usar SockJsClient em vez de StandardWebSocketClient.

# 5.2. StompSessionHandler
Podemos usar um StompSession para assinar um tópico WebSocket. Isso pode ser feito criando uma instância de StompSessionHandlerAdapter que, por sua vez, implementa o StompSessionHandler.

Um StompSessionHandler fornece eventos de ciclo de vida para uma sessão STOMP. Os eventos incluem um retorno de chamada quando a sessão é estabelecida e notificações em caso de falhas.

Assim que o cliente WebSocket se conecta ao endpoint, o StompSessionHandler é notificado e o método afterConnected() é chamado, onde usamos o StompSession para assinar o tópico:

```
@Override
public void afterConnected(
  StompSession session, StompHeaders connectedHeaders) {
    session.subscribe("/topic/messages", this);
    session.send("/app/chat", getSampleMessage());
}
@Override
public void handleFrame(StompHeaders headers, Object payload) {
    Message msg = (Message) payload;
    logger.info("Received : " + msg.getText()+ " from : " + msg.getFrom());
}
```

Certifique-se de que o servidor WebSocket esteja executando e executando o cliente, a mensagem será exibida no console:

```
INFO o.b.w.client.MyStompSessionHandler - New session established : 53b993eb-7ad6-4470-dd80-c4cfdab7f2ba
INFO o.b.w.client.MyStompSessionHandler - Subscribed to /topic/messages
INFO o.b.w.client.MyStompSessionHandler - Message sent to websocket server
INFO o.b.w.client.MyStompSessionHandler - Received : Howdy!! from : Nicky
```

# 6. Conclusão
Neste tutorial rápido, implementamos um cliente WebSocket baseado em Spring.