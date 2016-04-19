#  术语
* 平台，除非特殊说明在本文中平台即指ldp平台
* build，专指指镜像构建，应用程序构建会用编译  

#  应用在LDP平台发布基本流程  
*  build镜像，应用发布首先要讲应用代码构建成应用镜像，应用镜像构建的过程主要依赖应用代码中包含的Dockerfile来完成，在Dockerfile中明确了应用源代码编译应用程序的过程，以及安装应用程序所依赖的系统功能的过程。build镜像的操作可以在ldp平台中完成，可以在其他平台上完成  
*  deploy镜像，在镜像成功build后，就可以通过相应的镜像仓库将镜像deploy到ldp平台上来  
*  生成servce，面向平台内使用的负载均衡和服务发现机制  
*  生成route，面向平台外使用的负载均衡和服务发现机制    

首先从一个简单直观的例子开始，构建一个2048游戏应用，在这个应用的构建及发布过程中我们将看到如何将镜像发布到LDP平台，并通过LDP平台提供的各项功能来完成应用镜像构建、镜像发布、高可用保证、从平台外访问  2048应用，let's go！  

# hello 2048  
首先我们要登录ldp平台  

```  
$  oc login https://ldp-endpoint.xxx.xxx -u username -p password  
```  
然后我们通过一个简单的命令来完成2048应用的镜像构建和发布以及内部服务的生成  

```  
$  oc new-app https://github.com/alexwhen/docker-2048.git
```  
其中：  
* `oc` 是ldp平台的命令行控制工具，他提供了对ldp平台的所有控制功能  
* `new-app`是ldp平台的操作命令，它可以通过后续指定的若干参数完成一个应用的构建和发布  
* `https://github.com/alexwhen/docker-2048.git` 是一个git代码仓库，在github上搜索docker-2048即可得到，它是我们要在ldp平台发布的第一个应用  

输入命令并点击回车后，命令行会等待一段时间，等待时间长短与代码仓库的代码量，以及命令执行位置与github的网络条件有关，这段等待时间中oc会先clone代码仓库到本地，对代码仓库中的dockerfile进行解析，主要是获取基础镜像信息，命令执行成功后会显示如下信息  

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
通过输出信息我们可以看到ldp平台构建和发布应用的几个基本要素  
* `buildconfig`，可以简写为bc,用来存储镜像构建所需的配置信息，包括最基本的代码仓库地址，构建分支、tag、commit-id信息，dockerfile位置，镜像构建输出位置及名称，在相对高级的应用场景下还包含ci出发器，github webhock、私有git仓库登录信息等  
* `deployconfig`，简写为dc，用来存储镜像部署所需的配置信息，  
* `service`，简写为svc,是平台提供应用高可用和服务发现功能的入口  
* `imagestream`，简写为is,是平台显示私有仓库镜像信息的入口，通过他也是平台CD功能的触发入口  
这些基础要素的信息可以通过如下的命令进行查询：  

```  
oc get buildconfig  
oc get deployconfig  
oc get service  
oc get imagestream  
```  
还可以通过如下命令了解基础要素的详情  

```  
oc describe buildconfig  ****  
oc describe deployconfig  ****  
oc describe service  ****  
oc describe imagestream  ****  
```  
还可以通过如下命令修改基础要素的配置,具体修改的方式和内容讲在后续针对每个基础要素的专题介绍中展开  

```  
oc edit buildconfig ****  
oc edit deployconfig ****  
oc edit service ****  
oc edit imagestream ****  
```  

在简单了解了这些ldp平台的基本要素之后，我们来看看命令执行的结果，在这里要特别强调的是ldp平台是一个异步设计的命令绝大多数命令的返回结果仅仅代表系统已经接受到了请求，并不表示相关操作已经完成，也就是`oc new-app`执行成功后只是个开始，我们需要逐个确认每个基本要素的执行情况，只有当所有的基本要素都就位之后才表示一个应用真真发布成功。  
不过也有可能出现`oc new-app`命令本身就执行失败或部分失败的问题，大体上有如下几种可能性：  
* 代码仓库不可达，因为网络问题或者仓库地址错误或者私有代码仓库密码错误，导致oc命令不能正确clone代码仓库内容  
* 关键基础要素名称重复，如果平台中已经存在相同名称的基础要素，那么对该基础要的创建工作就会失败，当然基础要素创建的失败并不代表整个应用发布过程的失败，如果旧的基础要素信息满足要求的话是可以继续使用的而不用在意执行`new-app`过程中出现的错误提示，但是如果旧的基础要素信息不符合应用或者服务发布的要求，那么需要删除这些基础要素，基础要素的名称可以通过`new-app`的错误提示确定，也可以通过相关要素的查询命令进行查询，删除这些基础要素的命令是：  
    
  ```   
oc delete buildconfig ****   
oc delete deployconfig ****   
oc delete service ****   
oc delete imagestream ****  
  ```  
  

