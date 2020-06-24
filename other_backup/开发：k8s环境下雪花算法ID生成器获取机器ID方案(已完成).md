#### 一、场景

使用雪花算法生成全局唯一ID。雪花算法需要获取一个机器ID，在K8S集群部署的情况下，这个ID没办法写死在配置里，因为集群部署的时候，用同一个镜像部署，ID会重复，所以只能想办法从其他地方拿这个ID。

这里我们选择的是Redis。用Redis存ID，需要考虑两个问题：ID怎么注册，ID怎么回收。

下面是两个方案：一个是手动删除ID的方案，另外一个是ID会自动过期的方案。最终我选择了方案二：

#### 二、方案一

##### 思路

1.app启动时，去Redis注册ID。

2.app关闭时，去Redis注销ID。

##### 风险

1.Redis满了：没有风险。

2.Redis挂了：做好持久化，重启后也没有风险。

3.Spring挂了：这个时候没法正常退出，ID无法回收。极端情况下，ID就被app不断重启用完了。处理方案：需要再开一个线程，或者提供一个接口，手动去回收线程。思路：新开启一个线程，每3秒不断的去刷新ID的值为true。回收时，用另一个线程或手动全部置为false，然后等待3秒后，检查值为true的ID数量与当前服务器一致，然后删除值为false的ID。

由于方案二需要运维人员介入，所以最后选择了方案一。

#### 三、方案二

##### 思路

1.app启动时去Redis注册一个机器ID，该ID可自动过期。ID的范围为1-1023(这个范围由雪花算法设定的位数决定)，注册ID时由1往后遍历至1023，注册遇到的第1个没有被占用的ID，也就是Redis上面没有的ID。

2.app再另起一个线程，每3秒不断的去刷新这个ID，让它不过期。如果Redis上没有这个ID，则自动注册ID。

##### 风险

1.Redis满了：这时Redis可能会删除已有数据。需要把Redis的数据淘汰策略设置为满了就拒绝写操作，而不是删除已有数据，从而避免ID被删除。

2.Redis挂了：这时因为进程刷新不到ID，ID有可能过期。需要把过期时间调长，比如调为72小时。Redis整个挂了以后有12小时的恢复时间。

3.Spring挂了：这时因为**ID会自动过期**，没有风险，ID会被自动回收。

4.极端情况，定时刷新线程没了：有极小几率，定时刷新线程死了。这个时候ID有可能被回收掉。**需要对这个线程做一个监控**，确保它活着。

5.极端情况，ID被不断重启用完了：ID有效期设为12小时。如果ID在12小时内取了1023次，那么ID就被取完了。这种情况下，需要：(1)刷新线程的方法增加逻辑，当没有检测到ID存在时，要自动注册。(2)这种情况下，手动恢复的方法：**禁止所有新app的pod启动**，然后删除所有Redis的ID，等待3秒后，让app的ID刷新线程自动注册上来。

##### 注意事项总结

**1.Redis策略需要设置为满了时拒绝写，不删除过期类型的数据。**

**2.增加一个监控，确保app上刷新有效时间的线程活着，它死了的话有3天的时间修复或重启这个app服务。（3天后Redis上这个app的ID就会过期，ID这个app还在用，而新启动的服务会重复获取到这个ID）。**

**3.如果ID在3天内被耗尽，恢复的方法：禁止所有新app的pod启动，然后删除所有Redis上的ID，等待3秒后，让app的ID刷新线程自动注册即可。**

#### 四、方案二最终实施代码

##### 1.配置监听器获取机器ID

