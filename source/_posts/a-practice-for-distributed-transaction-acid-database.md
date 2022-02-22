title: LoT设备分布式一致性处理实践 - 本地消息表
date: 2022-02-18 14:49:30
tags:
- .NET
- Web Api
- CAP
categories:
- .NET
---


最近工作中我们新增了一批人脸识别设备，该设备支持`http commet轮询`以及`MQTT`的方式对接，综合各方面的优劣后，最终我们准备采用MQTT的方式来对接，同时我们也不可避免得遇到了分布式一致性的问题，在此做一次实践纪录。

在系统使用过程中，我们需要根据各自的权限设置，将系统中人员的白名单下发到不同的设备，后台系统通过`MQTT`将消息下发到设备，设备再通过`MQTT`将结果反馈到系统。同时因为设备的性能原因，有一些限制，设备不能在短时间内处理过多的消息，如果我们一下子把所有消息都通过`MQTT`压下去，会造成设备出现不可预期的异常，比如结果反馈延时、心跳包延时等问题。因此我们的后台系统需要处理异步消息结果一致性问题以及消息流量控制，这里我们通过`本地消息表`来实现。

## 本地消息表
本地消息表这个方案最初是ebay提出的，此方案的核心是将需要分布式处理的任务通过消息日志的方式来异步执行。消息日志可以存储到本地文本、数据库或消息队列，再通过业务规则自动或人工发起重试。

具体怎么做呢？消息生产方（也就是发起方），需要建一个消息表，并记录消息发送状态，然后消息会经过MQ发送到消息的消费方。如果消息发送失败，会进行重试发送。如果设备处理成功了，通知系统，我们就将业务数据标记为成功，同时删除本地消息。那如果设备处理成功了，但是在标记业务数据的时候后台服务崩溃了怎么办呢？由于设备处理白名单是幂等的，也就是说假如后台服务奔溃了，等它再次启动的时候，会重复下发一次，这时我们损失的是性能，但是最终一致性结果不会改变，所以这时我们选择牺牲一点性能。

## 本地消息表实现
准备本地消息表实体以及数据存储接口
```csharp

public class MediumMessage
{
    public string DbId { get; set; } = default!;

    /// <summary>
    /// 业务原始数据
    /// </summary>
    public string Origin { get; set; }

    public string Topic { get; set; }

    public string MessageId { get; set; }

    /// <summary>
    /// 消息内容
    /// </summary>
    public string Message { get; set; }

    public DateTime AddAt { get; set; }

    public DateTime? PublishAt { get; set; }

    /// <summary>
    /// 发送次数
    /// </summary>
    public int Tries { get; set; }

    public StatusName StatusName { get; set; }
}

public enum StatusName
{
    Failed = -1,
    Scheduled,
    Published
}

```

```csharp
public interface IDataStore
{

    /// <summary>
    /// 存储白名单消息
    /// </summary>
    MediumMessage StoreWhitelistMessage(string messageId, string topic, string message, WhitelistTask task,
        StatusName status);

    /// <summary>
    /// 根据messageId获取消息
    /// </summary>
    MediumMessage GetMessage(string messageId);

    /// <summary>
    /// 移除消息
    /// </summary>
    void RemoveMessage(string messageId);

    /// <summary>
    /// 白名单消息是否空
    /// </summary>
    bool IsWhitelistEmpty();

    /// <summary>
    /// 是否存在已发送消息的topic
    /// </summary>
    bool ExistPublishedTopic(string topic);

    /// <summary>
    /// 获取需要重试发送的消息
    /// </summary>
    IEnumerable<MediumMessage> GetMessagesOfNeedRetry();

    /// <summary>
    /// 更新发布状态
    /// </summary>
    void ChangePublishState(string messageId);

    /// <summary>
    /// 删除过期的消息
    /// </summary>
    int DeleteExpiredMessages();
}
```

项目初期，我们决定将数据存在内存中，所以目前我们是通过将消息存储在`ConcurrentDictionary`中实现的，虽然`ConcurrentDictionary`是线程安全的，但仅仅是在不同消息实体之间加锁，我们在更新单个消息的多个字段内容时，还是要注意配合其他混合锁使用。