###  镜像构建入门   
在确认`oc new-app`执行符合预期后为了确认应用在ldp平台上发布的情况，首先执行在ldp平台执行频率最高的命令   

```  
$  oc get pods   
```   
这里出现了一个ldp平台中最最重要的一个基本要素POD，它是平台调度的最小单元，它是由一个或多个容器组成的集合。………………   
通过执行上述命令，我们可以看到类似如下信息 
  
```  
NAME                  READY     STATUS      RESTARTS   AGE
docker-2048-1-build   0/1       Running     0          1m   
```      

可以看到已经有一个pod在运行，我们可以通过如下命令查看pod的执行情况   

```   
$  oc describe pods docker-2048-1-build 
```  

通过上述命令可以看到pods的基本信息，包括pods执行的节点，pods初始化过程中的信息，进一步我们可以使用如下命令  

```  
$  oc logs docker-2048-1-build  
```  
上述命令讲显示pod中应用在执行过程中输出到标准输出和标准错误中的信息，当输出信息显示为如下内容时说明镜像构建已经完成  

```  
push sussessed  
```  
我们再通过`oc get pods` 命令来查看必要的信息，输入内容如下:  

```  
docker-2048-1-build   POD     complete  
docker-2048-1-deploy  POD     running  
docker-2048-1-abcxyz  POD     containercreate  
```  
或者也有可能为  

```  
docker-2048-1-build   POD     complete  
docker-2048-1-deploy  POD     containercreate  
```  
以上描述的都是正常构建镜像情况的结果，假如在镜像构建过程中输出的信息为异常终端，或者pod的状态不为complete，需要通过`oc logs`和`oc describe`命令从如下几个方面进行排查: 
 
*  构建任务启动失败  
  *  POD资源配额已用完，build任务也是通过启动相应POD来完成，如果POD的配置已经达到上线将无法启动构建任务  
  *  调度构建任务POD失败，关于POD调度失败的问题详见相关专题  
  *  拉取构建任务镜像失败，在LDP平台中镜像构建任务是由一类系统基础镜像启动成POD后完成，如果构建基础镜像拉取失败，将无法启动构建任务。  
* 镜像构建失败  
  * 容器构建Dockerfile不正确，如果Dockerfile不正确容器将不能构建成功，因此需要保证Dockerfile经过严格测试，dockerfile中的关键信息发生变化是要及时更新，例如tomcat在Apache网站上的下载地址  
  * 构建过程中无法访问远程资源，例如要现在的软件包，要更新的系统功能，这多是由于镜像中系统更新或者软件下载的源默认为国外地址，在国内访问时存在较多限制导致，当然也可能由于系统的相关网络功能异常导致，随着系统的不断稳定这类问题出现的机率已相对较小，因此当镜像构建过程中出现网络方面问题时可以先排除是否为国内无法访问国外网络地址，如果是在访问国外网络地址不受限制或国内网络地址出现异常时先排除对端服务是否正常，如对端服务正常才考虑平台网络问题，系统管理员处理此类问题的方法和流程详见网络问题处理专题介绍  
  * 私有git仓库的登录的鉴权信息配置错误，可参考buildconfig专题介绍中私有git仓库登录鉴权信息配置过程进行检查  
  * 镜像构建所需计算、内存资源超出平台所分配资源，这多是针对镜像构建中执行的从源代码build执行程序的过程，表示构建任务相对复杂或者构建任务配置不正确导致在构建过程中消耗了过多的系统资源。  
