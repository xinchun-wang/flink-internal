### Flink DataStream的主要转换关系
Flink为流和批操作分别提供了DataStream API和DataSet API，方便了业务代码的开发。其中DataStream的API可以通过Operator进行各种Stream的转换和操作。下图展示了DataStream的主要Operator之间的转换关系图。
```plantuml
@startuml 
[*] --> DataStream
DataStream --> DataStream : union(DataStream)\nbroadcast()\nshuffle()\nforward()\nrebalance()\n...
DataStream --> SplitSteam : split(OutputSelector)
SplitSteam --> DataStream : select(...)
DataStream --> KeyedStream : keyBy(...)
KeyedStream --> SingleOutputStreamOperator : transform()\nprocess()\nsum(...)\nmin(...)\nmax(...)\nmaxBy()\nminBy()\n...
KeyedStream --> WindowedStream : window(...)\ntimewindow(...)\ncountWindow(...)
WindowedStream --> SingleOutputStreamOperator : reduce(...)
DataStream --> SingleOutputStreamOperator : map()\nflatMap()\nprocess(...)\nfilter()\nproject()\nassignTimestamps\nAndWatermarks()\n...
DataStream --> AllWindowedStream : timeWindowAll(...)\ncountWindowAll(...)
AllWindowedStream --> SingleOutputStreamOperator : reduce(...)
DataStream --> JoinedStreams : join(DataStream)
JoinedStreams --> DataStream :  window(...).apply(...)
DataStream --> ConnectedStreams : connect(DataStream)
ConnectedStreams --> SingleOutputStreamOperator : map()\nflatMap()\nprocess()
SingleOutputStreamOperator --> DataStream : getSideOutput()
DataStream --> DataStreamSink : print()\naddSink()\nwriteAsXXX()
KeyedStream --> DataStreamSink : addSink()
DataStreamSink --> [*]
@enduml
```
### DataStream的类结构
DataStream是Flink流的核心基础API，表示了有相同数据类型的持续不断的一系列元素的组合。一种类型DataStream可以通过不同的操作转变为另外一种类型的DataStream，如map、flatMap等操作等。DataStream的主要类结构如下所示：
![Class of DataStream](images/DataStream_class_diagram.png)

一个 DataStream可以从StreamExecutionEnvironment里面的若干方法获得
~~~java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
DataStream<T> input = env.addSource(SourceFunction);
/**其他方法包括：
* input = env.fromElements(T);
* input = env.createInput(InputFormat);
*/

~~~
DataStream上的转换操作都是逐条的，比如 map()、flatMap()、filter()等，以flatMap方法为例，说明一下DataStream内部是如何对输入的元素进行操作和处理的。

```plantuml
@startuml 
StreamExecutionEnvironment -> DataStreamSource: addSource
activate DataStreamSource
group flatMap
DataStreamSource -> SingleOutputStreamOperator: transform
activate SingleOutputStreamOperator
SingleOutputStreamOperator -> OneInputTransformation: new
activate OneInputTransformation
OneInputTransformation -> StreamFlatMap: getOperator
StreamFlatMap -> StreamFlatMap : processElement
StreamFlatMap -> StreamFlatMap : processWatermark
end
@enduml
```

#### SingleOutputStreamOperator核心方法解析