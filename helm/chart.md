## 定义

> Chart是Helm软件包的格式。chart 是描述相关的一组 Kubernetes 资源的文件集合。单个 chart 可能用于部署简单的东西，比如 memcached pod，或者一些复杂的东西，比如完整的具有 HTTP 服务，数据库，缓存等的 Web 应用程序。

## Chart文件结构

> 一个Chart是一个目录下的文件集合。目录的名字就是chart的名字(不包含版本信息)。例如，WordPress的chart存储在 ```/WordPress```目录下：

helm期望的目录结构如下：
```
wordpress/
    Chart.yaml          # yaml文件，包含chart的信息
    LICENSE             # 可选的: A plain text file containing the license for the chart
    README.md           # 可选的: A human-readable README file
    requirements.yaml   # 可选的: yaml文件，chart的依赖列表
    values.yaml         # 该chart的默认配置参数
    charts/             # 包含该chart依赖的charts
    templates/          # templates目录，和values共同生成k8s的有效清单文件
    templates/NOTES.txt # 可选的: A plain text file containing short usage notes
```
Helm 保留使用 charts / 和 templates / 目录以及上面列出的文件名称。其他

### Chart.yml

> Chart.yml文件包含chart的元信息,包含下列字段：

```
apiVersion:     chart的API版本，指定是'v1'(required)          
name:           chart名字(required)
version:        SemVer格式的chart版本 (required)
kubeVersion:    可兼容的K8s版本 (optional)
description:    说明 (optional)
keywords:
  - xx          该工程的关键字 (optional)
home:           项目首页url(optional)
sources:
  - xx          项目源码的url列表 (optional)
maintainers:    # (optional)
  - name:       The maintainer's name (required for each maintainer)
    email:      The maintainer's email (optional for each maintainer)
    url:        A URL for the maintainer (optional for each maintainer)
engine: gotpl   template引擎，默认是go模板 (optional, defaults to gotpl)
icon: xx        icon的url (optional).
appVersion:     chart包含的app版本(optional). This needn't be SemVer.
deprecated:     chart是否过时(optional, boolean)
tillerVersion:  chart使用的tiller版本， This should be expressed as a SemVer range: ">2.0.0" (optional)
```
* Charts版本

    version字段，必须是SemVer 2标准格式的。version字段在Helm客户端和Tiller服务端都有用到，当生成一个软件包时，```helm package```命令会按照Chart.yml文件中的version字段来命名软件包名字。

    例如nginx的chart的version: 1.2.3，则生成：nginx-1.2.3.tgz

* appVersion字段

    和version字段没有任何关系，是用来指定Chart中的应用的版本的。

### LICENSE, README AND NOTES

* LICENSE
* README.md
* templates/NOTES.txt
  
    当使用```helm install```, ```helm status```会展示该NOTES.txt的内容

### Chart的依赖

> 一个chart可以依赖任意数量的其他charts，这些依赖可以通过requirement.yaml动态链接或者在charts/目录下手动添加。

**Note**: 在Chart.yml中使用dependencies:在2.16后已完全移除，官方提倡使用requirement.yaml。

* requirements.yaml

    ```
    dependencies:
      - name: nginx-ingress
        version: 1.4.0
        repository: http://yq01-aip-aikefu16a7e2d8.yq01:8879
      - name: xx
        version: 0.1.0
        repository: aa.ada.sa
    ```
    * name: 需要的chart的name
    * version:对应的版本
    * repository: chart所在repo的地址, 可以使用```helm repo add```加入到本地

    可以使用```helm dependency update [Chart-name]```, 将对应的依赖下载到charts目录下

    ![](./images/helmdepup.png)

    附加参数：
      * alias：别名，对于使用相同的chart，可以指定别名来用多个
      * tags和condition：指定子chart是否能用的两个标签，condition是对tags的重写；详见https://v2.helm.sh/docs/developing_charts/#tags-and-condition-fields-in-requirements-yaml
      * import-values和exports：import-values是父chart的requirements.yaml中的参数，exports是子chart的values.yaml中的参数

* charts/目录

> 可以在charts目录下放置压缩或者解压文件，依赖chart的名字不能以_或者.开头，这种会被helm忽略。

```helm fetch```可以删除一个chart的依赖

### Templates和Values

> helm模板使用go语言模板书写，并且从Sprig库中添加了约50种附加模板功能以及其他一些专用功能.所有的模板文件放在 templates/目录下。当渲染charts时，helm将使用template引擎浏览目录中的每个文件。

> 模板的Values值有两种提供方式:

  * chart中的values.yaml文件，可以设置一些默认值
  * 使用helm install 在命令行中指定

> 当用户自定义值时，这些值将会覆盖charts的values.yaml文件

#### Templates 

template结构实例：
  ```
  apiVersion: v1
  kind: ReplicationController
  metadata:
    name: deis-database
    namespace: deis
    labels:
      app.kubernetes.io/managed-by: deis
  spec:
    replicas: 1
    selector:
      app.kubernetes.io/name: deis-database
    template:
      metadata:
        labels:
          app.kubernetes.io/name: deis-database
      spec:
        serviceAccount: deis-database
        containers:
          - name: deis-database
            image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
            imagePullPolicy: {{.Values.pullPolicy}}
            ports:
              - containerPort: 5432
            env:
              - name: DATABASE_STORAGE
                value: {{default "minio" .Values.storage}}
  ```