* 镜像上传失败  
  * 镜像构建正常但镜像上传失败，这类问题多是由平台内置镜像仓库异常导致，可尝试通过`oc start-build buildconfigname`来重新构建，以及删除bc、is后通过`oc new-build   https://github.com/***`来重新启动创建及启动构建任务，如持续失败或异常可与系统管理员联系。系统管理员处理此类问题的方法和流程详见镜像仓库问题处理专题介绍  
  * 还有一种相少见的情况为节点到内部镜像仓库网络异常导致，这是构建任务还在建立与平台内容镜像仓库的网络连接，如果此时构建任务失败多提示“no route to host”或者“50X”网关错误，开发者遇到此类问题可以在尝试按照上述镜像上传失败案例中提供的`oc start-build`以及`oc new-build`命令尝试重新构建，如不能解决可与系统管理源联系，系统管理员可参考平台网络相关检查和处理流程进行处理，详见具体专题。  

### 镜像部署入门  
上面一节我们讨论镜像构建和经常会在镜像构建过程中遇到的问题和处理方式，本节我们来看一下2048游戏的部署过程。依然是从最长用的`oc get pods`命令，开始通过执行该命令我们可以看到在目前系统已经完成的build任务和正在执行的deploy任务  

```    
docker-2048-1-build   POD     complete  
docker-2048-1-deploy  POD     running  
docker-2048-1-abcxyz  POD     containercreate  
``` 
从命令显示结果可以看出平台是通过一个部署任务POD来启动应用容器，一个POD的启动过程会经过调度，创建、启动、运行几个阶段。通常在平台内部构建的应用部署会很快。但是也会遇到一些问题，有些是平台安全策略导致应用不能获取所需的高级别系统权限，例如fork进程、root执行、使用主机端口等，有些是由于平台信息发生变化而部署任务未能及时更新或者填写错误导致，我们可以通过`oc get pods`来获得应用启动的状态，通常会遇到如下的异常信息

*  ImgPullBackOff  
   如果应用容器启动状态为`ImgPullBackoff`则说明平台无法从镜像仓库中拉取应用镜像，如果是从平台内置镜像仓库拉取应用镜像可以通过`oc get is`命令查看当前应用镜像的仓库地址信息，如果是从平台外镜像仓库中拉取镜像也需要再次确认镜像仓库地址是否正确。
*  CrashLoopBackOff
如果应用容器启动状态为`CrashLoopBackOff`则说明容器中得应用没有正常启动，导致这种情况的原因主要包括如下几个方面：  
	*  容器中应用启动方式不对，由于在容器中除非特殊需要通常是没有进程管理功能的，这也就是说容器中的应用不能以后台进程方式运行，子进程终止后容器也会终止，这些在应用适配容器以及Dockerfile编写时要格外注意，同时这些错误可以在容器构建完成后在Docker运行一下就可以很快发现  
	*  应用依赖的其他服务或者组件没有启动或者无法连接，可以通过`oc logs `命令检查应用日志
	*  容器中应用的启动用户不满足平台安全要求，为了安全考虑通常平台是不会允许以容器中的root用户来启动容器中的应用，而是会限制用户id在100以上的用户来启动应用，而这需要通过在Dockerfile中用user关键词来显示声明，例如
	
	  ```  
	  FROM baseimage
	  user root
	  RUN update_some_system_package
	  
	  user 200
	  RUN build_some_application
	  
	  CMD start_application.sh
	  ```  
	* 平台在安全上的考虑除了容器中应用的启动用户设置外，还会对容器中应用启动后的一些系统级的操作进行限制，例如fork、kill等。   	 

### 内部服务入门  
在完成了镜像构建和部署之后，我们就可以通过内部服务将已经部署的服务开放给平台中的其他应用来使用了。内部服务在平台中叫`service`，它是平台中的重要基础要素，可以通过`oc get service`命令经行查看。  

### 外部服务入门  
在应用开放给平台内部使用的同时，应用还可以开放给平台外部使用，将能力对外开放的功能叫`route`,它也是平台的基础要素之一，可以通过`oc expose svc`命令来实现。例如，将2048应用开放对外使用的命令为：  

```  
oc expose svc docker-2048  
```   

通过`oc get route`命令可以查看route的记录，有了route就可以在浏览器中使用2048游戏了。 

