---
title: "ShardingSphere-Proxy启动和分布式治理接入机制初探"
date: Wed May 19 17:44:20 CST 2021
categories: 中间件
tags: 中间件
draft: false
---

#### start.sh

先看看官方提供的启动脚本`start.sh`

```bash
#!/bin/bash

SERVER_NAME=ShardingSphere-Proxy

cd `dirname $0`  #进入当前Shell程序的目录
cd ..
DEPLOY_DIR=`pwd` #根目录

LOGS_DIR=${DEPLOY_DIR}/logs  #定义日志目录
if [ ! -d ${LOGS_DIR} ]; then
    mkdir ${LOGS_DIR}
fi

STDOUT_FILE=${LOGS_DIR}/stdout.log
EXT_LIB=${DEPLOY_DIR}/ext-lib  #拓展类存放目录（可选）

CLASS_PATH=.:${DEPLOY_DIR}/lib/*:${EXT_LIB}/*

JAVA_OPTS=" -Djava.awt.headless=true -Djava.net.preferIPv4Stack=true "

JAVA_MEM_OPTS=" -server -Xmx2g -Xms2g -Xmn1g -Xss256k -XX:+DisableExplicitGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:LargePageSizeInBytes=128m -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70 "

MAIN_CLASS=org.apache.shardingsphere.proxy.Bootstrap  #定义启动类

print_usage() {
    echo "usage: start.sh [port] [config_dir]"
    echo "  port: proxy listen port, default is 3307"
    echo "  config_dir: proxy config directory, default is conf"
    exit 0
}

if [ "$1" == "-h" ] || [ "$1" == "--help" ] ; then
    print_usage
fi

echo "Starting the $SERVER_NAME ..."

if [ $# == 1 ]; then  #一个参数表示指定端口
    MAIN_CLASS=${MAIN_CLASS}" "$1
    echo "The port is $1"
    set CLASS_PATH=../conf;%CLASS_PATH%  #把配置目录也加入CLASS_PATH
fi

if [ $# == 2 ]; then  #第二个参数指定配置文件
    MAIN_CLASS=${MAIN_CLASS}" "$1" "$2
    echo "The port is $1"
    echo "The configuration path is $DEPLOY_DIR/$2"
    CLASS_PATH=${DEPLOY_DIR}/$2:${CLASS_PATH}  #把配置目录也加入CLASS_PATH
fi

echo "The classpath is ${CLASS_PATH}"

nohup java ${JAVA_OPTS} ${JAVA_MEM_OPTS} -classpath ${CLASS_PATH} ${MAIN_CLASS} >> ${STDOUT_FILE} 2>&1 &
sleep 1
echo "Please check the STDOUT file: $STDOUT_FILE"
```

这里指定了启动类`org.apache.shardingsphere.proxy.Bootstrap`

#### Bootstrap

```java
		public static void main(final String[] args) throws IOException, SQLException {
  			// 这里BootstrapArguments会设置好端口和配置文件目录，若无指定则为默认
        BootstrapArguments bootstrapArgs = new BootstrapArguments(args);
  			// 具体如何加载yaml见下面ProxyConfigurationLoader.load方法
        YamlProxyConfiguration yamlConfig = ProxyConfigurationLoader.load(bootstrapArgs.getConfigurationPath());
      	// 重点在init方法
        createBootstrapInitializer(yamlConfig).init(yamlConfig, bootstrapArgs.getPort());
    }
    
		// 根据配置中是否有governance来决定使用哪个BootstrapInitializer
    private static BootstrapInitializer createBootstrapInitializer(final YamlProxyConfiguration yamlConfig) {
        return null == yamlConfig.getServerConfiguration().getGovernance() ? new StandardBootstrapInitializer() : new GovernanceBootstrapInitializer();
    }
```

#### ProxyConfigurationLoader

load()

```java
		public static YamlProxyConfiguration load(final String path) throws IOException {
      	// 加载server.yml文件
        YamlProxyServerConfiguration serverConfig = loadServerConfiguration(getResourceFile(String.join("/", path, SERVER_CONFIG_FILE)));
      	File configPath = getResourceFile(path);
      	// 把各种配置类加载进来（先进行一定检查）
        Collection<YamlProxyRuleConfiguration> ruleConfigurations = loadRuleConfigurations(configPath);
      	// 如果没有配置类，或者没有治理模块的配置类则抛出异常
        Preconditions.checkState(!ruleConfigurations.isEmpty() || null != serverConfig.getGovernance(), "Can not find any valid rule configurations file in path `%s`.", configPath.getPath());
        return new YamlProxyConfiguration(serverConfig, ruleConfigurations.stream().collect(Collectors.toMap(
                YamlProxyRuleConfiguration::getSchemaName, each -> each, (oldValue, currentValue) -> oldValue, LinkedHashMap::new)));
    }
```

getResourceFile()

```java
		private static File getResourceFile(final String path) {
        URL url = ProxyConfigurationLoader.class.getResource(path);
        return null == url ? new File(path) : new File(url.getFile());
    }
```

