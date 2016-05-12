#  服务部署  
##  配置服务  
### 通过环境变量配置服务  

　　在之前的案例中我们用最快的方式创建了docker-2048和wordpress应用，但在实际生产过程中应用服务部署的过程远比这些复杂，我们的服务为了提高应用的灵活性通常都会使用配置文件或者配置数据库来分离随环境变化而变化的信息，采用这种方式部署的服务通常都需要在服务启动前配置相应的环境变量，但是在将服务打包成容器镜像后就没有让我们手工配置环境变量的机会了，而且由于容器的不可变性也非常不推荐在容器启动后对容器进行修改和配置，因此就需要在服务打包成容器镜像前预留出所有的环境变量，在容器启动时直接将所需的环境变量传递给服务，例如下面是一个mysql服务的启动命令
  ```    
oc run mysql --image mysql --env MYSQL_ROOT_PASSWORD=my-secret-pw  
``` 
　　上面的例子是通过环境变量直接初始化服务，很多情况下配置是存储在特定配置文件中的，例如spring框架配置的连接池，我们可以通过正则表达式来把环境变量值替换到配置文件中，然后再启动tomcat
  ```    
#!/bin/bash
if [ "$MYSQL_PORT_3306_TCP_ADDR" ]; then
	sed  -i 's/^jdbc_url=.*$/jdbc_url=jdbc:mysql:\/\/'$MYSQL_PORT_3306_TCP_ADDR':'$MYSQL_PORT_3306_TCP_PORT'\/'$MYSQL_ENV_MYSQL_DATABASE'\?useUnicode=true\&characterEncoding=UTF-8\&zeroDateTimeBehavior=convertToNull /g' /usr/local/tomcat/webapps/Datahub-1.0-SNAPSHOT/WEB-INF/classes/config.properties


	sed -i  's/^jdbc_username=.*$/jdbc_username='$MYSQL_ENV_MYSQL_USER'/g' /usr/local/tomcat/webapps/Datahub-1.0-SNAPSHOT/WEB-INF/classes/config.properties

	sed  -i 's/^jdbc_password=.*$/jdbc_password='$MYSQL_ENV_MYSQL_PASSWORD'/g' /usr/local/tomcat/webapps/Datahub-1.0-SNAPSHOT/WEB-INF/classes/config.properties
fi

catalina.sh run  
```   
### 通过secrets或configmap配置服务
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

### 内部服务  
在完成了镜像构建和部署之后，我们就可以通过内部服务将已经部署的服务开放给平台中的其他应用来使用了。内部服务在平台中叫`service`，它是平台中的重要基础要素，可以通过`oc get service`命令经行查看。  

### 上传配置文件

### 设置资源需求


### route  
在应用开放给平台内部使用的同时，应用还可以开放给平台外部使用，将能力对外开放的功能叫`route`,它也是平台的基础要素之一，可以通过`oc expose svc`命令来实现。例如，将2048应用开放对外使用的命令为：  

```  
oc expose svc docker-2048  
```   

通过`oc get route`命令可以查看route的记录，有了route就可以在浏览器中使用2048游戏了。 