##  让服务支持https协议访问  
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
###  服务本身还没有使用https协议  
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