### 引入其他小伙伴   
在工作过程中我们常常和其他同事一起配合完成某些任务，这样就涉及到namespace概念，平台会为每个用户默认创建一个namespace，平台通过namespace来实现不同用户的应用、服务的隔离，通过namespace来控制每个用户的资源配额，例如cpu、内存的可用量，可以部署的应用量，可以创建的内、外部服务量等。平台中的用户可以将自己的namespace开放给别的用户使用或查询信息，例如  

```   
oc policy add-role-to user admin username  
```  
这可以将别的用户添加为个人namespace的管理员用户  
  
```  
oc policy add-role-to user view username  
```  
这可以给别的用户赋予个人namespace的浏览权限  
可以看出平台是通过角色来控制不同人对某一namespace所拥有的权限，默认情况下除了admin、view外，还有一个edit角色可以作用在  namespace上  

# 镜像构建进阶  
之前在部署2048应用时使用`oc new-app`命令同时创建了buildconfig、deployconfig和service，有时会需要单独创建buildconfig时，可以通过`oc new-build`命令完成，例如构建docker-2048就可以通过下面的命令完成  

```  
oc new-build https://github.com/alexwhen/docker-2048.git  
```   
在修改完成程序需要重新构建应用镜像时是可以通过 `oc start-build buildconfigname` 来再次启动构建任务可以节省不少时间。  
如果构建代码是放在私有仓库中的话，可以把私有git仓库的登陆信息配置在buildconfig中来保证平台对私有git仓库的访问权限，具体过程如下：

*  创建一个登陆凭据   
 
```   
oc secrets new-basicauth secretname --username=USERNAME --password=PASSWORD  
```   
*  将凭据绑定到平台的默认构建任务执行用户账户下

```
oc secrets add serviceaccount/builder secrets/secretname
```
*  将凭据绑定到需要使用凭据登陆的buildconfig下,这需要通过`oc edit bc buildconfigname`来修改buildconfig的配置信息，例如

```
apiVersion: "v1"
kind: "BuildConfig"
metadata:
  name: "sample-build"
spec:
  output:
    to:
      kind: "ImageStreamTag"
      name: "sample-image:latest"
  source:
    git:
      uri: "https://github.com/user/app.git" 
    sourceSecret:
      name: "secretname"
    type: "Git"
  strategy:
    sourceStrategy:
      from:
        kind: "ImageStreamTag"
        name: "python-33-centos7:latest"
    type: "Source"
```
其中添加的内容为

```
sourceSecret:
   name: "secretname"
```

# 镜像部署进阶  

一 概观
   
   平台中的部署（deployment，简称dc）可以看做是pod副本的控制器，用户可以自定义dc的配置文件来创建dc,也可以通过触发事件来创建dc,例如通过oc run命令来触发dc的创建。
   
二 创建dc
   
   dc包括以下几个关键部分：

1. 一个pod副本控制模板，这个模板描述了如何部署应用。
2. dc默认的副本数量。
3. 部署策略（部署策略包括Rolling Strategy、Recreate Strategy和Custom Strategy）
4. 自动触发dc进行重新部署的触发条件（Configuration Change Trigger 和 Image Change Trigger）
    
dc可以通过oc deploy dc命令来运行,以下是一个dc模板的示例：
   
```
	{
	  "kind": "DeploymentConfig",
	  "apiVersion": "v1",
	  "metadata": {
	    "name": "frontend"
	  },
	  "spec": {
	    "template": { **1**
	      "metadata": {
	        "labels": {
	          "name": "frontend"
	        }
	      },
	      "spec": {
	        "containers": [
	          {
	            "name": "helloworld",
	            "image": "openshift/origin-ruby-sample",
	            "ports": [
	              {
	                "containerPort": 8080,
	                "protocol": "TCP"
	              }
	            ]
	          }
	        ]
	      }
	    },
	    "replicas": 5, **2**
	    "selector": {
	      "name": "frontend"
	    },
	    "triggers": [
	      {
	        "type": "ConfigChange" **3**
	      },
	      {
	        "type": "ImageChange", **4**
	        "imageChangeParams": {
	          "automatic": true,
	          "containerNames": [
	            "helloworld"
	          ],
	          "from": {
	            "kind": "ImageStreamTag",
	            "name": "origin-ruby-sample:latest"
	          }
	        }
	      }
	    ],
	    "strategy": { **5**
	      "type": "Rolling"
	    }
	  }
	} 
	
```