```java
/**
 * @Desc 用于在Spring的Bean注入完成后，去Redis取雪花算法的machineId
 */
@Slf4j
public class DataMachineIdListener implements ApplicationListener<ContextRefreshedEvent> {
    @Resource
    private RedisAgent redisAgent;

    // 机器ID刷新周期：3秒。决定了Redis上面ID被清除时，老服务需要多久能重新注册。这也决定了新服务启动时的最小等待时间。
    private final Long REFRESH_PERIOD = 3 * 1000L;
    // 机器ID有效期：3天。决定了刷新时间的线程挂了，有多长时间可以用来恢复。
    private final Long EXPIRE_MILLIS = 3 * 24 * 60 * 60 * 1000L;
    // 获取机器ID的延迟时间。获取机器ID和ID初始化有可能是并发执行的。因此，获取ID时，需要至少延迟大于一个REFRESH_PERIOD的时间，等待机器ID初始化完成。
    private final Long DELAY_MILLIS = REFRESH_PERIOD + 1000L;

    private volatile Long registedDataMachineId = 0L;
    private final Lock dLock = new ReentrantLock();
    private final Condition idRegisted = dLock.newCondition();

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        dLock.lock();
        try {
            this.registDataMachineId(event);
            idRegisted.signalAll();
        } finally {
            dLock.unlock();
        }

        // 测试
//        testRegistDataMachineId(event);
    }

    // 注册机器ID
    private void registDataMachineId(ContextRefreshedEvent  event){
        WindProperties windProperties = event.getApplicationContext().getBean(WindProperties.class);
        Long dataMachineId = windProperties.getDataMachineId();
        String appName = event.getApplicationContext().getApplicationName();

        // 假设Redis挂掉重启时，数据全部被干掉。这个时候新服务在启动时，要留时间给正在运行的服务注册他们正在使用的dataMachineId。
        // 老服务最迟3秒可以刷新服务。新服务暂停DELAY_MILLIS秒，在加上新服务启动前SpringContext初始化的时间，足够老服务注册完成了。
        try {
            Thread.sleep(DELAY_MILLIS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // 尝试获取机器ID。改造的雪花算法规定的机器ID范围是1-1023
        for (int i = 1; i < 1023; i++) {
            if (redisAgent.setNx("common:dataMachineId_"+ i, "true", EXPIRE_MILLIS)) {
                // 获取ID成功。
                // 1.设置本机Id为该机器Id
                windProperties.setDataMachineId(i);
                registedDataMachineId = windProperties.getDataMachineId();
                log.info("获取机器ID成功。appName:{}, dataMachineId:{}", appName, i);
                // 2.新开一个线程，定时刷新
                this.refreshDataMachineIdExpireTime(
                        appName, REFRESH_PERIOD, "common:dataMachineId_"+ i, "true", EXPIRE_MILLIS);
                return;
            }
        }

        // 获取ID失败，机器ID已耗尽
        RuntimeException e = new RuntimeException("dataMachineId资源已耗尽，无法获取机器ID。请手动修复。" +
                "方法：保证当前没有新的app启动，然后删除Redis上所有的dataMachineId，等待3秒，让app的ID刷新线程重新自动注册。" +
                "确认Redis上机器ID的Key的数量与实际运行的pod数一致。");
        e.printStackTrace();
        throw e;
    }


    /**
     * 新开一个线程，定时刷新dataMachineId使用时间
     * @param refreshPeriod 定时刷新周期
     * @param key 刷新key
     * @param value 刷新次数
     * @param expireMillis key有效时间
     */
    private void refreshDataMachineIdExpireTime(
            String appName, Long refreshPeriod, String key, String value, Long expireMillis) {
        //开启一个线程执行定时任务:
        //1.每23小时更新一次超时时间
        new Timer("refreshDataMachineIdExpireTimeThread:"+appName+":"+key).schedule(new TimerTask() {
            @Override
            public void run() {
                // 没有则直接注册。防止Redis挂掉重启时，Reids没做持久化，导致dataMachineId被回收。
                // 有的话就直接刷新有效时间
                try {
                   redisAgent.setString(key, value, expireMillis);
//                        log.info("刷新机器ID过期时间, appname:{}, key:{}, value:{}, expireMillis:{}, refreshTime:{}",
//                            appName, key, value, expireMillis, new Date());
                } catch (Exception e) {
                    // 加try，防止Redis挂掉时，该线程报错死掉
                    e.printStackTrace();
                    log.error("刷新refreshDataMachineId失败！appname:{}, key:{}, value:{}, expireMillis:{}, refreshTime:{}。e:",
                            appName, key, value, expireMillis, new Date(), e);
                }
            }
        }, 3 * 1000, 3 * 1000);
    }

    // ID初始化时最少会延迟DELAY_MILLIS，因此超时时间必须大于这个数
    public Long getRegistedDataMachineId(Long timeOut) {
        Assert.isTrue(timeOut >= DELAY_MILLIS + 1000L, "超时时间小于最小初始化时间");

        Long startTime = System.nanoTime();
        dLock.lock();
        try{
            while (registedDataMachineId <= 0) {
                try {
                    idRegisted.await(timeOut, TimeUnit.MILLISECONDS);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                Long curTime = System.nanoTime();
                if ((registedDataMachineId > 0) || (curTime - startTime > timeOut)) {
                    break;
                }
            }

            if (registedDataMachineId <= 0) {
                throw new RuntimeException("超时，dataMachineId注册失败！dataMachineId值：" + registedDataMachineId);
            }

            return registedDataMachineId;
        } finally {
            dLock.unlock();
        }
    }

    public Long getRegistedDataMachineId() {
        // 默认超时时间为两个最小获取时间。
        return this.getRegistedDataMachineId(DELAY_MILLIS * 2);
    }

    private void testRegistDataMachineId(ContextRefreshedEvent event) {
        for (int i = 1; i <= 500; i++) {
            if (i==1) {
                System.out.println("=========1:"+new Date());
            }
            if (i==500) {
                System.out.println("=========500:"+new Date());
            }
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(RandomUtils.nextLong(0L,2000L));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    registDataMachineId(event);
                }
            }).start();
        }
    }
}
```

