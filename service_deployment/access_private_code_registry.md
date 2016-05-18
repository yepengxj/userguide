##  访问私有代码仓库
　　为了让datafoundry平台在构建镜像时访问私有代码，我们需要在datafoundry平台中创建secrets资源，并把secrets绑定到平台中。  
　　具体过程如下：  
1.  创建访问私有代码库secrets
``` 
oc secrets new-basicauth <basicsecret> \
--username=<USERNAME> \
--password=<PASSWORD>
``` 
　　其中：
  1.  basicsecret是给secrets起的一个可以识别的名字
  1.  USERNAME是登陆代码库的用户名
  1.  PASSWORD是登陆代码库的用户密码
1.  绑定secrets到平台默认的镜像构建账户中
``` 
 oc secrets add serviceaccount/builder secrets/<basicsecret>
``` 
1.  在buildconfig中指定secrets,
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
      name: "<basicsecret>"
    type: "Git"
  strategy:
    sourceStrategy:
      from:
        kind: "ImageStreamTag"
        name: "python-33-centos7:latest"
    type: "Source"
```   