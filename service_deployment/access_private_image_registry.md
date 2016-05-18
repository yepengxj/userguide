##  访问私有镜像库
　　有时我们需要部署已经上传到私有镜像仓库里的容器镜像，这时也需要给datafoundry平台配置相关secrets来保证datafoundry平台对私有镜像仓库有足够的访问权限
　　具体过程如下
1. 创建访问私有镜像库secrets   
``` 
oc secrets new-dockercfg <dockerpullsecret> \
--docker-server=<registry_server> \
--docker-username=<USERNAME> \ 
--docker-password=<PASSWORD> \
--docker-email=<EMAIL>
``` 
　　其中：
  1.  dockerpullsecret是给secrets起的一个可以识别的名字
  2.  registry_server是需要datafoundry登陆的私有镜像仓库地址，例如registry.dataos.io
  1.  USERNAME是登陆镜像库的用户名
  1.  PASSWORD是登陆镜像库的用户密码
  2.  EMAIL是登陆镜像库的email地址
1.  绑定secrets到平台默认的镜像部署系统账户中
``` 
oc secrets add serviceaccount/<default> secrets/<dockerpullsecret>
```   
　　其中：
  1.  default是datafoundry默认的镜像部署系统账户，用户也可以指定需要的系统账户