# 启动JobManager

启动JobManager进程，我们需要道${FLINK_HOME}/bin目录运行jobmanager.sh脚本，其用法如下：
~~~bash
jobmanager.sh ((start|start-foreground) [host] [webui-port])|stop|stop-all
~~~
执行jobmanager.sh的时候，默认会先执行bin/config.sh的内容。config.sh会解析conf/flink-conf.yaml配置文件里面的各个配置项，主要包括JobManager或者TaskManager的堆大小、环境变量设置、日志文件路径、Hadoop配置文件路径、HA设置等等。默认下情况下，在flink-conf.yaml中没有设置key为mode的配置项，FLINK_MODE在config.sh中会被设置为“new”模式，因此启动的JobManager类型为standalonesession。
~~~bash
. "$bin"/config.sh

JOBMANAGER_TYPE=jobmanager

if [[ "${FLINK_MODE}" == "new" ]]; then
    JOBMANAGER_TYPE=standalonesession
fi
~~~
其中start模式则启动为deamon方式，执行flink-deamon.sh运行；而start-foreground则启动为控制台方式，执行flink-console.sh运行。
~~~bash
if [[ $STARTSTOP == "start-foreground" ]]; then
    exec "${FLINK_BIN_DIR}"/flink-console.sh $JOBMANAGER_TYPE "${args[@]}"
else
    "${FLINK_BIN_DIR}"/flink-daemon.sh $STARTSTOP $JOBMANAGER_TYPE "${args[@]}"
fi
~~~
不管执行的那种模式，standalonesession模式运行的主函数均为
~~~bash
CLASS_TO_RUN=org.apache.flink.runtime.entrypoint.StandaloneSessionClusterEntrypoint
~~~
其中控制台模式使用的日志配置文件为：${FLINK_CONF_DIR}/log4j-console.properties或者logback-console.xml；而deamon模式使用的是${FLINK_CONF_DIR}/log4j.properties或者logback.xml。默认情况下，使用的是log4j进行日志输出。不管是jobmanager.sh还是flink-deamon.sh等脚本，都是为了初始化相关JVM参数、环境变量等，真正运行的逻辑还是在StandaloneSessionClusterEntrypoint中完成的。