loadServerConfiguration()

```java
    private static YamlProxyServerConfiguration loadServerConfiguration(final File yamlFile) throws IOException {
      	// YamlEngine通过org.yaml.snakeyaml.Yaml把yaml文件加载进来
      	// 这个地方要关注一下YamlProxyServerConfiguration类，后面的task应该要用到
        YamlProxyServerConfiguration result = YamlEngine.unmarshal(yamlFile, YamlProxyServerConfiguration.class);
        Preconditions.checkNotNull(result, "Server configuration file `%s` is invalid.", yamlFile.getName());
        Preconditions.checkState(!result.getUsers().isEmpty() || null != result.getGovernance(), "Authority configuration is invalid.");
        return result;
    }
```

#### 下面关注一下init方法

因为任务需要，这里只关注GovernanceBootstrapInitializer

下面的代码来自AbstractBootstrapInitializer

```java
@Override
public final void init(final YamlProxyConfiguration yamlConfig, final int port) throws SQLException {
  	// 这里调用的getProxyConfiguration由子类实现
    ProxyConfiguration proxyConfig = getProxyConfiguration(yamlConfig);
  	// TO LOOK
    MetaDataContexts metaDataContexts = decorateMetaDataContexts(createMetaDataContexts(proxyConfig));
    for (MetaDataAwareEventSubscriber each : ShardingSphereServiceLoader.getSingletonServiceInstances(MetaDataAwareEventSubscriber.class)) {
        each.setMetaDataContexts(metaDataContexts);
        ShardingSphereEventBus.getInstance().register(each);
    }
    String xaTransactionMangerType = metaDataContexts.getProps().getValue(ConfigurationPropertyKey.XA_TRANSACTION_MANAGER_TYPE);
    TransactionContexts transactionContexts = decorateTransactionContexts(createTransactionContexts(metaDataContexts), xaTransactionMangerType);
    ProxyContext.getInstance().init(metaDataContexts, transactionContexts);
    setDatabaseServerInfo();
    initScalingWorker(yamlConfig);
    shardingSphereProxy.start(port);
}
```

下面是上面调用的GovernanceBootstrapInitializer.getProxyConfiguration()

```java
private final GovernanceFacade governanceFacade = new GovernanceFacade();

@Override
protected ProxyConfiguration getProxyConfiguration(final YamlProxyConfiguration yamlConfig) {
    // 这里就是对governanceFacade进行了初始化
  	governanceFacade.init(new GovernanceConfigurationYamlSwapper().swapToObject(yamlConfig.getServerConfiguration().getGovernance()), yamlConfig.getRuleConfigurations().keySet());
    // 这里是重点，会与governance模块进行交互
  	initConfigurations(yamlConfig);
    return loadProxyConfiguration();
}
```

下面是GovernanceBootstrapInitializer.initConfigurations方法

```java
private void initConfigurations(final YamlProxyConfiguration yamlConfig) {
    YamlProxyServerConfiguration serverConfig = yamlConfig.getServerConfiguration();
    Map<String, YamlProxyRuleConfiguration> ruleConfigs = yamlConfig.getRuleConfigurations();
    // 如果规则配置为空，且服务配置的用户、props都为空
  	if (isEmptyLocalConfiguration(serverConfig, ruleConfigs)) {
        governanceFacade.onlineInstance();
    } else { // 否则
        governanceFacade.onlineInstance(getDataSourceConfigurationMap(ruleConfigs), 
                getRuleConfigurations(ruleConfigs), YamlUsersConfigurationConverter.convertShardingSphereUser(serverConfig.getUsers()), serverConfig.getProps());
    }
}
```

**划重点：GovernanceFacade.onlineInstance**

onlineInstace方法有两个

```java
public void onlineInstance(final Map<String, Map<String, DataSourceConfiguration>> dataSourceConfigMap,
                           final Map<String, Collection<RuleConfiguration>> schemaRuleMap, final Collection<ShardingSphereUser> users, final Properties props) {
  	// 将users和props写进注册中心（需要isOverwrite为true）
    registryCenter.persistGlobalConfiguration(users, props, isOverwrite);
  	// 将DataSourceConfiguration中的每一项都写入注册中心（需要isOverwrite为true）
    for (Entry<String, Map<String, DataSourceConfiguration>> entry : dataSourceConfigMap.entrySet()) {
        registryCenter.persistConfigurations(entry.getKey(), dataSourceConfigMap.get(entry.getKey()), schemaRuleMap.get(entry.getKey()), isOverwrite);
    }
    onlineInstance();
}
```

```java
public void onlineInstance() {
    registryCenter.persistInstanceOnline();
    registryCenter.persistDataNodes();
    registryCenter.persistPrimaryNodes();
    listenerManager.init();
}
```

#### 从这里开始进入了governance模块

#### RegistryCenter类

persistGlobalConfiguration方法有两个，onlineInstace首先调用的是下面这个。

