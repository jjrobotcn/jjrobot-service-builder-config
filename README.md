### 相关文件描述
- builder.yml： 服务配置版本解析配置
- service_labels.json: 服务信息描述配置
- rams_script.yml: RAMS后台服务资料管理类目中Script项内容
- nav@bks.yml: 导航模块@布科思无线导航版本配置
- nav@jjmgt.yml: 导航模块@磁导航版本配置
- nav@jjbase.yml: 导航模块@简易底盘版本配置
- ${service}@${module}.yml: ...模块@版本.yml

### 解析处理逻辑
用户配置:
```json
{
  "nav": {
    "module": "jjbase",
    "env": {
      "XXX": "YYY"
    }
  }
}
```
解析流程:
1. 同步版本文件
2. 解析builder.yml
3. 获取nav模块中module==jjbase项配置文件描述
4. 读取nav@jjbase.yml
5. ...其它模块
6. 提取各模块配置及用户动态配置(ENV)构建最终dist.yml内容

示例用户配置最终解析nav片断
```yaml
...
  nav:
    image: jjrobot/nav
    container_name: service-nav
    restart: always
    volumes:
      - /etc/localtime:/etc/localtime:ro
    environment:
      module: jjbase
      XXX: YYY
  gserial-chassis:
    image: jjrobot/grpc_serial
...
```

<div id="release"></div>

### 如何更新版本
- 更新模块相关配置内容
- 更新rams_script内容及release项中目标version
- git tag ${version} && push
- RAMS系统中发布


### 如何添加新服务模块
- 编写新服务模块配置文件${service}@${module}.yml
- builder.yml添加新服务模块描述
- service_labels.json添加新服务模块信息
- 参考 [如何更新版本](#release)

### 添加Face服务示例
#### 编写face@v1.yml
```yaml
version: "3"

services:
  face-recognition:
    image: registry.cn-shenzhen.aliyuncs.com/jjrobot2/face-recognition:0.0.1
    container_name: face-recognition
    restart: always
    environment:
      - DB_HOST=db
      - DB_PORT=27017
      - DB_USER=robotjj
      - DB_PASS=ROBOTjjROBOT
      - TOLERANCE=0.5
  face-recognition-gateway:
    image: registry.cn-shenzhen.aliyuncs.com/jjrobot2/face-recognition-gateway:0.0.1
    container_name: face-recognition-gateway
    restart: always
    environment:
      - SERVICE_ENDPOINT=face-recognition:8000
```

#### builder.yml添加服务模块解析
```yaml
...
  face:
    module:
      default: face@v1.yml
      v1: face@v1.yml
```

#### core.yml更新相关核心依赖
```yaml
...
  gateway:
    # 调整网关能力后的新镜像
    #   * 增加了对人脸识别接口的反向代理支持
    #   * 调整开发平台页面反向代理配置
    image: registry.cn-shenzhen.aliyuncs.com/jjrobot2/gateway:2.6.3_dev
    ...
    
  # 独立部署开发管理平台
  management-pages:
    image: registry.cn-shenzhen.aliyuncs.com/jjrobot2/management-pages:2.7.3_dev
    container_name: management-pages
    restart: always
...
```

#### service_labels.json添加服务描述信息
```
...
    "face-recognition": {
      "name": "face-server",
      "label": "人脸识别模块"
    },
    "face-recognition-gateway": {
      "name": "face-gateway",
      "label": "人脸识别模块代理网关"
    },
    "management-pages": {
      "name": "management-pages",
      "label": "开发管理平台页面"
    }
...
```
#### RAMS系统发布新版本
repo.script项填入以下内容的**BASE64**编码内容<br>
```yaml
version: "1"
resolvers:
  - file:
      src: https://s.jj-robot.com/install/jjrobotctl
      dst: /usr/local/bin/jjrobotctl
      x_perm: true
  - file:
      src: https://s.jj-robot.com/install/99-jjrobot-vol-serial.rules
      dst: /etc/udev/rules.d/99-jjrobot-vol-serial.rules
  - image:
      image: registry.cn-shenzhen.aliyuncs.com/jjrobot2/jjrobotd:2.3.0-rc1
      rename: jjrobot/jjrobotd
  - release:
      version: v1.0.0-rc2
```
上述script中描述了4个resolver
- file类型将由src下载内容至dst, 若x_perm==true将更新权限为0744，否则0644
- image类型将更新一个服务镜像并更名为jjrobot/jjrobotd
- release类型将获取[srv-builder](https://srvbuilder.jj-robot.com?sn=test)服务指定version的构建并将内容同步至Daemon本地执行更新

### Q&A
- Q: builder.yml中default项作用<br> 
  A: default项将作为默认加载项，当用户配置中无指定module时将使用default中的配置文件
