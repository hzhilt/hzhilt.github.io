---
layout: post
keywords: Start
description:  realm 初探(一)
title:  realm 初探(一)
categories: [技术]
tags: [技术]
group: archive
icon: globe
---


## 背景

先来聊聊realm的历史，realm为移动设备而生，其急速的开发速度可以省下码农来喝茶泡妞。他的愿望是替代sqlite和core data。目前很多大公司正在使用例如高朋，sap，intel，google，amazon，ebay，alibaba（好吓人，我估计就是里面一两个小team，预研式的使用而已，目前推广也是不唯余力啊！）。

realm的特点是简单易用（分分钟的事情），跨平台（ios，android，react-native），速度快（号称sqlite快），更多的支持。

那么如果要学的话，只需要打开[realm java快速入门](https://realm.io/cn/docs/java/latest/)就ok了，上手真的很快喔。能在开发文档提到的我就不bb了，官方提供的demo也非常详细。

那么我们简单的学了一个demo之后，我们就要思考，realm如何读写数据，有哪些不友好的地方。   

### 实战解说

我们先拿到Realm的一个实例
有一种方法，但是最后都是调用到
Realm.getInstance(),默认的数据库名字是DEFAULT_REALM_NAME = "default.realm"，
其中有一个重要的参数就是RealmConfiguration，这个配置是用来生成一个制定的Realm实例的，也就是说，每一个数据库都是有不同的配置，需要你在实际开发中处理。好吧，如果我们只有一个默认的数据库，那么，我们这样好了Realm.getDefaultInstance()，生成默认配置的realm供你使用。

```
    public static Realm getInstance(RealmConfiguration configuration) {
        if (configuration == null) {
            throw new IllegalArgumentException("A non-null RealmConfiguration must be provided");
        }
        return RealmCache.createRealmOrGetFromCache(configuration, Realm.class);
    }
```

下面我们来到了
RealmCache.createRealmOrGetFromCache(defaultConfiguration, Realm.class);

Realm缓存了realm的实例和相关的资源，同一个线程共享使用相同配置的realm。
那个createRealmOrGetFromCache，估计就是如果有缓存就从缓存中那，没有的滑就新建一个Realm。

```
    static synchronized <E extends BaseRealm> E createRealmOrGetFromCache(RealmConfiguration configuration,
                                                        Class<E> realmClass) {
        boolean isCacheInMap = true;
        RealmCache cache = cachesMap.get(configuration.getPath());
        if (cache == null) {
            // Create a new cache
            cache = new RealmCache(configuration);
            // The new cache should be added to the map later.
            isCacheInMap = false;
        } else {
            // Throw the exception if validation failed.
            cache.validateConfiguration(configuration);
        }

        RefAndCount refAndCount = cache.refAndCountMap.get(RealmCacheType.valueOf(realmClass));

        if (refAndCount.localRealm.get() == null) {
            // Create a new local Realm instance
            BaseRealm realm;

            if (realmClass == Realm.class) {
                // RealmMigrationNeededException might be thrown here.
                realm = Realm.createInstance(configuration, cache.typedColumnIndices);
            } else if (realmClass == DynamicRealm.class) {
                realm = DynamicRealm.createInstance(configuration);
            } else {
                throw new IllegalArgumentException(WRONG_REALM_CLASS_MESSAGE);
            }

            // The Realm instance has been created without exceptions. Cache and reference count can be updated now.

            // The cache is not in the map yet. Add it to the map after the Realm instance created successfully.
            if (!isCacheInMap) {
                cachesMap.put(configuration.getPath(), cache);
            }
            refAndCount.localRealm.set(realm);
            refAndCount.localCount.set(0);
        }

        Integer refCount = refAndCount.localCount.get();
        if (refCount == 0) {
            if (realmClass == Realm.class && refAndCount.globalCount == 0) {
                cache.typedColumnIndices = refAndCount.localRealm.get().schema.columnIndices;
            }
            // This is the first instance in current thread, increase the global count.
            refAndCount.globalCount++;
        }
        refAndCount.localCount.set(refCount + 1);

        @SuppressWarnings("unchecked")
        E realm = (E) refAndCount.localRealm.get();
        return realm;
    }
```

看看缓存有没有线程相关的realm信息，没有的话，new 一个 基类realm。
这个时候有一个DynamicRealm 是realm的表哥，使用的数据都是string类型的数据库。（先不管）

来了，我们要创建一个realm了
Realm.createInstance(RealmConfiguration configuration, ColumnIndices columnIndices),前一个是配置信息，后面一个参数从配置文件读取或者自动生成。

```
static Realm createInstance(RealmConfiguration configuration, ColumnIndices columnIndices) {
        try {
            return createAndValidate(configuration, columnIndices);

        } catch (RealmMigrationNeededException e) {
            if (configuration.shouldDeleteRealmIfMigrationNeeded()) {
                deleteRealm(configuration);
            } else {
                migrateRealm(configuration);
            }

            return createAndValidate(configuration, columnIndices);
        }
    }
```

createAndValidate(configuration, columnIndices)
这里捕获一个RealmMigrationNeededException。
好累啊，什么时候才到头啊！

这个时候有创建了一个新的realm，
Realm realm = new Realm(configuration, autoRefresh);
在这判断版本信息，不表。

initializeRealm(realm);

```
private static void initializeRealm(Realm realm) {
        long version = realm.getVersion();
        boolean commitNeeded = false;
        try {
            realm.beginTransaction();
            if (version == UNVERSIONED) {
                commitNeeded = true;
                realm.setVersion(realm.configuration.getSchemaVersion());
            }

            RealmProxyMediator mediator = realm.configuration.getSchemaMediator();
            final Set<Class<? extends RealmObject>> modelClasses = mediator.getModelClasses();
            final Map<Class<? extends RealmObject>, ColumnInfo> columnInfoMap;
            columnInfoMap = new HashMap<Class<? extends RealmObject>, ColumnInfo>(modelClasses.size());
            for (Class<? extends RealmObject> modelClass : modelClasses) {
                // Create and validate table
                if (version == UNVERSIONED) {
                    mediator.createTable(modelClass, realm.sharedGroupManager.getTransaction());
                }
                columnInfoMap.put(modelClass, mediator.validateTable(modelClass, realm.sharedGroupManager.getTransaction()));
            }
            realm.schema.columnIndices = new ColumnIndices(columnInfoMap);
        } finally {
            if (commitNeeded) {
                realm.commitTransaction();
            } else {
                realm.cancelTransaction();
            }
        }
    }
```

创建一个RealmProxyMediator（realm代理媒介），特定配置的实例媒介。
保存配置中的每一个model，和每个model的列。

然后通过mediator创建数据库。
mediator.createTable(modelClass, realm.sharedGroupManager.getTransaction());

RealmProxyMediator是抽象类，它有两个实现，分别为RealmProxyMediator和FilterableMediator

RealmProxyMediator是合并两个不同的RealmProxyMediator；
FilterableMediator是通过过滤器指定过滤的可用的类。

到这里我们就创建了一个table了。
然而怎么创建的table，realm却没有告诉我们，这也是realm的核心技术。

这一节先讲到这里，下一部分将会继续深入的剖析realm的结构。