```java
public void persistGlobalConfiguration(final Collection<ShardingSphereUser> users, final Properties props, final boolean isOverwrite) {
    persistUsers(users, isOverwrite);
    persistProperties(props, isOverwrite);
}
```

```java
private void persistUsers(final Collection<ShardingSphereUser> users, final boolean isOverwrite) {
    if (!users.isEmpty() && (isOverwrite || !hasUsers())) {
      	// 持久化到注册中心
        repository.persist(node.getUsersNode(), YamlEngine.marshal(YamlUsersConfigurationConverter.convertYamlUserConfigurations(users)));
    }
}
```

persist是由governance-repository-api模块的GovernanceRepository接口定义的一个方法，ss在governance-repository-provider的Zeekeeper和Etcd这两个类中对persist进行了实现。

下面来看看ss对zookeeper的persist的具体实现。

```java
@Override
public void persist(final String key, final String value) {
    try {
        if (!isExisted(key)) {
            client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(key, value.getBytes(StandardCharsets.UTF_8));
        } else {
            update(key, value);
        }
        // CHECKSTYLE:OFF
    } catch (final Exception ex) {
        // CHECKSTYLE:ON
        CuratorZookeeperExceptionHandler.handleException(ex);
    }
}
```

这个地方对于zk的操作用的是CuratorFramework框架

所以如果我们要在shardingshpere中与注册中心交互的话，只需要去调用RegistryRepository接口的一系列方法即可。

#### registryListenerManager.initListeners

前面的onlineInstance方法最后一个是对监听事件进行初始化，主要是通过registryListenerManager.init方法进行的。

```java
/**
 * Initialize all state changed listeners.
 */
public void initListeners() {
    terminalStateChangedListener.watch(Type.UPDATED);
    dataSourceStateChangedListener.watch(Type.UPDATED, Type.DELETED, Type.ADDED);
    lockChangedListener.watch(Type.ADDED, Type.DELETED);
    metaDataListener.watch();
    propertiesChangedListener.watch(Type.UPDATED);
    authenticationChangedListener.watch(Type.UPDATED);
    privilegeNodeChangedListener.watch(Type.UPDATED);
}
```

下面是PostGovernanceRepositoryEventListener抽象类的两个watch方法。


```java
@Override
public final void watch(final Type... types) {
    Collection<Type> typeList = Arrays.asList(types);
    for (String watchKey : watchKeys) {
        watch(watchKey, typeList);
    }
}

private void watch(final String watchKey, final Collection<Type> types) {
    governanceRepository.watch(watchKey, dataChangedEvent -> {
        if (types.contains(dataChangedEvent.getType())) {
            Optional<T> event = createEvent(dataChangedEvent);
            event.ifPresent(ShardingSphereEventBus.getInstance()::post);
        }
    });
}
```

这里的governanceRepository.watch是由的xxxRepository实现的，下面看一看CuratorZookeeperRepository是怎么实现。

```java
@Override
public void watch(final String key, final DataChangedEventListener listener) {
    String path = key + PATH_SEPARATOR;
    if (!caches.containsKey(path)) {
        addCacheData(key);
      	// 使用了基于Cache的事件监听
        CuratorCache cache = caches.get(path);
        cache.listenable().addListener((type, oldData, data) -> {
            String eventPath = CuratorCacheListener.Type.NODE_DELETED == type ? oldData.getPath() : data.getPath();
            byte[] eventDataByte = CuratorCacheListener.Type.NODE_DELETED == type ? oldData.getData() : data.getData();
            Type changedType = getChangedType(type);
            if (Type.IGNORED != changedType) {
              	// 这里的onChange会在时间发生的时候触发，见下面接口的注释
                listener.onChange(new DataChangedEvent(eventPath, null == eventDataByte ? null : new String(eventDataByte, StandardCharsets.UTF_8), changedType));
            }
        });
    }
}
```

```java
/**
 * Listener for data changed.
 */
public interface DataChangedEventListener {
    
    /**
     * Fire when data changed.
     * 
     * @param event data changed event
     */
    void onChange(DataChangedEvent event);
}
```

而初始化时，对事件的处理如下（上面贴过了）

```java
private void watch(final String watchKey, final Collection<Type> types) {
    governanceRepository.watch(watchKey, dataChangedEvent -> {
      	// Lamda实现onChange方法
        if (types.contains(dataChangedEvent.getType())) {
            Optional<T> event = createEvent(dataChangedEvent);
          	// 懒汉式单例模式创建ShardingSphereEventBus，把事件推送给订阅者
            event.ifPresent(ShardingSphereEventBus.getInstance()::post);
        }
    });
}
```

至此，接入服务治理的proxy的初始化完成。那么，ShardingSphereEventBus是在哪里处理的呢？

proxy对注册中心进行了监听，若有数据发生变化，则包装成相应的event类，然后post进ShardingSphereEventBus。这时，带@Subscribe注解的相应方法会被激活。RegistryCenter类中有很多被@Subscribe标记的方法，但都没有对DataChangedEvent进行处理。
