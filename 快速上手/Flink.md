flink checkpoint，就是快照记录的点，在触发快照的时候，会从源打入barrier，后面的每个环节看到这个barrier就会进行快照，以后出问题就可以从这个地方恢复。
checkpoint可以保证exactly once
https://www.jianshu.com/p/4d31d6cddc99

flink window和watermark
eventTime：数据的发生事件时间
window：flink划分流式数据处理的窗口，例如每5秒一个窗口进行计算
watermark：表示eventTime需要大于watermark之后，才开始对应的window计算，确保迟到的数据都能参与计算，例如watermark10秒，那最新数据到来后，才计算10秒前那个-15～-10那个窗口。
AssignerWithPeriodicWatermarks：定时打入watermark
AssignerWithPunctuatedWatermarks：每次有新数据都更新watermark，更耗资源但更准确
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime) 设置数据流处理时间特征是以数据的发生事件时间来计算，默认值是TimeCharacteristic.ProcessingTime数据处理时间，默认水位线更新是每隔200ms
事件时间(event time)： 事件产生的时间，记录的是设备生产(或者存储)事件的时间
摄取时间(ingestion time)： Flink 读取事件时记录的时间
处理时间(processing time)： Flink pipeline 中具体算子处理事件的时间
sideOutputLateData设置如果超过watermark之后才到达的旧数据怎样处理，默认行为是丢弃，可以通过这个api设置把超时数据保存到单独地方再处理。
getSideOutput获取过期数据

DataStream API https://ci.apache.org/projects/flink/flink-docs-master/zh/docs/try-flink/datastream/
