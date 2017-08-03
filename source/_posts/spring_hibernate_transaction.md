---
title: Spring、Hibernate事务管理
tags:
    - Spring
    - Hibernate
    - 事务
---

本文节选自我们的《自主实现SDN虚拟网络与企业私有云》一书。

在业务允许的范围内，事务应该越小越好。云平台灵活地采用Hibernate和Spring JDBC来完成所需的持久化工作。
如何实现将原本是大事务的业务切分成诸多小事务完成呢？当然前提是业务允许，这就要求在前期设计的业务流程中应该有这种考虑。
以save方法为例：
``` java
public final <T extends AbstractEntity> T save(final ValidatePrincipal principal, final T entity) {

    ... // 资源权限验证

    return transactionTemplate.execute(new TransactionCallback<T>() {
        public T doInTransaction(TransactionStatus status) {
            return dataSourceManager.save(principal.getUser(), entity);
        }
    });
}
```
采用Spring JDBC提交事务，例如在一个流程中有多次保存资源操作，通常的事务提交动作发生在整个流程结束时，如果需要将整个流程切分成多个事务分别提交，使用TransactionTemplate无疑是一个简单、有效的方法。
根据业务的需要，DataSourceManager封装了Hibernate的save方法。代码片段如下：
``` java
public <T extends AbstractEntity> T save(String user, T entity, boolean flush) {

    /* 自动设置主键 */
    if (entity.getId() == null) {
        entity.setId(ID.next());
    }
    /* 初次插入；设置记录信息 */
    long now = System.currentTimeMillis();
    entity.setCreateBy(user);
    entity.setCreateTime(now);
    entity.setModifyBy(user);
    entity.setModifyTime(now);
    Session session = getCurrentSession(flush);
    session.save(entity);
    if (flush) {
        session.flush();
    }
    
    return entity;
}
```
以下是经验之谈。平台在大多数资源相关的数据库表中增加了createBy、createTime、updateBy、updateTime，这种设计给平台带来了极大的好处，可以很容易地找出是谁操作的资源及操作的时间点；如果出现异常，也可以较容易地定位现场；再配合审计流程，基本上就可以掌控整个平台的每一个动作，从而明确了责任归属，也为后期的计费系统做足了数据保证。
通过以上方式您可以随意构造任意大小的事务。