1. 名称为frontend的副本控制器，描述了一个名称为Ruby的应用如何进行部署
2. 默认有5个名称为frontend的副本
3. 当dc配置文件发生任何改变都将触发新的部署
4. 当origin-ruby-sample:latest镜像发生任何改变都将触发新的部署（项目中请确保origin-ruby-sample:latest的可用性，具体可使用os get is查看镜像是否可用）
5. 默认部署策略为Rolling（滚动更新，即不影响原有pod的情况下，逐步替换原有pod）

三  开始部署
   
   可以使用平台界面或者以下命令开始部署：
   
```
   	$ oc deploy <deployment_config> --latest
```

如果已经有一个部署进程在进行，会给出部署正在进行的提示并且不能产生新的部署

   使用以下命令获取部署的信息：
   
```
$ oc deploy <deployment_config>
```

   或者使用以下信息获取更为详细的dc信息：
   
```
$ oc describe dc <deployment_config>
```

   使用以下命令取消和终止部署：
   
```
$ oc deploy <deployment_config> --cancel
```

使用以下命令重试最近一次失败的部署：

```
$ oc deploy <deployment_config> --retry
``` 

   如果最近一次部署不是失败的状态，此命令会给出提示并且不会创建一次重试的部署。这里要注意使用--retry参数将会继续使用失败部署所使用的dc配置，并不会加载新的配置，如果要加载最新配置，则要使用--latest参数

四 回滚部署

   回滚操作将会使部署回滚到前一次的部署

   使用以下命令进行回滚到最近一次成功的部署：
   
``` 
$ oc rollback <deployment_config>
``` 

   回滚过程中，dc配置文件将会被自动更新到回滚的dc配置，然后开始新的部署。当回滚结束后，为了防止再次触发部署，dc中image change trigger将会被屏蔽。可以使用以下命令开启image change trigger：

``` 
$ oc deploy <deployment_config> --enable-triggers
``` 

   使用以下命令回滚到具体的版本：
   
``` 
$ oc rollback <deployment_config> --to-version=1
``` 

   不进行回滚操作的情况下查看回滚的信息可以使用以下命令：
   
``` 
$ oc rollback <deployment_config> --dry-run
``` 

五 查看部署日志
   
   使用以下命令查看部署日志：
   
``` 
$ oc logs dc <deployment_config> [--follow]
``` 

   部署失败和正在进行的时候都可以查看日志的，但是当部署成功后，部署的pod会被自动删除，所以也就不能再查看部署日志。
   
   在部署pod还存在的情况下，可以使用以下命令查看历史部署信息：
   
```   
$ oc logs --version=1 dc <deployment_config>
```   

   这个命令会返回第一次部署的信息

六 自动触发  
自动触发定义了新的部署开始的条件。  
1. Configuration Change Trigger  
	如果dc配置文件中定义了Configuration Change Trigger，一旦dc被创建就会触发一次部署
	以下是Configuration Change Trigger的示例：  

	``` 
	"triggers": [
	  {
	    "type": "ConfigChange"
	  }
	]
		
	``` 
    
2. Image Change Trigger  
	
2. Image Change Trigger  
   
   Image Change Trigger触发方式为镜像发生改变，以下是Image Change Trigger的示例：
   
``` 
	   "triggers": [
	  {
	    "type": "ImageChange",
	    "imageChangeParams": {
	      "automatic": true, 
	      "from": {
	        "kind": "ImageStreamTag",
	        "name": "origin-ruby-sample:latest"
	      },
	      "containerNames": [
	        "helloworld"
	      ]
	    }
	  }
	]
``` 

   如果"automatic": true选项配置为false，则自动触发失效

七 部署策略
   
   部署策略决定了部署过程，我们提供多种部署策略来满足对可用性有不同要求的场景。
   readiness checks用来侦探部署pod是否准备好，如果准备好，则为部署策略返回部署pod可用，如果侦探到部署pod没有准备好，则部署停止。

   默认策略为Rolling strategy

   1. Rolling strategy
   
      Rolling strategy进行滚动更新，并且支持lifecycle hooks（lifecycle hooks用来在部署过程中插入新的过程）,以下是Rolling strategy的示例：
      
