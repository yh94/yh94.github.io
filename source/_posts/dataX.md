---
title: dataX剖析
date: 2022-06-10
tags:
- java
- 大数据
categories:
- java
- 数据同步
---
整个流程大致如下  
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200825180616261.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3ljaGFuZzU3Nw==,size_16,color_FFFFFF,t_70#pic_center)  
**先看下官方的介绍，了解下功能和结构。再进行源码的剖析**  
DataX 是一个异构数据源离线同步工具，致力于实现包括关系型数据库(MySQL、Oracle等)、HDFS、Hive、ODPS、HBase、FTP等各种异构数据源之间稳定高效的数据同步功能  
![在这里插入图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9jbG91ZC5naXRodWJ1c2VyY29udGVudC5jb20vYXNzZXRzLzEwNjcxNzUvMTc4Nzk4NDEvOTNiN2ZjMWMtNjkyNy0xMWU2LThjZGEtN2NmODQyMGZjNjVmLnBuZw?x-oss-process=image/format,png#pic_center)

DataX本身作为离线数据同步框架，采用Framework + plugin架构构建。将数据源读取和写入抽象成为Reader/Writer插件，纳入到整个同步框架中。

Reader：Reader为数据采集模块，负责采集数据源的数据，将数据发送给Framework。

Writer： Writer为数据写入模块，负责不断向Framework取数据，并将数据写入到目的端。

Framework：Framework用于连接reader和writer，作为两者的数据传输通道，并处理缓冲，流控，并发，数据转换等核心技术问题。

![在这里插入图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9jbG91ZC5naXRodWJ1c2VyY29udGVudC5jb20vYXNzZXRzLzEwNjcxNzUvMTc4Nzk4ODQvZWM3ZTM2ZjQtNjkyNy0xMWU2LThmNWYtZmZjNDNkNmE0NjhiLnBuZw?x-oss-process=image/format,png#pic_center)  
DataX3.0核心架构  
DataX 3.0 开源版本支持单机多线程模式完成同步作业运行，按一个DataX作业生命周期的时序图，从整体架构设计非常简要说明DataX各个模块相互关系  
![在这里插入图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9jbG91ZC5naXRodWJ1c2VyY29udGVudC5jb20vYXNzZXRzLzEwNjcxNzUvMTc4NTA4NDkvYWE2Yzk1YTgtNjg5MS0xMWU2LTk0YjctMzlmMGFiNWFmM2I0LnBuZw?x-oss-process=image/format,png#pic_center)  
核心模块  
DataX完成单个数据同步的作业，我们称之为Job，DataX接受到一个Job之后，将启动一个进程来完成整个作业同步过程。DataX Job模块是单个作业的中枢管理节点，承担了数据清理、子任务切分(将单一作业计算转化为多个子Task)、TaskGroup管理等功能。

DataXJob启动后，会根据不同的源端切分策略，将Job切分成多个小的Task(子任务)，以便于并发执行。Task便是DataX作业的最小单元，每一个Task都会负责一部分数据的同步工作。

切分多个Task之后，DataX Job会调用Scheduler模块，根据配置的并发数据量，将拆分成的Task重新组合，组装成TaskGroup(任务组)。每一个TaskGroup负责以一定的并发运行完毕分配好的所有Task，默认单个任务组的并发数量为5。

每一个Task都由TaskGroup负责启动，Task启动后，会固定启动Reader—>Channel—>Writer的线程来完成任务同步工作。

DataX作业运行起来之后， Job监控并等待多个TaskGroup模块任务完成，等待所有TaskGroup任务完成后Job成功退出。否则，异常退出，进程退出值非0

DataX任务切分  
例如用户提交了一个DataX作业，并且配置了10个并发，目的是将一个100张分表的mysql数据同步到目标库。 任务切分过程：

DataXJob根据分库分表切分成了100个Task。

根据10个并发，每个TaskGroup默认执行5个task(TaskGroup执行task数量可以设置)DataX计算共需要分配2个TaskGroup。

2个TaskGroup平分切分好的100个Task，每一个TaskGroup负责以5个并发共计运行50个Task

[](https://blog.csdn.net/ychang577/article/details/108226450)源码剖析
-----------------------------------------------------------------

任务执行的入口类为Engine

```
public static void main(String[] args) throws Exception {
    /**
     * 入参参数
     * -mode standalone
     * -jobid -1
     * -job E:\\workspace\\datax\\job\\job1.json
     */
    String[] param = {"-mode", "standalone", "-jobid", "-1", "-job", "E:\\workspace\\datax\\job\\job1.json"};
    System.setProperty("datax.home", "E:\\workspace\\DataX-master\\core\\src\\main");
    args = param;
    int exitCode = 0;
    try {
        Engine.entry(args);
    } catch (Throwable e) {
    }
}

```

**Job的核心配置**主要包括三个配置

> core.json DataX核心配置

> job.json 本次任务配置

> plugin.json 本次任务使用到read , write 插件配置

Engine#entry

1、解析命令行参数 -mode -jobid -job 获取jobid, 和job配置文件路径，执行模式(standalone, local, Distrubuted)

2、解析本次任务配置，创建新的Engine,执行start()方法

```
public static void entry(final String[] args) throws Throwable {
    Options options = new Options();
    options.addOption("job", true, "Job config.");
    options.addOption("jobid", true, "Job unique id.");
    options.addOption("mode", true, "Job runtime mode.");
    BasicParser parser = new BasicParser();
    CommandLine cl = parser.parse(options, args);   //解析命令参数
    String jobPath = cl.getOptionValue("job");
    // 如果用户没有明确指定jobid, 则 datax.py 会指定 jobid 默认值为-1
    String jobIdString = cl.getOptionValue("jobid");
    RUNTIME_MODE = cl.getOptionValue("mode");
    /**
        * 解析本次job配置
         * Configuration 包括3部分
         * 1、job.json   任务配置
         * 2. core.json  DataX配置
         * 3. plugin.json 插件配置,例如插件类路径...
         */
    Configuration configuration = ConfigParser.parse(jobPath);
    long jobId;
    if (!"-1".equalsIgnoreCase(jobIdString)) {
        jobId = Long.parseLong(jobIdString);
    } else {
        // only for dsc & ds & datax 3 update
        String dscJobUrlPatternString = "/instance/(\\d{1,})/config.xml";
        String dsJobUrlPatternString = "/inner/job/(\\d{1,})/config";
        String dsTaskGroupUrlPatternString = "/inner/job/(\\d{1,})/taskGroup/";
        List<String> patternStringList = Arrays.asList(dscJobUrlPatternString,
                dsJobUrlPatternString, dsTaskGroupUrlPatternString);
        jobId = parseJobIdFromUrl(patternStringList, jobPath);
    }
    boolean isStandAloneMode = "standalone".equalsIgnoreCase(RUNTIME_MODE);
    if (!isStandAloneMode && jobId == -1) {
        // 如果不是 standalone 模式，那么 jobId 一定不能为-1
        throw DataXException.asDataXException(FrameworkErrorCode.CONFIG_ERROR, "非 standalone 模式必须在 URL 中提供有效的 jobId.");
    }
    configuration.set(CoreConstant.DATAX_CORE_CONTAINER_JOB_ID, jobId);
    //打印vmInfo
    VMInfo vmInfo = VMInfo.getVmInfo();
    if (vmInfo != null) {
        LOG.info(vmInfo.toString());
    }
    LOG.info("\n" + Engine.filterJobConfiguration(configuration) + "\n");
    LOG.debug(configuration.toJSON());
    ConfigurationValidate.doValidate(configuration);
    Engine engine = new Engine();
    //通过配置创建一个JobContainer对象.执行start方法
    engine.start(configuration);
}

```

Engine#start

1、首先绑定下列转换信息，比如时间格式，时区，编码等,在core.json中的common.column配置项

2、设置插件配置

3、创建JobContainer ,并且启动

```
public void start(Configuration allConf) {
    // 绑定column转换信息 时间格式，时区，编码等,在core.json中的common.column配置项
    ColumnCast.bind(allConf);   
    //初始化PluginLoader，可以获取各种插件配置
    LoadUtil.bind(allConf);
    //core.container.model 容器模式，默认是job
    boolean isJob = !("taskGroup".equalsIgnoreCase(allConf
            .getString(CoreConstant.DATAX_CORE_CONTAINER_MODEL)));
    //JobContainer会在schedule后再行进行设置和调整值
    int channelNumber =0;
    AbstractContainer container;
    long instanceId;
    int taskGroupId = -1;
    if (isJob) {
        allConf.set(CoreConstant.DATAX_CORE_CONTAINER_JOB_MODE, RUNTIME_MODE);
        container = new JobContainer(allConf);
        instanceId = allConf.getLong(
                CoreConstant.DATAX_CORE_CONTAINER_JOB_ID, 0);
​
    } else {
        container = new TaskGroupContainer(allConf);
        instanceId = allConf.getLong(
                CoreConstant.DATAX_CORE_CONTAINER_JOB_ID);
        taskGroupId = allConf.getInt(
                CoreConstant.DATAX_CORE_CONTAINER_TASKGROUP_ID);
        channelNumber = allConf.getInt(
                CoreConstant.DATAX_CORE_CONTAINER_TASKGROUP_CHANNEL);
    }
    //缺省打开perfTrace
    boolean traceEnable = allConf.getBool(CoreConstant.DATAX_CORE_CONTAINER_TRACE_ENABLE, true);
    boolean perfReportEnable = allConf.getBool(CoreConstant.DATAX_CORE_REPORT_DATAX_PERFLOG, true);
    //standlone模式的datax shell任务不进行汇报
    if(instanceId == -1){
        perfReportEnable = false;
    }
    int priority = 0;
    try {
        priority = Integer.parseInt(System.getenv("SKYNET_PRIORITY"));
    }catch (NumberFormatException e){
        LOG.warn("prioriy set to 0, because NumberFormatException, the value is: "+System.getProperty("PROIORY"));
    }
    Configuration jobInfoConfig = allConf.getConfiguration(CoreConstant.DATAX_JOB_JOBINFO);
    //初始化PerfTrace
    PerfTrace perfTrace = PerfTrace.getInstance(isJob, instanceId, taskGroupId, priority, traceEnable);
    perfTrace.setJobInfo(jobInfoConfig,perfReportEnable,channelNumber);
    /**
     * JobContainer.start方法是整个框架核心,
     * 依次执行job的#preHandler,#init,#prepare,#split,#schedule,#post,#postHandler方法
     * 最重要的是#init,#split,#schedule
     */
    container.start();
}

```

JobContainer#start

任务容器启动器

preHandler 前置操作，加载job插件等 . (未使用到)

init 初始化read , write 插件。 这个过程会对数据源，表，列进行校验

initJobWriter

```
private void init() {
    //..省略部分代码
    //必须先Reader ，后Writer
    this.jobReader = this.initJobReader(jobPluginCollector);
    this.jobWriter = this.initJobWriter(jobPluginCollector);
}

```

initJobReader、initJobWriter 初始化插件中，使用了URLCloassLoader对插件类进行加载。解决了类冲突的问题。因此用户可以使用自己的类/jar包自定义插件。

jobReader.init()

```
private Reader.Job initJobReader(
        JobPluginCollector jobPluginCollector) {
    //获取读插件名称
    this.readerPluginName = this.configuration.getString(
            CoreConstant.DATAX_JOB_CONTENT_READER_NAME);    //job.content[0].reader.name
    //根据读插件类名称,加载插件的lib包加载到jvm中
    classLoaderSwapper.setCurrentThreadClassLoader(LoadUtil.getJarLoader(
            PluginType.READER, this.readerPluginName)); //重置插件jar classLoader
    //创建一个读对象
    Reader.Job jobReader = (Reader.Job) LoadUtil.loadJobPlugin(
            PluginType.READER, this.readerPluginName);
    // 设置reader的jobConfig
    jobReader.setPluginJobConf(this.configuration.getConfiguration(
            CoreConstant.DATAX_JOB_CONTENT_READER_PARAMETER));
    // 设置reader的readerConfig
    jobReader.setPeerPluginJobConf(this.configuration.getConfiguration(
            CoreConstant.DATAX_JOB_CONTENT_WRITER_PARAMETER));
    jobReader.setJobPluginCollector(jobPluginCollector);
    jobReader.init();   //读插件初始化(可以见具体读插件实现类,例如MysqlReader)
    //重置回归原classLoader
    classLoaderSwapper.restoreCurrentThreadClassLoader();
    return jobReader;
}

```

​  
以MysqlReader为例jobReader#init

```
public static class Job extends Reader.Job {
    ///...省略其他代码
    @Override
    public void init() {
        this.originalConfig = super.getPluginJobConf(); //Job配置
        Integer userConfigedFetchSize = this.originalConfig.getInt(Constant.FETCH_SIZE);
        if (userConfigedFetchSize != null) {
            LOG.warn("对 mysqlreader 不需要配置 fetchSize, mysqlreader 将会忽略这项配置. 如果您不想再看到此警告,请去除fetchSize 配置.");
        }
        this.originalConfig.set(Constant.FETCH_SIZE, Integer.MIN_VALUE);
        this.commonRdbmsReaderJob = new CommonRdbmsReader.Job(DATABASE_TYPE);
        this.commonRdbmsReaderJob.init(this.originalConfig);
    }
    ///...省略其他代码
}

```

以MysqlWriter为例

jobWriter.init();

```
private Writer.Job initJobWriter(
        JobPluginCollector jobPluginCollector) {
    this.writerPluginName = this.configuration.getString(
            CoreConstant.DATAX_JOB_CONTENT_WRITER_NAME);
    classLoaderSwapper.setCurrentThreadClassLoader(LoadUtil.getJarLoader(
            PluginType.WRITER, this.writerPluginName));
    Writer.Job jobWriter = (Writer.Job) LoadUtil.loadJobPlugin(
            PluginType.WRITER, this.writerPluginName);
    //...省略部分代码
    jobWriter.init();
    classLoaderSwapper.restoreCurrentThreadClassLoader();
    return jobWriter;
}

```

jobWriter#init

```
public static class Job extends Writer.Job {
    //...省略部分代码
    @Override
    public void init() {
        this.originalConfig = super.getPluginJobConf();
        this.commonRdbmsWriterJob = new CommonRdbmsWriter.Job(DATABASE_TYPE);
        this.commonRdbmsWriterJob.init(this.originalConfig);
    }
}

```

prepare read, write插件做一些前置操作

实现为具体插件类的prepare方法

split 任务拆分

schedule 首先完成的工作是把上一步reader和writer split的结果整合到具体taskGroupContainer中,同时不同的执行模式调用不同的调度策略，将所有任务调度起来

```
/**
 * 执行reader和writer最细粒度的切分，需要注意的是，writer的切分结果要参照reader的切分结果，
 * 达到切分后数目相等，才能满足1：1的通道模型，所以这里可以将reader和writer的配置整合到一起，
 * 然后，为避免顺序给读写端带来长尾影响，将整合的结果shuffler掉
 */

private int split() {
    //设置channel数量
    this.adjustChannelNumber();
    if (this.needChannelNumber <= 0) {
        this.needChannelNumber = 1;
    }
    List<Configuration> readerTaskConfigs = this
            .doReaderSplit(this.needChannelNumber);
    int taskNumber = readerTaskConfigs.size();
    List<Configuration> writerTaskConfigs = this
            .doWriterSplit(taskNumber);
    //job.content[0].transformer
    List<Configuration> transformerList = this.configuration.getListConfiguration(CoreConstant.DATAX_JOB_CONTENT_TRANSFORMER);
    //合并读任务配置、写任务配置、transformer配置
    //输入是reader和writer的parameter list，输出是content下面元素的list
    List<Configuration> contentConfig = mergeReaderAndWriterTaskConfigs(
            readerTaskConfigs, writerTaskConfigs, transformerList);
    //job.content 合并后的总配置
    this.configuration.set(CoreConstant.DATAX_JOB_CONTENT, contentConfig);
    return contentConfig.size();
}

```

设置管道数量 adjustChannelNumber()

1、配置 job.setting.speed.byte (jobByte)， 则必须设置 core.transport.channel.speed.byte (coreByte) ，channelNumber = coreByte / jobByte > 0 ? coreByte / jobByte : 1

2、配置job.setting.speed.record (jobRecord) , 则必须配置core.transport.channel.speed.record（coreRecord）

channelNumber = coreRecord/ jobRecord> 0 ? coreRecord/ jobRecord: 1

如果上面2个任意配置了一个或2个，取较小值为channelNumber ，如果没有配置。则必须要配置job.setting.speed.channel 作为管道数量

下面切分读任务和写任务，最后把读写任务配置合并
#doReaderSplit 读切分任务是read插件实现的
我这里使用的是mysqlreader 插件，最后还是引用的框架的split实现

```
public List<Configuration> split(int adviceNumber) {
    return this.commonRdbmsReaderJob.split(this.originalConfig, adviceNumber);
}

```

#doWriterSplit写切分任务
写切分任务数量是根据读切分后的数量而来，写任务必须与读任务数一致

```
private List<Configuration> doWriterSplit(int readerTaskNumber) {
    classLoaderSwapper.setCurrentThreadClassLoader(LoadUtil.getJarLoader(
            PluginType.WRITER, this.writerPluginName));
​
    List<Configuration> writerSlicesConfigs = this.jobWriter
            .split(readerTaskNumber);
    if (writerSlicesConfigs == null || writerSlicesConfigs.size() <= 0) {
        throw DataXException.asDataXException(
                FrameworkErrorCode.PLUGIN_SPLIT_ERROR,
                "writer切分的task不能小于等于0");
    }
    classLoaderSwapper.restoreCurrentThreadClassLoader();
    return writerSlicesConfigs;
}
public List<Configuration> split(int mandatoryNumber) {
    return this.commonRdbmsWriterJob.split(this.originalConfig, mandatoryNumber);
}

```

切分读任务：

1、table模式 ：当没有配置splitPk时，任务数量与table数量一样.比如table配置了2个(user\_info, user\_info\_1) 则生成2任务个配置

2、table模式 ：配置splitPk时，配合channel一起使用。任务数 = (向上取整)(channel/table数量) ,当任务数 > 1 .会重新切分任务

最终任务数 = 任务数 \* 5 + 1 。 （这里程序实际根据job配置的splitPk请求了数据库，并且把查询出的pk范围分别加入到每个任务的pk条件中去）。 例如user\_info表pk为user\_id. 最终的sql会加上 and user\_id > 下限 and user\_id < 上限.

3、querySql模式 ：有几条querySql ， 生成相同数量的任务配置

切分写任务

1、写单表的时，生成tabel表数量的任务 或者 querySql数量相等的任务

2、多表时，生成和表数量相同的任务

schedule执行任务  
1、首先拿到任务组执行任务数量配置，再结合切分后任务数量对任务进行分组

```
int channelsPerTaskGroup = this.configuration.getInt(
        CoreConstant.DATAX_CORE_CONTAINER_TASKGROUP_CHANNEL, 5);
int taskNumber = this.configuration.getList(
        CoreConstant.DATAX_JOB_CONTENT).size();
this.needChannelNumber = Math.min(this.needChannelNumber, taskNumber);

```

//通过获取配置信息得到每个taskGroup需要运行哪些tasks任务

```
List<Configuration> taskGroupConfigs = JobAssignUtil.assignFairly(this.configuration,
      this.needChannelNumber, channelsPerTaskGroup);

```

2、创建任务执行器

```
AbstractScheduler scheduler = initStandaloneScheduler(this.configuration);
private AbstractScheduler initStandaloneScheduler(Configuration configuration) {
        AbstractContainerCommunicator containerCommunicator = new StandAloneJobContainerCommunicator(configuration);
        super.setContainerCommunicator(containerCommunicator);
​
        return new StandAloneScheduler(containerCommunicator);
}

```

3、执行任务

scheduler.schedule(taskGroupConfigs); //schedule方法先获取收集信息的属性，比如说间隔多长时间汇报,休眠时间等  
//然后开启任务  
startAllTaskGroup(configurations); //重点，这里开启执行任务  
//后面代码是收集打印任务的汇报信息  
4、创建线程池，提交任务

创建了一个与任务数量大小一致的固定线程池，满足完成当前任务

```
public void startAllTaskGroup(List<Configuration> configurations) {
    //创建一个任务数量大小的固定线程池
    this.taskGroupContainerExecutorService = Executors
            .newFixedThreadPool(configurations.size());
    for (Configuration taskGroupConfiguration : configurations) {
        TaskGroupContainerRunner taskGroupContainerRunner = newTaskGroupContainerRunner(taskGroupConfiguration);
        this.taskGroupContainerExecutorService.execute(taskGroupContainerRunner);
    }
    this.taskGroupContainerExecutorService.shutdown();
}

```

5、容器组启动

TaskGroupContainer类为执行任务的承载容器，重点关注

```
public class TaskGroupContainerRunner implements Runnable {
   private TaskGroupContainer taskGroupContainer;
   private State state;
   public TaskGroupContainerRunner(TaskGroupContainer taskGroup) {
        this.taskGroupContainer = taskGroup;
        this.state = State.SUCCEEDED;
    }
   @Override
   public void run() {
      try {
            Thread.currentThread().setName(
                    String.format("taskGroup-%d", this.taskGroupContainer.getTaskGroupId()));
            this.taskGroupContainer.start();    //开始执行任务
         this.state = State.SUCCEEDED;
      } catch (Throwable e) {
         this.state = State.FAILED;
         throw DataXException.asDataXException(
               FrameworkErrorCode.RUNTIME_ERROR, e);
      }
   }
    //...
}

```

​  
TaskGroupContainer

```
public class TaskGroupContainer extends AbstractContainer {
    private static final Logger LOG = LoggerFactory
            .getLogger(TaskGroupContainer.class);
    //当前taskGroup所属jobId
    private long jobId;
    //当前taskGroupId
    private int taskGroupId;
    //使用的channel类
    private String channelClazz;
    //task收集器使用的类
    private String taskCollectorClass;
​
    private TaskMonitor taskMonitor = TaskMonitor.getInstance();
​
    public TaskGroupContainer(Configuration configuration) {
        super(configuration);
        initCommunicator(configuration);    //初始化通信器
        this.jobId = this.configuration.getLong(
                CoreConstant.DATAX_CORE_CONTAINER_JOB_ID);
        //core.container.taskGroup.id 任务组id
        this.taskGroupId = this.configuration.getInt(
                CoreConstant.DATAX_CORE_CONTAINER_TASKGROUP_ID);
        //管道实现类 core.transport.channel.class
        this.channelClazz = this.configuration.getString(
                CoreConstant.DATAX_CORE_TRANSPORT_CHANNEL_CLASS);
        //任务收集器 core.statistics.collector.plugin.taskClass
        this.taskCollectorClass = this.configuration.getString(
                CoreConstant.DATAX_CORE_STATISTICS_COLLECTOR_PLUGIN_TASKCLASS);
    }
    //...
    @Override
    public void start() {
        try {
            /**
             * 状态check时间间隔，较短，可以把任务及时分发到对应channel中
             * core.container.taskGroup.sleepInterval
             */
            int sleepIntervalInMillSec = this.configuration.getInt(
                    CoreConstant.DATAX_CORE_CONTAINER_TASKGROUP_SLEEPINTERVAL, 100);
            /**
             * 状态汇报时间间隔，稍长，避免大量汇报
             * core.container.taskGroup.reportInterval
             */
            long reportIntervalInMillSec = this.configuration.getLong(
                    CoreConstant.DATAX_CORE_CONTAINER_TASKGROUP_REPORTINTERVAL,
                    10000);
            /**
             * 2分钟汇报一次性能统计
             */
            //core.container.taskGroup.channel
            // 获取channel数目
            int channelNumber = this.configuration.getInt(
                    CoreConstant.DATAX_CORE_CONTAINER_TASKGROUP_CHANNEL);
            //最大重试次数 core.container.task.failOver.maxRetryTimes 默认1次
            int taskMaxRetryTimes = this.configuration.getInt(
                    CoreConstant.DATAX_CORE_CONTAINER_TASK_FAILOVER_MAXRETRYTIMES, 1);
            //任务组重试间隔时间 core.container.task.failOver.retryIntervalInMsec
            long taskRetryIntervalInMsec = this.configuration.getLong(
                    CoreConstant.DATAX_CORE_CONTAINER_TASK_FAILOVER_RETRYINTERVALINMSEC, 10000);
            //core.container.task.failOver.maxWaitInMsec
            long taskMaxWaitInMsec = this.configuration.getLong(CoreConstant.DATAX_CORE_CONTAINER_TASK_FAILOVER_MAXWAITINMSEC, 60000);
            //获取当前任务组所有任务配置
            List<Configuration> taskConfigs = this.configuration
                    .getListConfiguration(CoreConstant.DATAX_JOB_CONTENT);
            int taskCountInThisTaskGroup = taskConfigs.size();
            LOG.info(String.format(
                    "taskGroupId=[%d] start [%d] channels for [%d] tasks.",
                    this.taskGroupId, channelNumber, taskCountInThisTaskGroup));
            //任务组注册通信器
            this.containerCommunicator.registerCommunication(taskConfigs);
            //taskId与task配置
            Map<Integer, Configuration> taskConfigMap = buildTaskConfigMap(taskConfigs);
            List<Configuration> taskQueue = buildRemainTasks(taskConfigs); //待运行task列表
            Map<Integer, TaskExecutor> taskFailedExecutorMap = new HashMap<Integer, TaskExecutor>();            //taskId与上次失败实例
            List<TaskExecutor> runTasks = new ArrayList<TaskExecutor>(channelNumber); //正在运行task
            Map<Integer, Long> taskStartTimeMap = new HashMap<Integer, Long>(); //任务开始时间
            long lastReportTimeStamp = 0;
            Communication lastTaskGroupContainerCommunication = new Communication();
           //这里开始进入循环作业
            while (true) {
               //1.判断task状态
               boolean failedOrKilled = false;
               Map<Integer, Communication> communicationMap = containerCommunicator.getCommunicationMap();  //任务id对应通信器,用来收集任务作业情况
               for(Map.Entry<Integer, Communication> entry : communicationMap.entrySet()){
                  Integer taskId = entry.getKey();
                  Communication taskCommunication = entry.getValue();
                    if(!taskCommunication.isFinished()){
                        continue;   //当前任务未结束,继续执行
                    }
                    //已经结束的任务,从正在运行的任务集合中移除
                    TaskExecutor taskExecutor = removeTask(runTasks, taskId);
                    //上面从runTasks里移除了，因此对应在monitor里移除
                    taskMonitor.removeTask(taskId);
                    //失败，看task是否支持failover，重试次数未超过最大限制
                  if(taskCommunication.getState() == State.FAILED){
                        taskFailedExecutorMap.put(taskId, taskExecutor);
                     if(taskExecutor.supportFailOver() && taskExecutor.getAttemptCount() < taskMaxRetryTimes){
                            taskExecutor.shutdown(); //关闭老的executor
                            containerCommunicator.resetCommunication(taskId); //将task的状态重置
                        Configuration taskConfig = taskConfigMap.get(taskId);
                        taskQueue.add(taskConfig); //重新加入任务列表
                     }else{
                        failedOrKilled = true;
                         break;
                     }
                  }else if(taskCommunication.getState() == State.KILLED){
                     failedOrKilled = true;
                     break;
                  }else if(taskCommunication.getState() == State.SUCCEEDED){
                        Long taskStartTime = taskStartTimeMap.get(taskId);
                        if(taskStartTime != null){
                            Long usedTime = System.currentTimeMillis() - taskStartTime;
                            LOG.info("taskGroup[{}] taskId[{}] is successed, used[{}]ms",
                                    this.taskGroupId, taskId, usedTime);
                            //usedTime*1000*1000 转换成PerfRecord记录的ns，这里主要是简单登记，进行最长任务的打印。因此增加特定静态方法
                            PerfRecord.addPerfRecord(taskGroupId, taskId, PerfRecord.PHASE.TASK_TOTAL,taskStartTime, usedTime * 1000L * 1000L);
                            taskStartTimeMap.remove(taskId);
                            taskConfigMap.remove(taskId);
                        }
                    }
               }
               
                // 2.发现该taskGroup下taskExecutor的总状态失败则汇报错误
                if (failedOrKilled) {
                    lastTaskGroupContainerCommunication = reportTaskGroupCommunication(
                            lastTaskGroupContainerCommunication, taskCountInThisTaskGroup);
​
                    throw DataXException.asDataXException(
                            FrameworkErrorCode.PLUGIN_RUNTIME_ERROR, lastTaskGroupContainerCommunication.getThrowable());
                }
                
                //3.有任务未执行，且正在运行的任务数小于最大通道限制
                Iterator<Configuration> iterator = taskQueue.iterator();
                while(iterator.hasNext() && runTasks.size() < channelNumber){
                    Configuration taskConfig = iterator.next();
                    Integer taskId = taskConfig.getInt(CoreConstant.TASK_ID);
                    int attemptCount = 1;
                    TaskExecutor lastExecutor = taskFailedExecutorMap.get(taskId);
                    if(lastExecutor!=null){
                        attemptCount = lastExecutor.getAttemptCount() + 1;
                        long now = System.currentTimeMillis();
                        long failedTime = lastExecutor.getTimeStamp();
                        if(now - failedTime < taskRetryIntervalInMsec){  //未到等待时间，继续留在队列
                            continue;
                        }
                        if(!lastExecutor.isShutdown()){ //上次失败的task仍未结束
                            if(now - failedTime > taskMaxWaitInMsec){
                                markCommunicationFailed(taskId);
                                reportTaskGroupCommunication(lastTaskGroupContainerCommunication, taskCountInThisTaskGroup);
                                throw DataXException.asDataXException(CommonErrorCode.WAIT_TIME_EXCEED, "task failover等待超时");
                            }else{
                                lastExecutor.shutdown(); //再次尝试关闭
                                continue;
                            }
                        }else{
                            LOG.info("taskGroup[{}] taskId[{}] attemptCount[{}] has already shutdown",
                                    this.taskGroupId, taskId, lastExecutor.getAttemptCount());
                        }
                    }
                    Configuration taskConfigForRun = taskMaxRetryTimes > 1 ? taskConfig.clone() : taskConfig;
                   TaskExecutor taskExecutor = new TaskExecutor(taskConfigForRun, attemptCount);
                    taskStartTimeMap.put(taskId, System.currentTimeMillis());
                   taskExecutor.doStart();
​
                    iterator.remove();
                    runTasks.add(taskExecutor); //继续添加到运行的任务集合
                    //上面，增加task到runTasks列表，因此在monitor里注册。
                    taskMonitor.registerTask(taskId, this.containerCommunicator.getCommunication(taskId));
                  //刚刚已经添加了task，这里把任务id从失败map移除
                    taskFailedExecutorMap.remove(taskId);
                    LOG.info("taskGroup[{}] taskId[{}] attemptCount[{}] is started",
                            this.taskGroupId, taskId, attemptCount);
                }
​
                //4.任务列表为空，executor已结束, 搜集状态为success--->成功
                if (taskQueue.isEmpty() && isAllTaskDone(runTasks) && containerCommunicator.collectState() == State.SUCCEEDED) {
                   // 成功的情况下，也需要汇报一次。否则在任务结束非常快的情况下，采集的信息将会不准确
                    lastTaskGroupContainerCommunication = reportTaskGroupCommunication(
                            lastTaskGroupContainerCommunication, taskCountInThisTaskGroup);
                    LOG.info("taskGroup[{}] completed it's tasks.", this.taskGroupId);
                    break;
                }
                // 5.如果当前时间已经超出汇报时间的interval，那么我们需要马上汇报
                long now = System.currentTimeMillis();
                if (now - lastReportTimeStamp > reportIntervalInMillSec) {
                    lastTaskGroupContainerCommunication = reportTaskGroupCommunication(
                            lastTaskGroupContainerCommunication, taskCountInThisTaskGroup);
                    lastReportTimeStamp = now;
                    //taskMonitor对于正在运行的task，每reportIntervalInMillSec进行检查
                    for(TaskExecutor taskExecutor:runTasks){   taskMonitor.report(taskExecutor.getTaskId(),this.containerCommunicator.getCommunication(taskExecutor.getTaskId()));
                    }
​
                }
                Thread.sleep(sleepIntervalInMillSec);
            }
​
            //6.最后还要汇报一次
            reportTaskGroupCommunication(lastTaskGroupContainerCommunication, taskCountInThisTaskGroup);
        } catch (Throwable e) {
            Communication nowTaskGroupContainerCommunication = this.containerCommunicator.collect();
            if (nowTaskGroupContainerCommunication.getThrowable() == null) {
                nowTaskGroupContainerCommunication.setThrowable(e);
            }
            nowTaskGroupContainerCommunication.setState(State.FAILED);
            this.containerCommunicator.report(nowTaskGroupContainerCommunication);
            throw DataXException.asDataXException(
                    FrameworkErrorCode.RUNTIME_ERROR, e);
        }finally {
            if(!PerfTrace.getInstance().isJob()){
                //最后打印cpu的平均消耗，GC的统计
                VMInfo vmInfo = VMInfo.getVmInfo();
                if (vmInfo != null) {
                    vmInfo.getDelta(false);
                    LOG.info(vmInfo.totalString());
                }
                LOG.info(PerfTrace.getInstance().summarizeNoException());
            }
        }
    }
    
}

```

TaskExecute 具体执行类  
开启了2个线程，分别进行读操作和写操作。

数据处理：读操作（ReaderRunner）把从数据库中读出来的每条数据封装为一个个Record放入Channel中,当数据读完时，写入一个TerminateRecord标识结束

写操作（WriterRunner）不断从Channel中读取Record，直到读到TerminateRecord标识数据以取完

ReaderRunner分别执行#init, #prepare, #startRead, #post 并记录每个阶段的处理信息(数据量，数据大小…)

WriterRunner分别执行#init, #prepare, #start, #post 并记录每个阶段的处理信息(数据量，数据大小…)

```
class TaskExecutor {
    private Configuration taskConfig;   //当前任务配置项
    private Channel channel;    //管道 用于缓存读出来的数据
    private Thread readerThread;    //读线程
    private Thread writerThread;    //写线程
    private ReaderRunner readerRunner;
    private WriterRunner writerRunner;
​
    /**
     * 该处的taskCommunication在多处用到：
     * 1. channel
     * 2. readerRunner和writerRunner
     * 3. reader和writer的taskPluginCollector
     */
    private Communication taskCommunication;
​
    public TaskExecutor(Configuration taskConf, int attemptCount) {
        // 获取该taskExecutor的配置
        this.taskConfig = taskConf;
        //...
        /**
         * 由taskId得到该taskExecutor的Communication
         * 要传给readerRunner和writerRunner，同时要传给channel作统计用
         */
        this.taskCommunication = containerCommunicator
                .getCommunication(taskId);
        //实例化存储读数据的管道
        this.channel = ClassUtil.instantiate(channelClazz,
                Channel.class, configuration);
        this.channel.setCommunication(this.taskCommunication);
        /**
         * 获取transformer的参数
         */
        List<TransformerExecution> transformerInfoExecs = TransformerUtil.buildTransformerInfo(taskConfig);
        /**
         * 生成writerThread
         */
        writerRunner = (WriterRunner) generateRunner(PluginType.WRITER);
        this.writerThread = new Thread(writerRunner,
                String.format("%d-%d-%d-writer",
                        jobId, taskGroupId, this.taskId));
        //通过设置thread的contextClassLoader，即可实现同步和主程序不通的加载器
        this.writerThread.setContextClassLoader(LoadUtil.getJarLoader(
                PluginType.WRITER, this.taskConfig.getString(
                        CoreConstant.JOB_WRITER_NAME)));
        /**
         * 生成readerThread
         */
        readerRunner = (ReaderRunner) generateRunner(PluginType.READER,transformerInfoExecs);
        this.readerThread = new Thread(readerRunner,
                String.format("%d-%d-%d-reader",
                        jobId, taskGroupId, this.taskId));
        /**
         * 通过设置thread的contextClassLoader，即可实现同步和主程序不通的加载器
         */
        this.readerThread.setContextClassLoader(LoadUtil.getJarLoader(
                PluginType.READER, this.taskConfig.getString(
                        CoreConstant.JOB_READER_NAME)));
    }
​
    public void doStart() {
        this.writerThread.start();
        // reader没有起来，writer不可能结束
        if (!this.writerThread.isAlive() || this.taskCommunication.getState() == State.FAILED) {
            throw DataXException.asDataXException(
                    FrameworkErrorCode.RUNTIME_ERROR,
                    this.taskCommunication.getThrowable());
        }
        this.readerThread.start();
        // 这里reader可能很快结束
        if (!this.readerThread.isAlive() && this.taskCommunication.getState() == State.FAILED) {
            // 这里有可能出现Reader线上启动即挂情况 对于这类情况 需要立刻抛出异常
            throw DataXException.asDataXException(
                    FrameworkErrorCode.RUNTIME_ERROR,
                    this.taskCommunication.getThrowable());
        }
​
    }
}

```

以MySqlReader插件为例，ReaderRunner在执行#init, #prepare, #startRead, #post 时，实现在MysqlReader中

```
public static class Task extends Reader.Task {
    public void startRead(RecordSender recordSender) {
        int fetchSize = this.readerSliceConfig.getInt(Constant.FETCH_SIZE);
        //mysqlReader调commonRdbmsReaderTask读数据
        this.commonRdbmsReaderTask.startRead(this.readerSliceConfig, recordSender,
                super.getTaskPluginCollector(), fetchSize);
    }
    //...
}

```
```
public void startRead(Configuration readerSliceConfig,
                      RecordSender recordSender,
                      TaskPluginCollector taskPluginCollector, int fetchSize) {
    //获取job配置中的querySql
    String querySql = readerSliceConfig.getString(Key.QUERY_SQL);
    //获取job配置中的tabelName   querySql和table 不能并存,否则程序会提示配置错误
    String table = readerSliceConfig.getString(Key.TABLE);
    PerfTrace.getInstance().addTaskDetails(taskId, table + "," + basicMsg);
    LOG.info("Begin to read record by Sql: [{}\n] {}.",
            querySql, basicMsg);
    PerfRecord queryPerfRecord = new PerfRecord(taskGroupId,taskId, PerfRecord.PHASE.SQL_QUERY);
    queryPerfRecord.start();
    //获取数据库连接
    Connection conn = DBUtil.getConnection(this.dataBaseType, jdbcUrl,
            username, password);
​
    // session config .etc related
    DBUtil.dealWithSessionConfig(conn, readerSliceConfig,
            this.dataBaseType, basicMsg);
    int columnNumber = 0;
    ResultSet rs = null;
    try {
        rs = DBUtil.query(conn, querySql, fetchSize);   //拿数据
        queryPerfRecord.end();
        //获取表原信息
        ResultSetMetaData metaData = rs.getMetaData();
        columnNumber = metaData.getColumnCount();
        //这个统计干净的result_Next时间
        PerfRecord allResultPerfRecord = new PerfRecord(taskGroupId, taskId, PerfRecord.PHASE.RESULT_NEXT_ALL);
        allResultPerfRecord.start();
        long rsNextUsedTime = 0;
        long lastTime = System.nanoTime();
        while (rs.next()) {
            rsNextUsedTime += (System.nanoTime() - lastTime);
            //将读的数据放入recordSender对象的channel中
            this.transportOneRecord(recordSender, rs,
                    metaData, columnNumber, mandatoryEncoding, taskPluginCollector);
            lastTime = System.nanoTime();
        }
        allResultPerfRecord.end(rsNextUsedTime);
        //目前大盘是依赖这个打印，而之前这个Finish read record是包含了sql查询和result next的全部时间
        LOG.info("Finished read record by Sql: [{}\n] {}.",
                querySql, basicMsg);
    }catch (Exception e) {
        throw RdbmsException.asQueryException(this.dataBaseType, e, querySql, table, username);
    } finally {
        DBUtil.closeDBResources(null, conn);
    }
}
protected Record transportOneRecord(RecordSender recordSender, ResultSet rs, 
        ResultSetMetaData metaData, int columnNumber, String mandatoryEncoding, 
        TaskPluginCollector taskPluginCollector) {
    //每条数据组装一个record
    Record record = buildRecord(recordSender,rs,metaData,columnNumber,mandatoryEncoding,taskPluginCollector); 
    //record放入recordSender缓存
    recordSender.sendToWriter(record);
    return record;
}
flush
public void sendToWriter(Record record) {
   //...
   boolean isFull = (this.bufferIndex >= this.bufferSize || this.memoryBytes.get() + record.getMemorySize() > this.byteCapacity);
   if (isFull) {
      flush();  //这里会把buffer放入channel中
   }
   //放入集合列表
   this.buffer.add(record);
   this.bufferIndex++;
   memoryBytes.addAndGet(record.getMemorySize());
}
​
public void flush() {
    //...
    this.channel.pushAll(this.buffer);
    //...清空buffer
}

```

看完#flush可能会有个疑问,这里要进行判断是否ifFull才把buffer放入channel。那最后一批不满足ifFull条件的数据如何处理？

写入  
taskReader.startRead(recordSender); //读插件开始读  
recordSender.terminate(); //把缓存中剩余数据写如recordSender的channel中，并且写入TerminateRecord打上结束标记  
至此读任务就结束了

接下来看写任务

以MySqlWriter插件为例，WriterRunner分别执行#init, #prepare, #start, #post， 实现在MySqlWriter中

```
public static class Task extends Writer.Task {
    //TODO 改用连接池，确保每次获取的连接都是可用的（注意：连接可能需要每次都初始化其 session）
    public void startWrite(RecordReceiver recordReceiver) {
        this.commonRdbmsWriterTask.startWrite(recordReceiver, this.writerSliceConfig,
                super.getTaskPluginCollector());
    }
     //...
}
public void startWrite(RecordReceiver recordReceiver,
                       Configuration writerSliceConfig,
                       TaskPluginCollector taskPluginCollector) {
    Connection connection = DBUtil.getConnection(this.dataBaseType,
            this.jdbcUrl, username, password);
    DBUtil.dealWithSessionConfig(connection, writerSliceConfig,
            this.dataBaseType, BASIC_MESSAGE);
    startWriteWithConnection(recordReceiver, taskPluginCollector, connection);  //开始写
}

```

//…清空缓存,关闭资源

```
public void startWriteWithConnection(RecordReceiver recordReceiver, TaskPluginCollector taskPluginCollector, Connection connection) {
    this.taskPluginCollector = taskPluginCollector;
    // 用于写入数据的时候的类型根据目的表字段类型转换
    this.resultSetMetaData = DBUtil.getColumnMetaData(connection,
            this.table, StringUtils.join(this.columns, ","));
    // 写数据库的SQL语句
    calcWriteRecordSql();
    List<Record> writeBuffer = new ArrayList<Record>(this.batchSize);
    int bufferBytes = 0;
    try {
        Record record;
        while ((record = recordReceiver.getFromReader()) != null) {
            //...读列与写列数量不一致，会抛异常,这里省略
            //写缓存添加数据
            writeBuffer.add(record);
            bufferBytes += record.getMemorySize();
            if (writeBuffer.size() >= batchSize || bufferBytes >= batchByteSize) {
                doBatchInsert(connection, writeBuffer); //开始批量写
                writeBuffer.clear();
                bufferBytes = 0;
            }
        }
        if (!writeBuffer.isEmpty()) {
            doBatchInsert(connection, writeBuffer);
            writeBuffer.clear();
            bufferBytes = 0;
        }
    } catch (Exception e) {
        //...
    } finally {
        //...清空缓存,关闭资源
    }
}

```

写任务是批量操作,值得注意的是，当批量写操作出现错误时，程序会争取再执行一次写操作,对每条数据分别进行写操作

在#doOneInsert中

```
protected void doBatchInsert(Connection connection, List<Record> buffer)
        throws SQLException {
    PreparedStatement preparedStatement = null;
    try {
        connection.setAutoCommit(false);
        preparedStatement = connection
                .prepareStatement(this.writeRecordSql);
        for (Record record : buffer) {
            preparedStatement = fillPreparedStatement(
                    preparedStatement, record);
            preparedStatement.addBatch();   //批量提交
        }
        preparedStatement.executeBatch();   //批量执行
        connection.commit();
    } catch (SQLException e) {
        LOG.warn("回滚此次写入, 采用每次写入一行方式提交. 因为:" + e.getMessage());
        connection.rollback();
        doOneInsert(connection, buffer);
    } catch (Exception e) {
        throw DataXException.asDataXException(
                DBUtilErrorCode.WRITE_DATA_ERROR, e);
    } finally {
        DBUtil.closeDBResources(preparedStatement, null);
    }
}
​
protected void doOneInsert(Connection connection, List<Record> buffer) {
    PreparedStatement preparedStatement = null;
    try {
        connection.setAutoCommit(true);
        preparedStatement = connection
                .prepareStatement(this.writeRecordSql);
        for (Record record : buffer) {
            try {
                preparedStatement = fillPreparedStatement(
                        preparedStatement, record);
                preparedStatement.execute();
            } catch (SQLException e) {
                LOG.debug(e.toString());
                this.taskPluginCollector.collectDirtyRecord(record, e);
            } finally {
                // 最后不要忘了关闭 preparedStatement
                preparedStatement.clearParameters();
            }
        }
    } catch (Exception e) {
        //...抛异常
    } finally {
        //...关闭资源
    }
}

```

至此写操作完毕

[](https://blog.csdn.net/ychang577/article/details/108226450)总结
---------------------------------------------------------------

整个应用，采用framework + read plugin + write plugin方式。对于核心配置文件core.json ，任务配置文件 job.json, 插件配置文件plugin.json 程序已经处理。并且结合Communication收集处理数据指标。我们只需要实现  
Reader , Reader.Job , Reader.Task  
Writer , Writer.Job , Writer.Task  
根据项目的需求进行处理即可





本文转自 [https://blog.csdn.net/ychang577/article/details/108226450](https://blog.csdn.net/ychang577/article/details/108226450)，如有侵权，请联系删除。
