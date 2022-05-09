## 一、简述

Nacos实现了服务的配置中心与服务注册发现的功能，Nacos可以通过可视化的配置降低相关的学习与维护成本，实现动态的配置管理与分环境的配置中心控制。同时Nacos提供了基于http/RCP的服务注册与发现功能。

![](../img/spring/springcloudalibaba/nacos.png)

注册的信息会存放在 map 中，而且还是个两层的 `ConcurrentHashMap<String, Map<String, Service>>`，外层 map 的 key 是个 namespace，value 是个Map<String, Service>，内层 map 的 key 是 group::serviceName，value是个 service 类。

```java

private String token;
private List<String> owners = new ArrayList<>();
private Boolean resetWeight = false;
private Boolean enabled = true;
private Selector selector = new NoneSelector();
private String namespaceId;
private Map<String, Cluster> clusterMap = new HashMap<String, Cluster>();
```

