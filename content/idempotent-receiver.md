# 幂等接收者（Idempotent Receiver）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/idempotent-receiver.html

识别来自客户端的请求是否唯一，以便在客户端重试时，忽略重复的请求。

**2021.1.26**

## 问题

客户端给服务器发请求，可能没有得到应答。客户端不可能知道请求是丢失了，还是服务端在处理请求之前就崩溃了。为了确保请求得到处理，客户端唯有重复发送请求。

如果服务器已经处理了请求，然后奔溃了，之后，客户端重试时，服务器端会收到客户端的重复请求。

## 解决方案

每个客户端都会分配得到一个唯一的 ID，用以对客户端进行识别。发送任何请求之前，客户端需要先向服务器进行注册。

```java
class ConsistentCoreClient…
  private void registerWithLeader() {
      RequestOrResponse request
              = new RequestOrResponse(RequestId.RegisterClientRequest.getId(),
              correlationId.incrementAndGet());

      //blockingSend will attempt to create a new connection if there is a network error.
      RequestOrResponse response = blockingSend(request);
      RegisterClientResponse registerClientResponse
              = JsonSerDes.deserialize(response.getMessageBodyJson(),
              RegisterClientResponse.class);
      this.clientId = registerClientResponse.getClientId();
  }
```

当服务器接收到来自客户端的注册请求，它就给客户端分配一个唯一的 ID，如果服务器是一个[一致性内核（Consistent Core）](content/consistent-core.md)，它可以先分配预写日志索引当做客户端标识符。

```java
class ReplicatedKVStore…
  private Map<Long, Session> clientSessions = new ConcurrentHashMap<>();

  private RegisterClientResponse registerClient(WALEntry walEntry) {
      Long clientId = walEntry.getEntryId();
      //clientId to store client responses.
      clientSessions.put(clientId, new Session(clock.nanoTime()));
      return new RegisterClientResponse(clientId);
  }
```

服务器会创建一个会话（session），以便为注册客户端的请求存储应答。它还会追踪会话的创建时间，这样，会话不起作用时，就可以把它丢弃了，这会在后面详细讨论。

```java
class Session {
    long lastAccessTimestamp;
    Queue<Response> clientResponses = new ArrayDeque<>();

    public Session(long lastAccessTimestamp) {
        this.lastAccessTimestamp = lastAccessTimestamp;
    }

    public long getLastAccessTimestamp() {
        return lastAccessTimestamp;
    }

    public Optional<Response> getResponse(int requestNumber) {
        return clientResponses.stream().
                filter(r -> requestNumber == r.getRequestNumber()).findFirst();
    }

    private static final int MAX_SAVED_RESPONSES = 5;

    public void addResponse(Response response) {
        if (clientResponses.size() == MAX_SAVED_RESPONSES) {
            clientResponses.remove(); //remove the oldest request
        }
        clientResponses.add(response);
    }

    public void refresh(long nanoTime) {
        this.lastAccessTimestamp = nanoTime;
    }
}
```

对一个一致性内核而言，客户端的请求也要作为共识算法的一部分进行复制。如此一来，即便既有的领导者失效了，客户端注册依然是可用的。对于后续的请求，服务器还要存储发送给客户端的应答。