'''
		   "strategy": {
		  "type": "Rolling",
		  "rollingParams": {
		    "timeoutSeconds": 120, 
		    "maxSurge": "20%", 
		    "maxUnavailable": "10%" 
		    "pre": {}, 
		    "post": {}
		  }
		}
'''

  2. Recreate Strategy
  
     Recreate Strategy也支持lifecycle hooks，并且有一个基本的退出操作，以下是Recreate Strategy示例：

		  "strategy": {
		  "type": "Recreate",
		  "recreateParams": { 
		    "pre": {}, 
		    "post": {}
		  }
		}
   3. Custom Strategy
      Custom Strategy允许用户自定义部署过程

      以下是Custom Strategy的示例：

		  "strategy": {    
		  "type": "Custom",    
		  "customParams": {    
		    "image": "organization/strategy",   
		    "command": ["command", "arg1"],
		    "environment": [
		      {
		        "name": "ENV_1",
		        "value": "VALUE_1"
		      }
		    ]
		  }
		}

   
    以上示例中的image：organization/strategy定义了部署过程，command部分会覆盖镜像中dockerfile中所写的cmd,环境变量所定义的变量将会传入到部署过程中。

八 Lifecycle Hooks
   
   Lifecycle Hooks允许在部署过程中插入过程，以下是Lifecycle Hooks的示例：

	   "pre": {
	  "failurePolicy": "Abort",
	  "execNewPod": {} 
	}

   以上的execNewPod是基于pod的Lifecycle Hooks，目前部署过程只可以插入基于pod的Lifecycle Hooks

   每一个Lifecycle Hooks都有failurePolicy，failurePolicy定义了当hook失败后部署策略应该采取什么行动。failurePolicy包括三种行动：Abort、Retry和Ignore。
   
   以下是基于pod的Lifecycle Hooks的示例：

	   {
	  "kind": "DeploymentConfig",
	  "apiVersion": "v1",
	  "metadata": {
	    "name": "frontend"
	  },
	  "spec": {
	    "template": {
	      "metadata": {
	        "labels": {
	          "name": "frontend"
	        }
	      },
	      "spec": {
	        "containers": [
	          {
	            "name": "helloworld",
	            "image": "openshift/origin-ruby-sample"
	          }
	        ]
	      }
	    }
	    "replicas": 5,
	    "selector": {
	      "name": "frontend"
	    },
	    "strategy": {
	      "type": "Rolling",
	      "rollingParams": {
	        "pre": {
	          "failurePolicy": "Abort",
	          "execNewPod": {
	            "containerName": "helloworld", 
	            "command": [ 
	              "/usr/bin/command", "arg1", "arg2"
	            ],
	            "env": [ 
	              {
	                "name": "CUSTOM_VAR1",
	                "value": "custom_value1"
	              }
	            ],
	            "volumes": ["data"] 
	          }
	        }
	      }
	    }
	  }
	}

九 部署资源消耗

   如果部署所在的项目定义了资源限制，那么所有的pod都将受限于资源限制。在部署策略中也可以定义资源限制，以下是资源限制示例：

	   type: "Recreate"
	resources:
	  limits:
	    cpu: "100m" 
	    memory: "256Mi" 

十 手工扩展部署

   使用以下命令进行手工扩展部署实例：
   
   	$ oc scale dc frontend --replicas=3

十一 分配pod到指定的节点上
    
   可以使用node selectors为pod指定节点，增加node selector前需要先为node打标签，以下是node selector的示例：

	    apiVersion: v1
	    kind: Pod
	    spec:
	      nodeSelector:
	        disktype: ssd
   
   
   
# 外部服务进阶

一 概观

   平台上的应用经常会需要连接平台外部的服务，例如外部的数据库，或者外部SaaS endpoint。这些外部资源可以按照模型被制作成为平台内的服务，这样平台上的应用就可以像与平台内服务交互一样与外部服务进行交互。

