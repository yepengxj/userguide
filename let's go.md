# datafoundry平台快速上手

## hello 2048  
* 　　首先我们要登录datafoundry平台  
  ```  
  $  oc login https://datafoundry-endpoint.xxx.xxx -u username -p password  
  ```  
* 　　然后我们通过一个简单的命令来完成2048应用的镜像构建和发布以及内部服务的生成  

  ```  
  $  oc new-app https://github.com/alexwhen/docker-2048.git
  ```  
其中：  
  * 　　`oc` 是datafoundry平台的命令行控制工具，他提供了对datafoundry平台的所有控制功能  
  * 　　`new-app`是datafoundry平台的操作命令，它可以通过后续指定的若干参数完成一个应用的构建和发布  
  * 　　`https://github.com/alexwhen/docker-2048.git` 是一个git代码仓库，在github上搜索docker-2048即可得到，它是我们要在datafoundry平台发布的第一个应用  
  * 　　输入命令并点击回车后，命令行会等待一段时间，等待时间长短与代码仓库的代码量，以及命令执行位置与github的网络条件有关，这段等待时间中oc会先clone代码仓库到本地，对代码仓库中的dockerfile进行解析，主要是获取基础镜像信息，命令执行成功后会显示如下信息  
    ```  
    --> Found Docker image 9d71014 (8 weeks old) from docker.io for "library/alpine"

        * An image stream will be created as "alpine:latest" that will track the source image
        * A Docker build using source code from https://github.com/alexwhen/docker-2048.git will be created
          * The resulting image will be pushed to image stream "docker-2048:latest"
          * Every time "alpine:latest" changes a new build will be triggered
        * This image will be deployed in deployment config "docker-2048"
        * Port 80 will be load balanced by service "docker-2048"
          * Other containers can access this service through the hostname "docker-2048"
        * WARNING: Image "docker-2048" runs as the 'root' user which may not be permitted by your cluster administrator

    --> Creating resources with label app=docker-2048 ...
        imagestream "alpine" created
        imagestream "docker-2048" created
        buildconfig "docker-2048" created
        deploymentconfig "docker-2048" created
        service "docker-2048" created
    --> Success
        Build scheduled for "docker-2048", use 'oc logs' to track its progress.
        Run 'oc status' to view your app. 
    ```  
*  　　通过输出信息我们可以看到datafoundry平台构建和发布应用的几个基本要素  
  * 　　`buildconfig`，可以简写为bc,用来存储镜像构建所需的配置信息，包括最基本的代码仓库地址，构建分支、tag、commit-id信息，dockerfile位置，镜像构建输出位置及名称，在相对高级的应用场景下还包含ci出发器，github webhock、私有git仓库登录信息等  
  * 　　`deployconfig`，简写为dc，用来存储镜像部署所需的配置信息，  
  * 　　`service`，简写为svc,是平台提供应用高可用和服务发现功能的入口  
  * 　　`imagestream`，简写为is,是平台显示私有仓库镜像信息的入口，通过他也是平台CD功能的触发入口  

*  　　这些基础要素的信息可以通过如下的命令进行查询：  
  ```  
  oc get buildconfig <buildconfig-name>
  oc get deployconfig <deployconfig-name>
  oc get service <service-name>
  oc get imagestream <imagestream-name>
  ```  
*  　　还可以通过如下命令了解基础要素的详情  
  ```  
  oc describe buildconfig <buildconfig-name>  
  oc describe deployconfig <deployconfig-name>  
  oc describe service <service-name>  
  oc describe imagestream <imagestream-name>  
  ```  
*  　　还可以通过如下命令修改基础要素的配置,具体修改的方式和内容讲在后续针对每个基础要素的专题介绍中展开  
  ```  
  oc edit buildconfig <buildconfig-name>  
  oc edit deployconfig <deployconfig-name>  
  oc edit service <service-name>  
  oc edit imagestream <imagestream-name>  
  ```  
　　在简单了解了这些datafoundry平台的基本要素之后，我们来看看命令执行的结果，在这里要特别强调的是datafoundry平台是一个异步设计的命令绝大多数命令的返回结果仅仅代表系统已经接受到了请求，并不表示相关操作已经完成，也就是`oc new-app`执行成功后只是个开始，我们需要逐个确认每个基本要素的执行情况，只有当所有的基本要素都就位之后才表示一个应用真真发布成功。  
  　　不过也有可能出现`oc new-app`命令本身就执行失败或部分失败的问题，大体上有如下几种可能性：  
  * 　　代码仓库不可达，因为网络问题或者仓库地址错误或者私有代码仓库密码错误，导致oc命令不能正确clone代码仓库内容  
  * 　　关键基础要素名称重复，如果平台中已经存在相同名称的基础要素，那么对该基础要的创建工作就会失败，当然基础要素创建的失败并不代表整个应用发布过程的失败，如果旧的基础要素信息满足要求的话是可以继续使用的而不用在意执行`new-app`过程中出现的错误提示，但是如果旧的基础要素信息不符合应用或者服务发布的要求，那么需要删除这些基础要素，基础要素的名称可以通过`new-app`的错误提示确定，也可以通过相关要素的查询命令进行查询，删除这些基础要素的命令是：  

    ```   
  oc delete buildconfig <buildconfig-name>  
  oc delete deployconfig <deployconfig-name>  
  oc delete service <service-name>  
  oc delete imagestream <imagestream-name>  
    ```  
  
## hello wordpress  
　　平台入门的另一个经典是部署一个wordpress应用，但是和以往不一样的是我们使用datafoundry平台提供的MySQL后端服务来保存wordpress的数据
* 　产品datafoundry平台后端服务列表  
  　　我们首先要通过datafoundry平台生成一个MySQL的后端服务之前我们可以先查看一下目前datafoundry平台已经集成的后端服务  
```   
oc get bs -n openshift  
```
注意：   
    * 后端服务是datafoundry特有功能，所以必须要使用datafoundry客户端连接datafoundry服务端后查看
    * 在查看datafoundry平台已集成的后端时要添加后端服务默认的集成命名空间openshift    
   
  　　通过以上命令输出结果为：
 ```   
NAME         LABELS                           BINDABLE   STATUS
Cassandra    asiainfo.io/servicebroker=etcd   true       Active
ETCD         asiainfo.io/servicebroker=etcd   true       Active
Greenplum    asiainfo.io/servicebroker=rdb    true       Active
Kafka        asiainfo.io/servicebroker=etcd   true       Active
MongoDB      asiainfo.io/servicebroker=rdb    true       Active
Mysql        asiainfo.io/servicebroker=rdb    true       Active
PostgreSQL   asiainfo.io/servicebroker=rdb    true       Active
RabbitMQ     asiainfo.io/servicebroker=etcd   true       Active
Redis        asiainfo.io/servicebroker=etcd   true       Active
Spark        asiainfo.io/servicebroker=etcd   true       Active
Storm        asiainfo.io/servicebroker=etcd   true       Active
ZooKeeper    asiainfo.io/servicebroker=etcd   true       Active
```   
 