>幂等和非幂等请求
>>注意到一些请求的幂等属性是很重要的。比如说，在一个键值存储中，设置键值和值就天然是幂等的。即便同样的键值和值设置了多次，也不会产生什么问题。
>>另一方面，创建[租约（Lease）](https://martinfowler.com/articles/patterns-of-distributed-systems/time-bound-lease.html)却并不幂等。如果租约已经创建，再次尝试创建租约 的请求就会失败。这就是问题了。考虑一下这样一个场景。一个客户端发送请求创建租约，服务器成功地创建了租约，然后，崩溃了，或者是，应答发给客户端之前连接断开了。客户端会重新创建连接，重新尝试创建租约；因为服务端已经有了一个指定名称的租约，所以，它会返回错误。因此，客户端就会认为没有这个租约。显然，这并不是我们预期的行为。
>>有了幂等接收者，客户端会用同样的请求号发送租约请求。因为表示“请求已经处理过”的应答已经存在服务器上了，这个应答就可以直接返回。这样一来，如果是客户端在连接断开之前已经成功创建了租约，后续重试相同的请求时，它会得到应有的应答。

对于收到的每个非幂等请求（参见边栏），服务端成功执行之后，都会将应答存在客户端会话中。

```java
class ReplicatedKVStore…
  private Response applyRegisterLeaseCommand(WALEntry walEntry, RegisterLeaseCommand command) {
      logger.info("Creating lease with id " + command.getName()
+ "with timeout " + command.getTimeout()
+ " on server " + getServer().getServerId());
      try {
          leaseTracker.addLease(command.getName(),
command.getTimeout());
          Response success = Response.success(walEntry.getEntryId());
          if (command.hasClientId()) {
              Session session = clientSessions.get(command.getClientId());
              session.addResponse(success.withRequestNumber(command.getRequestNumber()));
          }
          return success;

      } catch (DuplicateLeaseException e) {
          return Response.error(1, e.getMessage(), walEntry.getEntryId());
      }
  }
```

客户端发送给服务器的每个请求里都包含客户端的标识符。客户端还保持了一个计数器，每个发送给服务器的请求都会分配到一个请求号。

```java
class ConsistentCoreClient…
  int nextRequestNumber = 1;

  public void registerLease(String name, long ttl) {
      RegisterLeaseRequest registerLeaseRequest
              = new RegisterLeaseRequest(clientId, nextRequestNumber, name, ttl);

      nextRequestNumber++; //increment request number for next request.
      var serializedRequest = serialize(registerLeaseRequest);

      logger.info("Sending RegisterLeaseRequest for " + name);
      blockingSendWithRetries(serializedRequest);
  }

  private static final int MAX_RETRIES = 3;

  private RequestOrResponse blockingSendWithRetries(RequestOrResponse request) {
      for (int i = 0; i <= MAX_RETRIES; i++) {
          try {
              //blockingSend will attempt to create a new connection is there is no connection.
              return blockingSend(request);
          } catch (NetworkException e) {
              resetConnectionToLeader();
              logger.error("Failed sending request  " + request + ". Try " + i, e);
          }
      }
      throw new NetworkException("Timed out after " + MAX_RETRIES + " retries");
  }
```

服务器收到请求时，它会先检查来自同一个客户端给定的请求号是否已经处理过了。如果找到已保存的应答，它就会把相同的应答返回给客户端，而无需重新处理请求。

```java
class ReplicatedKVStore…
  private Response applyWalEntry(WALEntry walEntry) {
      Command command = deserialize(walEntry);
      if (command.hasClientId()) {
          Session session = clientSessions.get(command.getClientId());
          Optional<Response> savedResponse = session.getResponse(command.getRequestNumber());
          if(savedResponse.isPresent()) {
              return savedResponse.get();
          } //else continue and execute this command.
      }
```

### 已保存的客户端请求过期处理

按客户端存储的请求不可能是永久保存的。有几种方式可以对请求进行过期处理。在 Raft 的[参考实现](https://github.com/logcabin/logcabin)中，客户端会保存一个单独的号码，以便记录成功收到应答的请求号。这个号码稍后会随着每个请求发送给服务器。这样，对于请求号小于这个号码的请求，服务器就可以安全地将其丢弃了。

如果客户端能够保证只在接收到上一个请求的应答之后，再发起下一个请求，那么，服务器端一旦接收到来自这个客户端的请求，就可以放心地删除之前所有的请求。使用[请求管道（Request Pipeline）](https://martinfowler.com/articles/patterns-of-distributed-systems/request-pipeline.html)还会有个问题，可能有在途（in-flight）请求存在，也就是客户端没有收到应答。如果服务器端知道客户端能够接受的在途请求的最大数量，它就可以保留那么多的应答，删除其它的应答。比如说，[kafka](https://kafka.apache.org/) 的 producer 能够接受的最大在途请求数量是 5 个，因此，它最多保存 5 个之前的请求。

```java
class Session…
  private static final int MAX_SAVED_RESPONSES = 5;

  public void addResponse(Response response) {
      if (clientResponses.size() == MAX_SAVED_RESPONSES) {
          clientResponses.remove(); //remove the oldest request
      }
      clientResponses.add(response);
  }
```

### 删除已注册的客户端

客户端的回话也不会在服务器上永久保存。一个服务器会对其存储的客户端回话有一个最大保活时间。客户单周期性地发送[心跳（HeartBeat）](heartbeat.md)。如果在保活时间内没有收到心跳，服务器上客户端的状态就会被删除掉。

服务器会启动一个定时任务，周期性地检查是否有过期会话，删除已过期的会话。

```java
class ReplicatedKVStore…
  private long heartBeatIntervalMs = TimeUnit.SECONDS.toMillis(10);
  private long sessionTimeoutNanos = TimeUnit.MINUTES.toNanos(5);

  private void startSessionCheckerTask() {
      scheduledTask = executor.scheduleWithFixedDelay(()->{
removeExpiredSession();
      }, heartBeatIntervalMs, heartBeatIntervalMs, TimeUnit.MILLISECONDS);
  }

  private void removeExpiredSession() {
      long now = System.nanoTime();
      for (Long clientId : clientSessions.keySet()) {
          Session session = clientSessions.get(clientId);
          long elapsedNanosSinceLastAccess = now - session.getLastAccessTimestamp();
          if (elapsedNanosSinceLastAccess > sessionTimeoutNanos) {
              clientSessions.remove(clientId);
          }
      }
  }
```

## 示例

[Raft](https://raft.github.io/) 有一个实现了幂等性的参考实现，提供了线性一致性的行为。

[Kafka](https://kafka.apache.org/) 有一个[幂等 Producer](https://cwiki.apache.org/confluence/display/KAFKA/Idempotent+Producer)，允许客户端重试请求，忽略重复的请求。

[ZooKeeper](https://zookeeper.apache.org/) 有 Session 的概念，还有 zxid，用于客户端恢复。HBase 有一个 [hbase-recoverable-zookeeper](https://docs.cloudera.com/HDPDocuments/HDP2/HDP-2.4.0/bk_hbase_java_api/org/apache/hadoop/hbase/zookeeper/RecoverableZooKeeper.html) 的封装，它实现了遵循 [zookeeper-error-handling](https://cwiki.apache.org/confluence/display/ZOOKEEPER/ErrorHandling) 指导的幂等的行为。
