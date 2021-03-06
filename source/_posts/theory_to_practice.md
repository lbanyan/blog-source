---
title: 理论到实际的总结
date: 2019-9-14 11:15
---

### 自旋锁理论在百万级消息消费中的应用
在进行小电铺装修批量复制时，发送复制商品会先于发送复制营销活动，以尽量保证复制商品先于复制营销活动（营销活动的创建是基于某商品来创建的），复制商品是一个很短小的任务，耗时极短。偶尔会出现部分营销活动复制不成功的情况，其原因也大多是对应的商品还未复制成功。对于这种情况，我便仿照自旋锁理论，在复制营销活动未找到对应商品时，进行自旋3次。这样就未出现过营销活动未复制成功的情况了。

### 借鉴Spring Bean管理模式，实现策略模式
新上线统一用户系统时，需要将旧系统的用户迁移至该系统，用户基本信息的迁移、用户验证信息的迁移、用户地址的迁移，不同的迁移需要使用不同的策略，这时就需要使用策略模式，使代码更优雅。具体如下：
``` java
public abstract class Migration extends AbstractMigrationService {

    public abstract MigrationTypeEnum getType();

    public abstract void migrate(MigrationInfo migrationInfo);
}

@Component
public class MigrationCollector implements ApplicationContextAware, InitializingBean {

    final Map<Integer, Migration> cache = new HashMap<>();
    ApplicationContext applicationContext;

    @Override
    public void afterPropertiesSet() {

        Map<String, Migration> map = applicationContext.getBeansOfType(Migration.class);
        for (Migration handler : map.values()) {
            MigrationTypeEnum type = handler.getType();
            AssertUtil.isEmpty(cache.get(type.getValue()), String.format("存在两个相同的 Handler", type));
            cache.put(type.getValue(), handler);
        }
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {

        this.applicationContext = applicationContext;
    }

    public Migration getMigrationHandler(Integer type) {

        return cache.get(type);
    }
}

/**
 * 旧系统用户绑定手机号，更新新库
 *
 * @author xingping
 */
@Component
public class OTNPhoneMigration extends Migration {

    @Resource
    private NewUserDao newUserDao;

    @Override
    public MigrationTypeEnum getType() {
        return MigrationTypeEnum.OTN_PHONE;
    }

    @Override
    public void migrate(MigrationInfo migrationInfo) {

        // TODO 具体迁移实现
    }
}
```

### Java信号量的使用
有一个定时任务线程和另一个订阅线程对同一个全局变量huyaIps集合进行操作，定时任务线程会直接覆盖huyaIps，订阅线程会对huyaIps进行add或remove元素操作。
此处存在并发问题，例如定时任务覆盖huyaIps前一刻，订阅线程修改了huyaIps，然后定时任务覆盖，会导致huyaIps更新不及时，需要等到下一个定时任务同步周期才能被更新（定时任务是全量同步huyaIps，订阅方式是实时变更huyaIps）。
此时借助信号量Semaphore，许可证为1，两个线程执行前均需要获取许可，只有获取到许可的一方才可以执行。此处两个线程的执行时间很短，用本方法是合适的，不会出现一方严重阻塞另一方的行为。