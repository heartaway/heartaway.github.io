---
layout: post
title: spring quartz 指定trigger的执行机器
categories: Java
description:
keywords: quartz
date: 2018-03-06
---

我们在使用spring-quartz的时候，常常会遇到一个场景，那就是任务的调度会随着争取锁的先后顺序而出现不固定机器执行的场景，这在正常业务逻辑中具备了很好的容灾能力，但是在我们排查问题时，却带来了困绕，如果出现问题，我们期望任务调度固定在一台机器上进行执行，方便我们对问题的定位和排查。

这里就探讨如何扩展spring-quartz来实现任务的固定机器执行。

### 整体思路
要指定机器运行trigger，那首先我们必须清楚spring-quartz cluster模式下，任务的触发时如何进行分布式执行的。
spring-quartz cluster 是借助数据锁来实现并发控制的，需要注意的是分布式环境下需要保证各机器系统时间一致性；

核心处理线程QuartzSchedulerThread决定了扫描那些JOB，以及触发执行和生命周期的维护。
QuartzSchedulerThread中run方法：


```java
//在trigger表中扫描指定SCHED_NAME、状态为WAITING，下次触发时间在30秒内的触发器
triggers = qsRsrcs.getJobStore().acquireNextTriggers(
                                now + idleWaitTime, Math.min(availThreadCount, qsRsrcs.getMaxBatchSize()), qsRsrcs.getBatchTimeWindow());
....                  
//触发器执行
List<TriggerFiredResult> res = qsRsrcs.getJobStore().triggersFired(triggers);                                                          
...
//释放触发器
qsRsrcs.getJobStore().releaseAcquiredTrigger(triggers.get(i));
...
//完成触发器
qsRsrcs.getJobStore().triggeredJobComplete(triggers.get(i), bndle.getJobDetail(), CompletedExecutionInstruction.SET_ALL_JOB_TRIGGERS_ERROR);                          
```

我们就可以在acquireNextTriggers中做扩展，获取全部或者指定了实例ID的trigger。

延伸思考：acquireNextTriggers获取到trigger列表后，假设机器宕机，这些trigger如何路由到其他机器中正常运行？


### 扩展实例ID的生成策略
spring-quartz一般配置的org.quartz.scheduler.instanceId：AUTO，采用的是SimpleInstanceIdGenerator ID生成策略。

```java
public class SimpleInstanceIdGenerator implements InstanceIdGenerator {
    public String generateInstanceId() throws SchedulerException {
        try {
            return InetAddress.getLocalHost().getHostName() + System.currentTimeMillis();
        } catch (Exception e) {
            throw new SchedulerException("Couldn't get host name!", e);
        }
    }
}
```
SimpleInstanceIdGenerator 采用机器主机名称与当前时间戳作为instanceId,我们期望在开发环境使用hostName，在生产环境(Linux)采用IP作为实例ID。

实现InstanceIdGenerator接口，实现自己的ID生成策略，QuartzSchedulerInstanceIdGenerator的实现逻辑为：

```java
public class QuartzSchedulerInstanceIdGenerator implements InstanceIdGenerator {
    private static final Logger log = LoggerFactory.getLogger(QuartzSchedulerInstanceIdGenerator.class);
    private static final String OS_NAME = "os.name";
    private static final String WINDOWS = "Windows";
    private static final String MAC = "Mac OS";

    @Override
    public String generateInstanceId() throws SchedulerException {
        String id;
        try {
            if (isLocalDev()) {
                id = getHostName();
            } else {
                id = IpUtil.getIp();
            }

            if (StringUtils.isBlank(id)) {
                id = InetAddress.getLocalHost().getHostName() + System.currentTimeMillis();
            }
        } catch (Exception e) {
            throw new SchedulerException("Couldn't generate instance id!", e);
        }
        return id;
    }

    private String getHostName() throws SchedulerException {
        try {
            return InetAddress.getLocalHost().getHostName();
        } catch (Exception e) {
            throw new SchedulerException("Couldn't get host name!", e);
        }
    }

    private boolean isLocalDev() {
        if (StringUtils.indexOfIgnoreCase(System.getProperty(OS_NAME), WINDOWS) >= 0) {
            return true;
        }

        if (StringUtils.indexOfIgnoreCase(System.getProperty(OS_NAME), MAC) >= 0) {
            return true;
        }
        return false;
    }
}
```

在quartz.properties中配置：

```java
org.quartz.scheduler.instanceIdGenerator.class=com.xx.ext.QuartzSchedulerInstanceIdGenerator
```

### 扩展StdJDBCDelegate
SchedulerFactoryBean 工厂Bean负责加载配置信息，初始化SchedulerFactory和Scheduler实例，jobStore负责job、trigger等的持久化工作，针对不同的数据库类型，可以配置不同的DriverDelegate。Spring中默认使用LocalDataSourceJobStore作为JobStore的处理类。

SchedulerFactory中initSchedulerFactory方法:

```java
       if (this.configLocation != null) {
			if (logger.isInfoEnabled()) {
				logger.info("Loading Quartz config from [" + this.configLocation + "]");
			}
			PropertiesLoaderUtils.fillProperties(mergedProps, this.configLocation);
	   }

		CollectionUtils.mergePropertiesIntoMap(this.quartzProperties, mergedProps);

		if (this.dataSource != null) {
			mergedProps.put(StdSchedulerFactory.PROP_JOB_STORE_CLASS, LocalDataSourceJobStore.class.getName());
		}
```
可以看到SchedulerFactoryBean中设置PROP_JOB_STORE_CLASS属性是在合并用户设置的配置文件之后，也就是PROP_JOB_STORE_CLASS的实现类Spring强制指定为LocalDataSourceJobStore而无法更改，即便是我们在quartz.properties中配置了org.quartz.jobStore.class属性，也会被LocalDataSourceJobStore覆盖掉。


LocalDataSourceJobStore继承自JobStoreSupport，JobStoreSupport默认配置使用StdJDBCDelegate作为与数据库交互的代理处理类；我们可以通过扩展StdJDBCDelegate类，来实现底层数据库交互的扩展。通过配置org.quartz.jobStore.driverDelegateClass属性，指定driverDelegate为我们扩展后的Delegate。


在quartz.properties中配置：

```java
org.quartz.jobStore.driverDelegateClass=com.xx.ext.StdJDBCDelegateExt
```

针对${table_prefix}_quartz_triggers表新增字段INSTANCE_NAME，此字段为InstanceIdGenerator生成的实例ID，此字段会透传到StdJDBCDelegate中的属性instanceId中，因此我们可以扩展StdJDBCDelegate，通过instanceId字段来指定获取的trigger列表。

```java
public class StdJDBCDelegateExt extends StdJDBCDelegate {

    String INSTANCE_NAME_VALUE = "{2}";

    String SELECT_NEXT_TRIGGER_TO_ACQUIRE_EXT = "SELECT "
        + COL_TRIGGER_NAME + ", " + COL_TRIGGER_GROUP + ", "
        + COL_NEXT_FIRE_TIME + ", " + COL_PRIORITY + " FROM "
        + TABLE_PREFIX_SUBST + TABLE_TRIGGERS + " WHERE "
        + COL_SCHEDULER_NAME + " = " + SCHED_NAME_SUBST
        + " AND " + COL_TRIGGER_STATE + " = ? AND " + COL_NEXT_FIRE_TIME + " <= ? "
        + " AND (" + COL_INSTANCE_NAME + " IS NULL OR " + COL_INSTANCE_NAME + " = " + INSTANCE_NAME_VALUE + ")"
        + "AND (" + COL_MISFIRE_INSTRUCTION + " = -1 OR (" + COL_MISFIRE_INSTRUCTION + " != -1 AND "
        + COL_NEXT_FIRE_TIME + " >= ?)) "
        + "ORDER BY " + COL_NEXT_FIRE_TIME + " ASC, " + COL_PRIORITY + " DESC";

    /**
     * 允许设置trigger 指定一台机器进行任务调度执行；
     *
     * @param conn
     * @param noLaterThan
     * @param noEarlierThan
     * @param maxCount
     * @return
     * @throws SQLException
     */
    @Override
    public List<TriggerKey> selectTriggerToAcquire(Connection conn, long noLaterThan, long noEarlierThan, int maxCount)
        throws SQLException {
        PreparedStatement ps = null;
        ResultSet rs = null;
        List<TriggerKey> nextTriggers = new LinkedList<TriggerKey>();
        try {
            ps = conn.prepareStatement(rtp(SELECT_NEXT_TRIGGER_TO_ACQUIRE_EXT, instanceId));

            // Set max rows to retrieve
            if (maxCount < 1) {
                maxCount = 1;
            }
            ps.setMaxRows(maxCount);

            // Try to give jdbc driver a hint to hopefully not pull over more than the few rows we actually need.
            // Note: in some jdbc drivers, such as MySQL, you must set maxRows before fetchSize, or you get exception!
            ps.setFetchSize(maxCount);

            ps.setString(1, STATE_WAITING);
            ps.setBigDecimal(2, new BigDecimal(String.valueOf(noLaterThan)));
            ps.setBigDecimal(3, new BigDecimal(String.valueOf(noEarlierThan)));
            rs = ps.executeQuery();

            while (rs.next() && nextTriggers.size() <= maxCount) {
                nextTriggers.add(triggerKey(
                    rs.getString(COL_TRIGGER_NAME),
                    rs.getString(COL_TRIGGER_GROUP)));
            }

            return nextTriggers;
        } finally {
            closeResultSet(rs);
            closeStatement(ps);
        }
    }

    protected final String rtp(String query, String instanceName) {
        return MessageFormat.format(query,
            new Object[] {tablePrefix, getSchedulerNameLiteral(), "'" + instanceName + "'"});
    }
}
```

至此，完成了指定trigger在特定instanceId上运行，但是有一个问题，假如我们期望trigger可以在多台instanceId上随机执行的话，该如何实现呢？



