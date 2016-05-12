#  服务部署  
##  配置服务  
### 通过环境变量配置服务  

　　在之前的案例中我们用最快的方式创建了docker-2048和wordpress应用，但在实际生产过程中应用服务部署的过程远比这些复杂，为了提高服务的灵活性通常都会使用配置文件或者配置数据库来分离随环境变化而变化的信息，采用这种方式部署的服务通常都需要在服务启动前配置相应的环境变量，但是在将服务打包成容器镜像后就没有让我们手工配置环境变量的机会了，而且由于容器的不可变性也非常不推荐在容器启动后对容器进行修改和配置，因此就需要在服务打包成容器镜像前预留出所有的环境变量，在容器启动时直接将所需的环境变量传递给服务，例如下面是一个mysql服务的启动命令
```    
oc run mysql --image mysql --env MYSQL_ROOT_PASSWORD=my-secret-pw  
``` 
　　上面的例子是通过环境变量直接初始化服务，很多情况下配置是存储在特定配置文件中的，例如spring框架配置的连接池，我们可以通过正则表达式来把环境变量值替换到配置文件中，然后再启动tomcat
  ```    
#!/bin/bash
if [ "$MYSQL_PORT_3306_TCP_ADDR" ]; then
	sed -i 's/^jdbc_url=.*$/jdbc_url=jdbc:mysql:\/\/'$MYSQL_PORT_3306_TCP_ADDR':'$MYSQL_PORT_3306_TCP_PORT'\/'$MYSQL_ENV_MYSQL_DATABASE'\?useUnicode=true\&characterEncoding=UTF-8\&zeroDateTimeBehavior=convertToNull /g' /usr/local/tomcat/webapps/Datahub-1.0-SNAPSHOT/WEB-INF/classes/config.properties
	sed -i  's/^jdbc_username=.*$/jdbc_username='$MYSQL_ENV_MYSQL_USER'/g' /usr/local/tomcat/webapps/Datahub-1.0-SNAPSHOT/WEB-INF/classes/config.properties
	sed -i 's/^jdbc_password=.*$/jdbc_password='$MYSQL_ENV_MYSQL_PASSWORD'/g' /usr/local/tomcat/webapps/Datahub-1.0-SNAPSHOT/WEB-INF/classes/config.properties
fi

catalina.sh run  
``` 
　　有时调试正则表达式是一件非常痛苦的事情，我们可以通过模板生成器来讲环境变量值写入到配置文件中，例如python的envtpl工具，模板工具不但可以减轻调试正则表达式的工作量，可能提供默认值、条件取值等更有价值的功能。
``` 
evntpl config.properties.tpl
``` 
### 通过configmap配置服务
　　上一节，我们介绍了如何通过环境变量来初始化配置文件，这种方法的好处是通用性很好，在任何的容器环境都可以使用，但这种方式有一些缺点，例如要提前准备配置文件模板，增加配置项就需要重新构建镜像，这都带来一定的开发工作量，随着kubernetes本身功能的不断丰富，我们现在有了更多配置服务的方法，下面我们介绍如何通过configmap来传递配置文件。  
　　configmap功能可以讲服务所需的配置文件以卷的形式挂载到容器中，这和在docker中通过`--volume`方式挂在配置文件的方式非常相似，但是在kubernetes configmap功能中配置文件是被保存在ETCD中的，这样就不用担心文件会意外丢失，而且可以自动随着容器在平台中不同节点间的迁移而迁移，下面我们来看看如何创建一个configmap，创建configmap的命令如下：  
``` 
oc create configmap <configmap_name> [options]
``` 
　　我们创建一个redis服务，通过configmap来传递redis启动参数，redis配置文件`redis-config`内容如下：
``` 
maxmemory 2mb
maxmemory-policy allkeys-lru
``` 
　　先创建configmap
``` 
oc create configmap example-redis-config --from-file=redis-config
```  
　　查看创建结果
``` 
oc get configmap example-redis-config -o yaml

apiVersion: v1
data:
  redis-config: |
    maxmemory 2mb
    maxmemory-policy allkeys-lru
kind: ConfigMap
metadata:
  creationTimestamp: 2016-04-06T05:53:07Z
  name: example-redis-config
  namespace: default
  resourceVersion: "2985"
  selfLink: /api/v1/namespaces/default/configmaps/example-redis-config
  uid: d65739c1-fbbb-11e5-8a72-68f728db1985
```  
　　创建一个名为`redis-pod.yaml`的POD定义文件并通过`oc create -f redis-pod.yaml`命令创建这个POD
```
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: kubernetes/redis:v1
    env:
    - name: MASTER
      value: "true"
    ports:
    - containerPort: 6379
    resources:
      limits:
        cpu: "0.1"
    volumeMounts:
    - mountPath: /redis-master-data
      name: data
    - mountPath: /redis-master
      name: config
  volumes:
    - name: data
      emptyDir: {}
    - name: config
      configMap:
        name: example-redis-config
        items:
        - key: redis-config
          path: redis.conf
```  
　　查看POD启动情况
```  
$ oc exec -it redis redis-cli
127.0.0.1:6379> CONFIG GET maxmemory
1) "maxmemory"
2) "2097152"
127.0.0.1:6379> CONFIG GET maxmemory-policy
1) "maxmemory-policy"
2) "allkeys-lru"
```  
　　使用configmap这种方式来传递服务配置的好处是不需要再为传递服务参数而准备容器镜像和服务配置模板，这样我们就可以更方便的使用公共镜像，而不用在公共镜像基础之上创建自定义镜像。
## 通过service完成服务发现  
　　kubernetes的service功能除了可以在平台内部进行负载均衡外，还可以完成在平台内的服务发现。service名称是平台内的DNS短地址，因此可以通过service名称进行同一命名空间内服务间的相互发现和访问，跨命名空间的service访问地址为<servicename>.<namespace>.svc.cluster.local  
  
## 让服务支持https方式访问  
　　https服务现在已经非常普及了，在datafoundry平台上也可以支持，具体使用上可以分两中情况，服务本身已经使用https和服务本身还没有使用https   
###  服务本身已经使用https协议  
　　如果服务本身已经使用https方式部署，让datafoundry平台支持https访问协议就非常简易了，只需通过如下命令创建https协议的route即可  
```
 oc create route passthrough [NAME] \
 --service=SERVICE \
 --hostname=[HOSTNAME]
``` 
其中：  
　　NAME参数是route的名字
　　SERVICE参数是route所对应的service名称，这是为了通过service获取需通过route分发流量的POD IP和端口信息  
　　HOSTNAME参数是route对外提供的域名信息  
*  服务本身还没有使用https协议  
　　这种情况可以需要先为服务申请签名证书，然后可以通过如下命令创建使用https协议的route
```
 oc create route edge [NAME] \
 --service=SERVICE \
 --hostname=[HOSTNAME] \
 --key==example-test.key \
 --cert=example-test.crt
```   
其中：  
　　NAME、SERVICE、HOSTNAME的含义之前已经介绍过  
　　key、cert参数是从签名机构申请回来的证书文件目录    