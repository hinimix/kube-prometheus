# Ksonnet使用手册

源项目[ksonnet][ksonnet]

## 介绍
**ksonnet** mixin libraries 允许使用人把一个kubernetes程序分割成几个模块化的组件

比如, 一个开发小组的YAML文件可以分割成应用小分队和日志小分队来分别写各自的YAML文件.
可以让日志小分队的人先写yaml, 然后应用小分队添加并引用即可, 这样比写一个单个的YAML
文件好多了

在这里, 我们会探讨libraries如何构建, 用[fluentd][fluentd]的mixin库(官方仓库
[mixins repository][fluentd-mixin]).
我们可以看到一个用[Elasticsearch][elastic]的团队如何使用Fluentd的mixin库轻松
的配置Fluentd去tail Elasticsearch的日志, 然后传输给Kibana提交给Dashboard


* 点击[the Elasticwebsite][elastic] 获取`Elasticsearch`, `Kibana`相关概念
* 点击[the Fluentd website][fluentd], 获取`fluentd`相关概念


## 构建mixin的需求
如果想要构建自己的mixin库, 或者想在**ksonnet**里使用内建的mixins, 需要执行以下标
星任务

* 安装 **Jsonnet**, version 0.9.4 or later
* 本地 clone **ksonnet** repository
* 安装和配置Visual Studio Code extension (可选的)
* 建立一个测试的kubernetes集群

## 设计和架构

办法是让Elasticsearch程序的发送日志到标准输出, 然后Fluentd tail这些日志后发送
搭配Kibana呈现出来

在Kubernetes里, 访问`Pod`的日志涉及以下几个条件
* 让Fluentd有权限去访问`Pod`日志
* 把有`Pod`日志的存储卷追加到Fluentd的容器里, 以便Fluentd访问

这里会详细介绍example的关键细节, 在更高的角度上, 这个流程被分解为
* `DaemonSet`: 部署在每一个主机上, 以便于tail `Pod`日志, 无论
  Elasticsearch运行在哪个主机上, 都可以被tail

  这个`DaemonSet`只包含核心Fluentd程序定义, 它没有权限去访问`Pod`日志, 也没有挂载可以访问的存储卷
  
* 一个单独的mixin定义了`DaemonSet` 需要访问 `Pod` 日志的 `VolumeMounts` 和 `Volumes`
* 一个单独的mixin定义了`DaemonSet`的访问权限
* 一个RBAC对象, 集群管理员需要发送给集群认证,有权限访问`Pod` 日志

这套用法分割了不同部分的关注点:
开发人员可以定义`DaemonSet`,
集群管理员定义权限, 或者其他`DaemonSet`用到的权限,
在不用重载整个集群配置的情况下, `DeamonSet`或者访问权限可以被单独修改
在这里, 我们定义的`DaemonSet`信息可以在不调整基础`DaemonSet`定义情
况下定义`Volumes`和`VolumeMounts`


### 定义mixins去配置访问pod日志

来看一下我们如何解耦一个完整的Fluentd配置, 让日志小分队只写Fluentd的
DaemonSet核心, 然后写一个**ksonnet**库让你去自定义关键细节

`fluentd-es-ds.jsonnet` 定义了一个基础的DaemonSet, 给它添加访问权限

```javascript
// daemonset
local ds =
  // base daemonset
  fluentd.app.daemonSetBuilder.new(config) +
  // add configuration for access to pod logs
  fluentd.app.daemonSetBuilder.configureForPodLogs(config);

 // create access permissions for pod logs
 local rbacObjs = fluentd.app.admin.rbacForPodLogs(config);
```
> 注意这个基础的DaemonSet什么都不能做, 它不知道什么pod日志是它需要的, 
> * 它需要Volumes和VolumeMount来提供日志信息在哪
> * 它还需要访问权限, RBAC
>
看看这东西还有什么优点

在`fluentd.libsonnet`, 我们定义mixin的`DaemonSet`, 
现在开始看**ksonnet**真正工作的样子, 

mixin指明了Fluentd需要的`VolumeMounts`和`Volumes`(和`DaemonSet`定义分离的), 
这让我们从部署细节上解耦了程序的定义

需要注意下列片段: `addHostMountedPodLogs`的参数`containerSelector`, 
我们传输了一个函数给`ds.mapContainers`来迭代我们自己的容器(这里是Fluentd容器), 
添加了容器需要的VolumeMounts, 
`Pod`的日志也从函数里被抽象出来


```javascript
  mixin:: {
    daemonSet:: {
      // Takes two volume names and produces a
      // mixin that mounts the Kubernetes pod logs into a set of
      // containers specified by `containerSelector`.
      addHostMountedPodLogs(
        varlogName, podlogsName, containerSelector=function(c) true
      )::
        local podLogs = $.parts.podLogs(varlogName, podlogsName);

        // Add volume to DaemonSet.
        ds.mixin.spec.template.spec.volumes([
          podLogs.varLogVolume,
          podLogs.podLogVolume,
        ]) +

        // Iterate over a specified set of containers to add the VolumeMounts
        ds.mapContainers(
          function (c)
            if containerSelector(c)
            then
              c + container.volumeMounts([
                podLogs.varLogMount,
                podLogs.podLogMount,
              ])
            else c),
    },
  },
```