##### 2.雪花算法工具类，从监听器拿取机器ID

```java
/**
 *
 * Twitter_Snowflake
 *
 * <h2>SnowFlake的结构如下(每部分用-分开):</h2>
 * <p>0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000
 * <ul>
 * <li>1位标识，由于long基本类型在Java中是带符号的，最高位是符号位，正数是0，负数是1，所以id一般是正数，最高位是0
 * <li>41位时间截(毫秒级)，注意，41位时间截不是存储当前时间的时间截，而是存储时间截的差值（当前时间截 - 开始时间截)
 * 得到的值），这里的的开始时间截，一般是我们的id生成器开始使用的时间，由我们程序来指定的（如下下面程序IdWorker类的startTime属性）。
 * 41位的时间截，可以使用69年，年T = (1L << 41) / (1000L * 60 * 60 * 24 * 365) = 69
 * <li>10位的数据机器位，可以部署在1024个节点，包括5位datacenterId和5位workerId
 * <li>12位序列，毫秒内的计数，12位的计数顺序号支持每个节点每毫秒(同一机器，同一时间截)产生4096个ID序号。12位不够用时强制得到新的时间前缀。
 * </ul>
 * <p>加起来刚好64位，为一个Long型。
 *
 * <h2>优缺点</h2>
 * <ul>
 * <li>优点：整体上按照时间自增排序，并且整个分布式系统内不会产生ID碰撞(由数据中心ID和机器ID作区分)，并且效率较高。
 * <li>缺点：对系统时间的依赖性非常强，需关闭ntp的时间同步功能。
 * </ul>
 *
 * <h2>改造</h2>
 * <ul>
 * <li>将5位datacenterId和5位workerId合并为dataMachineId（10位的数据机器位）。
 * <li>dataMachineId通过分布式配置中心配置，默认为0.
 * </ul>
 */
@Slf4j
public final class SnowFlakeIdUtil {
    /** 起始的时间戳 */
    private final static long START_STAMP = 1567267200000L; // 2019年9月1日

    /** 每一部分占用的位数 */
    private final static long SEQUENCE_BIT = 12; // 序列号占用的位数
    private final static long DATE_MACHINE_BIT = 10; // 数据机器位

    /** 每一部分的最大值 */
    private final static long MAX_DATE_MACHINE_NUM = -1L ^ (-1L << DATE_MACHINE_BIT);
    private final static long MAX_SEQUENCE = -1L ^ (-1L << SEQUENCE_BIT);

    /** 每一部分向左的位移 */
    private final static long DATE_MACHINE_LEFT = SEQUENCE_BIT;
    private final static long TIMESTAMP_LEFT = DATE_MACHINE_LEFT + DATE_MACHINE_BIT;

    /** 数据机器标识（通过分布式配置中心配置，默认为0）  */
    private long dataMachineId;
    /** 序列号 */
    private long sequence = 0L;
    /** 上一次时间戳 */
    private volatile long lastStamp = -1L;

    private static volatile SnowFlakeIdUtil idWorker = null;

    public static SnowFlakeIdUtil getInstance() {
        if (ObjectUtil.isNull(idWorker)) {
            synchronized (SnowFlakeIdUtil.class) {
                if (ObjectUtil.isNull(idWorker)) {
                    idWorker = new SnowFlakeIdUtil();
                }
            }
        }

        return idWorker;
    }

    private SnowFlakeIdUtil() {
//    Long dataMachineIdL = SpringContextUtil.getBean(WindProperties.class).getDataMachineId();
        DataMachineIdListener dlisten = SpringContextUtil.getBean(DataMachineIdListener.class);
        if (ObjectUtil.isNull(dlisten)) {
            throw new IllegalArgumentException("DataMachineIdListener未注入！");
        }

        dataMachineId = dlisten.getRegistedDataMachineId();
        log.info("++++++++++++++++++++++++++++++ Id生成器所使用的dataMachineId:" + dataMachineId);
        if (dataMachineId > MAX_DATE_MACHINE_NUM || dataMachineId <= 0) {
            throw new IllegalArgumentException("dataMachineId 取值范围(0, " + MAX_DATE_MACHINE_NUM + "]");
        }
    }

    /**
     * 产生下一个ID
     *
     * @return
     */
    public synchronized long nextId() {
        long currStamp = getNewStamp();
        if (currStamp < lastStamp) {
            throw new RuntimeException("发生时钟回拨，拒绝生产id");
        }

        if (currStamp == lastStamp) {
            // 相同毫秒内，序列号自增
            sequence = (sequence + 1) & MAX_SEQUENCE;
            // 同一毫秒的序列数已经达到最大
            if (sequence == 0L) {
                currStamp = getNextMill();
            }
        } else {
            // 不同毫秒内，序列号置为0
            sequence = 0L;
        }

        lastStamp = currStamp;

        return (currStamp - START_STAMP) << TIMESTAMP_LEFT // 时间戳部分
                | dataMachineId << DATE_MACHINE_LEFT // 数据机器标识部分
                | sequence; // 序列号部分
    }

    private long getNextMill() {
        long mill = getNewStamp();
        while (mill <= lastStamp) {
            mill = getNewStamp();
        }
        return mill;
    }

    private long getNewStamp() {
        return System.currentTimeMillis();
    }

}
```