## 后台任务实现
后台任务部分关键代码节选：
```csharp
public class FaceWhitelistWorker{
    private readonly MqttClientService _mqttService;
    private readonly ILogger _logger;
    private readonly IApiClient _api;
    private readonly IDataStore _dataStore;

    public FaceWhitelistWorker(MqttClientService mqttService,
        ILogger<FaceWhitelistWorker> logger,
        IApiClient api,
        IDataStore dataStore)
    {
        _mqttService = mqttService;
        _logger = logger;
        _api = api;
        _dataStore = dataStore;
    }
      public async Task StartAsync(CancellationToken cancellationToken)
    {
        await StartSchedule(cancellationToken);
        await StartPublish(cancellationToken);
        await StartClear(cancellationToken);
    }

    private Task StartClear(CancellationToken ct)
    {
        const int retryInterval = 3000;
        return Task.Factory.StartNew(async () =>
        {
            while (true)
            {
                if (ct.IsCancellationRequested) return;
                var count = _dataStore.DeleteExpiredMessages();
                if (count > 0)  _logger.LogDebug("Cleared {Count} messages", count);

                // 略：标记业务数据结果
                await Task.Delay(retryInterval, ct);
            }
        }, ct, TaskCreationOptions.LongRunning, TaskScheduler.Default);
    }

    private Task StartPublish(CancellationToken ct)
    {
        const int retryInterval = 500;
        return Task.Factory.StartNew(async () =>
        {
            await Task.Delay(5000, ct);
            while (true)
            {
                if (ct.IsCancellationRequested) return;
                if (_mqttService.MqttClient is not { IsConnected: true })
                {
                    await Task.Delay(5000, ct);
                    continue;
                }

                var mediumMessages = _dataStore.GetMessagesOfNeedRetry();
                if (mediumMessages == null || !mediumMessages.Any())
                {
                    await Task.Delay(retryInterval, ct);
                    continue;
                }

                foreach (var mediumMessage in mediumMessages)
                {
                    if (_dataStore.ExistPublishedTopic(mediumMessage.Topic))
                    {
                        _logger.LogDebug("Skip message {Id} with topic {Topic}", mediumMessage.MessageId,
                            mediumMessage.Topic);
                    }
                    else
                    {
                        var mqttMessage = new MqttApplicationMessageBuilder()
                            .WithTopic(mediumMessage.Topic)
                            .WithPayload(mediumMessage.Message)
                            .WithAtMostOnceQoS().Build();
                        await _mqttService.PublishAsync(mqttMessage);
                        _dataStore.ChangePublishState(mediumMessage.MessageId);
                        _logger.LogDebug("Publish message {Id} with topic {Topic}", mediumMessage.MessageId,
                            mediumMessage.Topic);
                    }
                }

                await Task.Delay(retryInterval, ct);
            }
        }, ct, TaskCreationOptions.LongRunning, TaskScheduler.Default);
    }

    private Task StartSchedule(CancellationToken ct)
    {
        return Task.Factory.StartNew(async () =>
        {
            const int waitInterval = 5000;
            await Task.Delay(waitInterval, ct); // 延迟5s等待Mqtt连接
            while (true)
            {
                if (ct.IsCancellationRequested) return;
                if (_mqttService.MqttClient is not { IsConnected: true })
                {
                    await Task.Delay(waitInterval, ct); // 等待Mqtt连接
                    continue;
                }

                if (ct.IsCancellationRequested) return;

                if (!_dataStore.IsWhitelistEmpty())
                {
                    await Task.Delay(waitInterval, ct); // 等待处理现有白名单
                    continue;
                }

                Pageable<WhitelistTask> page = null;
                // 略：获取白名单任务，没次拉取数量根据实际情况而定
               
                if (page?.Data == null || page.Data.Count == 0)
                {
                    await Task.Delay(10000, ct); // 无白名单任务，等待10s
                }
                else
                {
                    _logger.LogDebug("Fetched whitelist tasks from api, count:{Count}", page.Data.Count());
                    foreach (var task in page.Data)
                    {
                        if (ct.IsCancellationRequested) return;
                        if (task.Action == WhitelistTaskAction.Delete)
                        {
                            await ProcessDeleteAction(task);
                        }
                        else if (task.Action == WhitelistTaskAction.Add)
                        {
                            await ProcessAddAction(task);
                        }
                    }
                }
            }
        }, ct, TaskCreationOptions.LongRunning, TaskScheduler.Default);
    }

}
```