是我们用`daemonSetBuilder`来创建DaemonSet叫做`daemonSet mixin` 
同时定义了DaemoSEt需要的`configureForPodLogs`方法
 
从上一个代码段里, 我们可以知道DaemonSet自身并不知道这些细节

```javascript
    daemonSetBuilder:: {
      new(config):: {
        toArray():: [self.daemonSet],
        daemonSet:: $.parts.daemonSet(config.daemonSet.name, config.container.name, config.container.tag, config.namespace)
      },

      // access configuration
      configureForPodLogs(
        config,
        varlogVolName="varlog",
        podLogsVolName="varlibdockercontainers",
      )::
        {} + {
          daemonSet+::
            $.mixin.daemonSet.addHostMountedPodLogs(
              varlogVolName,
              podLogsVolName,
              $.util.containerNameInSet(config.container.name)) +
            // RBAC and service account
            ds.mixin.spec.template.spec.serviceAccountName(config.rbac.accountName)
        },
    },
```

在前面代码段里, 我们注意到我们指明了ServiceAccount
现在该配置RBAC对象来赋予Fluentd的访问权限了.


### 定义 RBAC 对象
单独的定义RBAC对象能分开单独管理, 这让集群管理人员和程序开发人员分开工作,
集群管理人员可以用很少几行来确定和定义应用程序的访问权限,

定义Kubernetes的访问权限的RBAC对象封装在下列定义里`fluentd.libsonnet`


```javascript
    admin:: {
      rbacForPodLogs(config)::
        $.parts.rbac(config.rbac.accountName, config.namespace),
    },
```


打开这个代码段: 

`fluentd.libsonnet` 定义了所有需要的RBAC对象. 
注意我们分别抽象了ServiceAccount的属性, 在单独的`config`对象里指定他们的值, 

这种方法能让我们确定ServiceAccount是不是正确关联了需要的对象


```javascript
    rbac(name, namespace)::
      local metadata = svcAccount.mixin.metadata.name(name) +
        svcAccount.mixin.metadata.namespace(namespace);

      local hcServiceAccount = svcAccount.new() +
        metadata;

      local hcClusterRole =
        clRole.new() +
        metadata +
        clRole.rules(
          rule.new() +
          rule.apiGroups("*") +
          rule.resources(["pods", "nodes"]) +
          rule.verbs(["list", "watch"])
        );

      local hcClusterRoleBinding =
        clRoleBinding.new() +
        metadata +
        clRoleBinding.mixin.roleRef.apiGroup("rbac.authorization.k8s.io") +
        clRoleBinding.mixin.roleRef.name(name) +
        clRoleBinding.mixin.roleRef.mixinInstance({kind: "ClusterRole"}) +
        clRoleBinding.subjects(
          subject.new() +
          subject.name(name) +
          subject.namespace(namespace)
          {kind: "ServiceAccount"}
        );

```

在 `fluentd-es-ds.jsonnet` 里我们这样定义:

```javascript
local config = {
  namespace:: "elasticsearch",
  container:: {
    name:: "fluentd-es",
    tag:: "1.22",
  },
  daemonSet:: {
    name:: "fluentd-es-v1.22",
  },
  rbac:: {
    accountName:: "fluentd-serviceaccount"
  },
};
```

我们可以用rbac方法字段调用`namespace` 和 `AccountName`, 

## 总结

我们从最开始, 到建了一个简单的DaemonSet, 它的pod日志和它的访问权限

你看到了日志和权限是分开, 我们这样定义不需要重写整个配置


```javascript
// daemonset
local ds =
  // base daemonset
  fluentd.app.daemonSetBuilder.new(config) +
  // add configuration for access to pod logs
  fluentd.app.daemonSetBuilder.configureForPodLogs(config);

 // create access permissions for pod logs
 local rbacObjs = fluentd.app.admin.rbacForPodLogs(config);
```

## 更近一步

这个例子也包含了生成JSON files, 帮助我们理解了 **ksonnet**的细节, 
分解和抽象了完整的配置

既然你要写一个自定义的mixins, 可以试试如何导入一个最小的组件来操作

And feel free to contribute your own examples to our mixins repository!

[readme]: ../../readme.md "ksonnet readme"
[fluentd-mixin]: https://github.com/ksonnet/mixins/tree/master/incubator/fluentd
[fluentd]: http://www.fluentd.org/architecture
[elastic]: https://www.elastic.co/products
[ksonnet]: https://github.com/ksonnet/ksonnet