##### 3.监听器注入

在sprint.factorires注册自动配置类

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.wind.cloud.starter.configure.WindAutoConfigure
```

在自动配置类配置监听器自动注入

```java
@Configuration
@EnableConfigurationProperties(WindProperties.class)
public class WindAutoConfigure {
    @Bean
    public DataMachineIdListener dataMachineIdListener() {
        return new DataMachineIdListener();
    }
}
```

##### 4.其他工具类

1.SpringContextUtil

```java
public class SpringContextUtil implements ApplicationListener<ContextRefreshedEvent> {

    private static ApplicationContext applicationContext;

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (SpringContextUtil.applicationContext == null) {
            SpringContextUtil.applicationContext = event.getApplicationContext();
        }
    }

    public static ApplicationContext getApplicationContext() {
        return applicationContext;
    }

    /**
     * 获取class对应的bean
     *
     * @param <T> 类型
     * @return bean
     */
    public static <T> T getBean(Class<T> requiredType) {
        if (applicationContext == null) {
            return null;
        }
        return applicationContext.getBean(requiredType);
    }

    public static <T> T getBean(String name, Class<T> requiredType) {
        if (applicationContext == null) {
            return null;
        }
        return applicationContext.getBean(name, requiredType);
    }
}
```
注入：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.wind.cloud.starter.common.business.autoconfigure.UtilAutoConfigure
```

```java
@Configuration
public class UtilAutoConfigure {
    // 获取context存入Util，可方便的操作yml等文件配置
    @Bean
    public SpringContextUtil springContextUtil() {
        return new SpringContextUtil();
    }
}
```

2.RedisAgent

```java
/**
 * @Desc redis常用api封装
 */
@Slf4j
public class RedisAgent {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    
        /**
     * 设置k-v对
     *
     * @param key
     * @param value
     * @param expireMillis 有效期（毫秒）
     */
    public void setString(String key, String value, long expireMillis) {
        stringRedisTemplate.opsForValue().set(key, value, expireMillis, TimeUnit.MILLISECONDS);
    }
    
        /**
     * 根据key获取value
     *
     * @param key
     * @return
     */
    public String getString(String key) {
        return stringRedisTemplate.opsForValue().get(key);
    }

    /**
     * 不存在则设置；存在则不设置
     *
     * @param key
     * @param value
     * @param expireMillis 有效期（毫秒）
     * @return {@code true}表示成功；{@code false}表示失败；null when used in pipeline / transaction.
     */
    public Boolean setNx(String key, String value, long expireMillis) {
        return stringRedisTemplate.execute(new RedisCallback<Boolean>() {
            @Override
            public Boolean doInRedis(RedisConnection connection) throws DataAccessException {
                return connection.set(key.getBytes(), value.getBytes(), Expiration.milliseconds(expireMillis),
                        RedisStringCommands.SetOption.SET_IF_ABSENT);
            }
        }, true);
    }
```
注入：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.wind.cloud.starter.redis.autoconfigure.RedisAutoConfigure
```


```java
@Configuration
@ConditionalOnClass({ Redisson.class, RedisOperations.class })
@EnableCaching
public class RedisAutoConfigure {
    @Bean
    public RedisAgent redisAgent() {
        return new RedisAgent();
    }
}
```

3.WindProperties

```java
/**
 * @Desc yml文件公共属性配置定义
 */
@Getter
@Setter
@ConfigurationProperties(prefix = "wind")
public class WindProperties extends BaseDto {
    /** id生成器数据机器标识配置, 范围：1-1023 */
    private long dataMachineId;
    private String logProfile;

    /** 切面配置 */
    private AspectProperties aspect = new AspectProperties();

    @UtilityClass
    public static final class PropertiesName {
        public static final String DATA_MACHINE_ID = "dataMachineId";
        public static final String LOG_PROFILE = "logProfile";
    }
}
```