这是一个k8s的RC模板实例，他可以使用以下四个模板值(通常定义在Values.yaml中)：
  * imageRegistry：docker镜像的注册地址
  * dockerTag：docker镜像版本
  * pullPolicy：k8s拉取策略
  * storage：存储，默认为minio

上述所有值由template作者定义的，helm不需要或者规定这些参数

#### 预定义的Values

> 通过Values.yaml文件或者 --set标签定义的Values值，可以在模板中使用 .Values前缀来引用。但是还存在一些可以其他预先定义的数值。

以下的values是预先定义好的，适用于每个template但是不能被覆盖,并且名字大小写敏感：

  * Release.Name: release的名字(不是chart)
  * Release.Time: release发布的最新时间，与release实例的last Realease时间一致
  * Release.Namespace: release的namespace
  * Release.Service: 发布release的服务，通常是tiller
  * Release.IsUpgrade: 当前操作是upgrade或者rollback时，该值是true
  * Release.IsInstall: 当前操作是install时，该值是true
  * Release.Revision:  修订号，从1开始，随着命令helm upgrade递增
  * Chart: Chart.yaml的内容，可以使用Chart.version可作为chart的版本等
  * Files: 包含chart中的所有非特殊文件，是类似map结构的。Files可以 {{index .Files "file.name"}}、{{.Files.Get name}}或者
      {{.Files.GetString name}}函数来获得。文件内容可以使用{{.Files.GetBytes}}函数，得到的是[]byte类型
  * Capabilities: map结构的，包含k8s版本：{{.Capabilities.KubeVersion}}, tiller版本： {{.Capabilities.TillerVersion}}, 支持的k8s api版本： {{.Capabilities.APIVersions.Has "batch/v1"

#### Values

> values是YAML格式的，chart包含一个默认的values.yaml文件,helm install命令可以使用额外的YAML文件，这种方式会将该myvals.yaml和默认的values.yaml合并,并覆盖默认文件中的对应数据。

```
helm install --values=myvals.yaml wordpress
```

#### Scope, Dependencies, Values

> Values即可以为顶层的chart声明values，也可以为charts目录中的charts声明变量。换句话说，values文件可以为charts以及其依赖项提供值。

> 高level的chart有权访问其下的所有变量，但是低level的chart不能访问父层chart的values。

> Values是基于chart的name的，但是name可以被简化，例如子chart是mysql，那么在父层chart中可以使用.Values.mysql.password来访问该values，但在mysql的chart中只需要简化成.Values.password

* Global values

helm2.0.0-alph.2支持global value。这为父层chart传递values给子chart提供了一种方式，在设置像labels属性这种元信息 metadata时比较方便。

当子chart声明了 global values，只影响其子chart，对父层的没有影响。父层chart的global values优先于子chart。

## 使用helm管理charts

helm提供了多种和charts相关的命令

* 创建chart： helm create [chartName]
* 打包chart： helm package [chartName]
* 检查chart:  helm lint [chartName]

## chart repositories

> helm repo命令用于管理chart的仓库，但是helm没有提供上传charts的工具

## Hooks脚手架
以后有时间在看，粗略的看就是可以在k8s服务的生命周期内做一些job，比如数据备份之类的。

## template的函数

比如, 下面的tempalte部分包含一个mytpl的template，然后将结果小写，并且用双引号引起来
```
value: {{ include "mytpl" . | lower | quote }}
```
* quote strings, dont't quote integers

当使用string数据时，用引号引起来总是比光秃秃的更安全：
```
name: {{ .Values.MyName | quote }}
```
但是数据是integers类型时，不要加引号，在k8s解析时会出错。但是当template中字段是env时，所有的values期望的都是string。

* include函数

为了能够包含模板，并且对该模板的输出执行操作，helm使用了include函数:
```
{{- include "toYaml" $value | nindent 2 }}
```
上面包含了一个名为toYaml的模板,将其传递给$value,然后将输出传递给nindent函数。使用{{- ... | nindent _n_ }}模式使得上下文中的include更易读，因为该函数会删掉左边空格包括前一个换行符，然后重新生成一行并且按照指定数量进行缩进。

* required 函数

required函数使得开发者能够根据模板渲染的需要声明values。如果values在values.yml中为空，模板将不会渲染并且返回错误信息：
```
{{ required "A valid foo is required!" .Values.foo }}
```
当 .Values.foo 没有定义时，将返回上述的信息。

* tpl函数

tpl函数允许开发人员将字符串作为模板内的模板值。这有助于将template用作value或者渲染额外的配置文件作用于chart。例如：

```
# values
template: "{{ .Values.name }}"
name: "Tom"

# template
{{ tpl .Values.template . }}

# output
Tom
```
配置文件：
```
# 外部配置文件 conf/app.conf
firstName={{ .Values.firstName }}
lastName={{ .Values.lastName }}

# values
firstName: Peter
lastName: Parker

# template
{{ tpl (.Files.Get "conf/app.conf") . }}

# output
firstName=Peter
lastName=Parker
```

## 其他

* 更新release
  
```
helm upgrade --install <release name> --values <values file> <chart directory> 
```