通过代码我们可以看到，核心定时任务包含3个，`StartSchedule`、`StartPublish`以及`StartClear`，其中`StartSchedule`负责拉取白名单任务，将所有消息压入本地消息表;`StartPublish`负责处理本地消息表的内容，该发送就发送，该重试就重试；而`StartClear`负责清理多次重试后仍然失败的消息。

最后还有一个问题，我们前面说到，终端设备每次只能处理一条任务，我们是怎么控制输出流量，防止消息堵塞的呢？这在`InMemoryDataStore.GetMessagesOfNeedRetry`方法里可以看到，我们对于同一个设备，只输出第一条消息，通过不断得轮询，最终完成所有消息分发。

```csharp
public IEnumerable<MediumMessage> GetMessagesOfNeedRetry()
{
    var result = WhitelistMessages.Values
        .Where(x => x.Tries < MaxTryCount &&
                    (x.StatusName is StatusName.Scheduled or StatusName.Failed ||
                     x.StatusName == StatusName.Published &&
                     x.PublishAt?.AddMilliseconds(RetryInternal) <= DateTime.Now))
        .OrderBy(t => t.AddAt)
        .GroupBy(t => t.Topic)
        .Select(t => t.First());
    return result;
}
```

## 输出结果
```
[12:19:54 DBG] Fetched whitelist tasks from api, count:5
[12:19:54 DBG] Publish message EditPerson-951-1494527067538440192 with topic dev/face/2042253
[12:19:54 DBG] Publish message EditPerson-965-1494527067840430080 with topic dev/face/11111
[12:19:55 INF] Mqtt message received with topic: dev/face/2042253/Ack
[12:19:55 WRN] Cannot find message handler for topic: dev/face/2042253/Ack
[12:19:55 DBG] Skip message EditPerson-954-1494527067689435136 with topic dev/face/2042253
[12:19:55 DBG] Skip message EditPerson-968-1494527068167585792 with topic dev/face/11111
[12:19:55 DBG] Skip message EditPerson-954-1494527067689435136 with topic dev/face/2042253
[12:19:55 DBG] Skip message EditPerson-968-1494527068167585792 with topic dev/face/11111
[12:19:56 DBG] Skip message EditPerson-954-1494527067689435136 with topic dev/face/2042253
[12:19:56 DBG] Skip message EditPerson-968-1494527068167585792 with topic dev/face/11111
[12:19:56 DBG] Publish message EditPerson-951-1494527067538440192 with topic dev/face/2042253
[12:19:56 DBG] Publish message EditPerson-965-1494527067840430080 with topic dev/face/11111
[12:19:57 DBG] Skip message EditPerson-954-1494527067689435136 with topic dev/face/2042253
[12:19:57 DBG] Skip message EditPerson-968-1494527068167585792 with topic dev/face/11111
[12:19:57 INF] Mqtt message received with topic: dev/face/2042253/Ack
[12:19:57 WRN] Cannot find message handler for topic: dev/face/2042253/Ack
[12:19:57 DBG] Skip message EditPerson-954-1494527067689435136 with topic dev/face/2042253
[12:19:57 DBG] Skip message EditPerson-968-1494527068167585792 with topic dev/face/11111
[12:19:58 DBG] Skip message EditPerson-954-1494527067689435136 with topic dev/face/2042253
[12:19:58 DBG] Skip message EditPerson-968-1494527068167585792 with topic dev/face/11111
[12:19:58 DBG] Publish message EditPerson-951-1494527067538440192 with topic dev/face/2042253
[12:19:59 DBG] Publish message EditPerson-965-1494527067840430080 with topic dev/face/11111
[12:19:59 INF] Mqtt message received with topic: dev/face/2042253/heartbeat
[12:19:59 DBG] Skip message EditPerson-954-1494527067689435136 with topic dev/face/2042253
[12:19:59 DBG] Skip message EditPerson-968-1494527068167585792 with topic dev/face/11111
[12:19:59 INF] Mqtt message received with topic: dev/face/2042253/Ack
[12:19:59 WRN] Cannot find message handler for topic: dev/face/2042253/Ack
[12:20:00 DBG] Skip message EditPerson-954-1494527067689435136 with topic dev/face/2042253
[12:20:00 DBG] Skip message EditPerson-968-1494527068167585792 with topic dev/face/11111
[12:20:00 DBG] Skip message EditPerson-954-1494527067689435136 with topic dev/face/2042253
[12:20:00 DBG] Skip message EditPerson-968-1494527068167585792 with topic dev/face/11111
[12:20:01 DBG] Publish message EditPerson-954-1494527067689435136 with topic dev/face/2042253
[12:20:01 DBG] Publish message EditPerson-968-1494527068167585792 with topic dev/face/11111
[12:20:01 DBG] Skip message EditPerson-957-1494527067722989568 with topic dev/face/2042253
[12:20:01 INF] Mqtt message received with topic: dev/face/2042253/Ack
[12:20:01 WRN] Cannot find message handler for topic: dev/face/2042253/Ack
[12:20:02 DBG] Skip message EditPerson-957-1494527067722989568 with topic dev/face/2042253
[12:20:02 DBG] Skip message EditPerson-957-1494527067722989568 with topic dev/face/2042253
[12:20:03 DBG] Publish message EditPerson-954-1494527067689435136 with topic dev/face/2042253
[12:20:03 DBG] Publish message EditPerson-968-1494527068167585792 with topic dev/face/11111
[12:20:03 DBG] Skip message EditPerson-957-1494527067722989568 with topic dev/face/2042253
[12:20:03 INF] Mqtt message received with topic: dev/face/2042253/Ack
[12:20:03 WRN] Cannot find message handler for topic: dev/face/2042253/Ack
[12:20:03 DBG] Cleared 2 messages
[12:20:04 DBG] Skip message EditPerson-957-1494527067722989568 with topic dev/face/2042253
[12:20:04 DBG] Skip message EditPerson-957-1494527067722989568 with topic dev/face/2042253
[12:20:05 DBG] Publish message EditPerson-954-1494527067689435136 with topic dev/face/2042253
[12:20:05 DBG] Publish message EditPerson-968-1494527068167585792 with topic dev/face/11111
[12:20:05 INF] Mqtt message received with topic: dev/face/2042253/Ack
[12:20:05 WRN] Cannot find message handler for topic: dev/face/2042253/Ack
[12:20:05 DBG] Skip message EditPerson-957-1494527067722989568 with topic dev/face/2042253
[12:20:06 DBG] Skip message EditPerson-957-1494527067722989568 with topic dev/face/2042253
[12:20:06 DBG] Skip message EditPerson-957-1494527067722989568 with topic dev/face/2042253
[12:20:07 DBG] Publish message EditPerson-957-1494527067722989568 with topic dev/face/2042253
[12:20:07 INF] Mqtt message received with topic: dev/face/2042253/Ack
[12:20:07 WRN] Cannot find message handler for topic: dev/face/2042253/Ack
[12:20:09 DBG] Publish message EditPerson-957-1494527067722989568 with topic dev/face/2042253
[12:20:09 INF] Mqtt message received with topic: dev/face/2042253/Ack
[12:20:09 WRN] Cannot find message handler for topic: dev/face/2042253/Ack
[12:20:09 DBG] Cleared 2 messages
[12:20:11 DBG] Publish message EditPerson-957-1494527067722989568 with topic dev/face/2042253
[12:20:11 INF] Mqtt message received with topic: dev/face/2042253/Ack
[12:20:11 WRN] Cannot find message handler for topic: dev/face/2042253/Ack
[12:20:16 DBG] Cleared 1 messages
[12:20:19 INF] Mqtt message received with topic: dev/face/2042253/heartbeat
[12:20:19 DBG] Fetched whitelist tasks from api, count:5
```

我们在后台创建了包含两个设备的5个白名单任务，通过日志我们可以发现，正好是完成了一轮的任务处理。另外，实例代码不是最终代码，开发中我们仍然要根据实际情况和机器配置做好异常处理，调试轮询的时间间隔，设定每次处理的任务数量，以及论证是否要将内存数据库改为其他数据库。同时，示例中并没有标记任务成功的动作，我们是按照所有任务均执行失败的情况演示的。