二 外部mysql数据库库

   最常见的外部服务就是mysql数据库了。平台上的应用连接外部数据库需要满足以下几点：

   1. 可供交互的端点
   2. 一套包括用户名、密码和数据库的认证
   
   连接外部数据库的解决方案包括：

   1. 一个openshift的service对象
   2. 这个服务有至少一个可交互的端点
   3. 包括以上所说认证信息的环境变量

   以下步骤描述了	如何连接外部mysql数据库：
  
   1. 为外部服务创建一个service，方法与创建平台内的service服务一样，除了service配置文件中的selector区域
      service中的selector与pod配置文件中的label一致时，就将pod与service对应。外部服务的service并不需要与任何pod连接，我们只需要让selector区域留空，这就代表了该service对应的是外部service。这样，EndpointsController就暂时不会去匹配service的端点与pod，我们就可以手动去配置该service的端点，以下是手动配置service端点的示例：

		      kind: "Service"
		      apiVersion: "v1"
		      metadata:
		        name: "external-mysql-service"
		      spec:
		        ports:
		          -
		            name: "mysql"
		            protocol: "TCP"
		            port: 3306
		            targetPort: 3306
		            nodePort: 0
		      selector: {} 

  2. 为service创建endpoint，endpoint告诉service的proxy和router指向该service的流量往哪里转发。以下是创建endpoint的示例：
  
			  kind: "Endpoints"
			  apiVersion: "v1"
			  metadata:
			    name: "external-mysql-service" **1**
			  subsets: **2**
			    -
			      addresses:
			        -
			          IP: "10.0.0.0" **3**
			      ports:
			        -
			          port: 3306 **4**
			          name: "mysql"
   
              1. service的名字
              2. 如果提供多个endpoints,指向该service的流向将会被负载均衡到这些不同的endpoints上
              3. endpoints的IP，不可以是loopback (127.0.0.0/8)、link-local (169.254.0.0/16)和link-local multicast (224.0.0.0/24)
              4. 端口和名字必须和以上所提到的认证信息中的端口及名字一致
  3. service以及service的endpoint创建好后，在pod中使用环境变量传递访问以上创建的service的信息，以下是创建pod的示例：
  
		   kind: "DeploymentConfig"
		   apiVersion: "v1"
		   metadata:
		     name: "my-app-deployment"
		   spec: **1**
		    strategy:
		      type: "Rolling"
		      rollingParams:
		        updatePeriodSeconds: 1
		        intervalSeconds: 1
		        timeoutSeconds: 120
		    replicas: 2
		    selector:
		      name: "frontend"
		    template:
		      metadata:
		        labels:
		          name: "frontend"
		      spec:
		        containers:
		          -
		            name: "helloworld"
		            image: "origin-ruby-sample"
		            ports:
		              -
		                containerPort: 3306
		                protocol: "TCP"
		            env:
		              -
		                name: "MYSQL_USER"
		                value: "${MYSQL_USER}" **2**
		              -
		                name: "MYSQL_PASSWORD"
		                value: "${MYSQL_PASSWORD}" **3**
		              -
		                name: "MYSQL_DATABASE"
		                value: "${MYSQL_DATABASE}" **4**
		    1. 这里省略了dc配置文件中的其他内容
		    2. 用户名
		    3. 密码
		    4. 数据库
		    
三 外部SaaS Provider

   常见的外部服务是外部SaaS endpoint，为了支持外部SaaS endpoint，应用需要满足以下几点：

   1. 一个可供交互的端点（endpoint）
   2. 一套包括API key、用户名和密码的认证信息
   
   以下步骤描述了如何与外部SaaS Provider连接：

   1. 为外部服务创建一个service，方法与创建平台内的service服务一样，除了service配置文件中的selector区域
   2. 为service创建endpoint，endpoint告诉service的proxy和router指向该service的流量往哪里转发，以下是endpoint示例：
   
		   kind: "Endpoints"
		   apiVersion: "v1"
		   metadata:
		     name: "example-external-service" 
		   subsets: 
		   - addresses:
		     - ip: "10.10.1.1"
		     ports:
		     - name: "mysql"
		       port: 3306

 以上示例的pod中，所接受到的环境变量如下所示：

 EXAMPLE_EXTERNAL_SERVICE_SERVICE_HOST=<ip_address>

 EXAMPLE_EXTERNAL_SERVICE_SERVICE_PORT=<port_number>

 SAAS_API_KEY=<saas_api_key>

 SAAS_USERNAME=<saas_username>

 SAAS_PASSPHRASE=<saas_passphrase>


