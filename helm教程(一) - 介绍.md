---
title: helm教程(一) - 介绍
date: 2022-05-24
tags:
    - Kubernetes
categories:
    - Kubernetes
---



# Helm介绍

## Helm chart目录结构

```shell
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```

`templates/` 目录包括了模板文件。当Helm评估chart时，会通过模板渲染引擎将所有文件发送到`templates/`目录中。 然后收集模板的结果并发送给Kubernetes。

`values.yaml` 文件也导入到了模板。这个文件包含了chart的 *默认值* 。这些值会在用户执行`helm install` 或 `helm upgrade`时被覆盖。

`Chart.yaml` 文件包含了该chart的描述。你可以从模板中访问它。`charts/`目录 *可以* 包含其他的chart(称之为 *子chart*)。 



## Helm 命令介绍

**安装chart**

```
$ helm install {.Release.Name} ./mychart
NAME: {.Release.Name}
LAST DEPLOYED: Tue Nov  1 17:36:01 2016
NAMESPACE: default
STATUS: DEPLOYED
REVISION: 1
TEST SUITE: None
```

**卸载chart**

```
$ helm uninstall {.Release.Name}
release "{.Release.Name}" uninstalled
```

**查看实际加载的模板**

```
$ helm get manifest {.Release.Name}

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

**测试模板渲染的内容但又不想安装任何实际应用**

```
$ helm install --debug --dry-run {.Release.Name} ./mychart
```



## 内置对象

对象可以是非常简单的:仅有一个值。或者可以包含其他对象或方法。比如，`Release`对象可以包含其他对象（比如：`Release.Name`）和`Files`对象有一组方法。

**`Release`: 对象描述了版本发布本身**。

- `Release.Name`： release名称
- `Release.Namespace`： 版本中包含的命名空间(如果manifest没有覆盖的话)
- `Release.IsUpgrade`： 如果当前操作是升级或回滚的话，该值将被设置为`true`
- `Release.IsInstall`： 如果当前操作是安装的话，该值将被设置为`true`
- `Release.Revision`： 此次修订的版本号。安装时是1，每次升级或回滚都会自增
- `Release.Service`： 该service用来渲染当前模板。Helm里始终`Helm`

**`Values`: 对象是从`values.yaml`文件和用户提供的文件传进模板的。**

**`Chart`: 对象是`Chart.yaml`文件内容。**在 [Chart 指南](https://helm.sh/zh/docs/topics/charts#Chart-yaml-文件) 中列出了可获得属性

**`Files`： 在chart中提供访问所有的非特殊文件的对象。你不能使用它访问`Template`对象，只能访问其他文件。**

- `Files.Get` 通过文件名获取文件的方法。 （`.Files.Getconfig.ini`）
- `Files.GetBytes` 用字节数组代替字符串获取文件内容的方法。 对图片之类的文件很有用
- `Files.Glob` 用给定的shell glob模式匹配文件名返回文件列表的方法
- `Files.Lines` 逐行读取文件内容的方法。迭代文件中每一行时很有用
- `Files.AsSecrets` 使用Base 64编码字符串返回文件体的方法
- `Files.AsConfig` 使用YAML格式返回文件体的方法

**`Capabilities`： 提供关于Kubernetes集群支持功能的信息**

- `Capabilities.APIVersions` 是一个版本列表
- `Capabilities.APIVersions.Has $version` 说明集群中的版本 (比如,`batch/v1`) 或是资源 (比如, `apps/v1/Deployment`) 是否可用
- `Capabilities.KubeVersion` 和`Capabilities.KubeVersion.Version` 是Kubernetes的版本号
- `Capabilities.KubeVersion.Major` Kubernetes的主版本
- `Capabilities.KubeVersion.Minor` Kubernetes的次版本
- `Capabilities.HelmVersion` 包含Helm版本详细信息的对象，和 `helm version` 的输出一致
- `Capabilities.HelmVersion.Version` 是当前Helm语义格式的版本
- `Capabilities.HelmVersion.GitCommit` Helm的git sha1值
- `Capabilities.HelmVersion.GitTreeState` 是Helm git树的状态
- `Capabilities.HelmVersion.GoVersion` 是使用的Go编译器版本

**`Template`： 包含当前被执行的当前模板信息**

- `Template.Name`: 当前模板的命名空间文件路径 (e.g. `mychart/templates/mytemplate.yaml`)
- `Template.BasePath`: 当前chart模板目录的路径 (e.g. `mychart/templates`)



## Values 文件

**可以获取Values的位置：**

- chart中的`values.yaml`文件
- 如果是子chart，就是父chart中的`values.yaml`文件
- 使用`-f`参数(`helm install -f myvals.yaml ./mychart`)传递到 `helm install` 或 `helm upgrade`的values文件
- 使用`--set` (比如`helm install --set foo=bar ./mychart`)传递的单个参数

以上列表有明确顺序：默认使用`values.yaml`，可以被父chart的`values.yaml`覆盖，继而被用户提供values文件覆盖， 最后会被`--set`参数覆盖，优先级为`values.yaml`最低，`--set`参数最高。
