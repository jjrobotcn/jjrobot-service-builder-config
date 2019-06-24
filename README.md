### 相关文件描述
- builder.yml： 服务配置版本解析配置
- service_labels.json: 服务信息描述配置
- install_script.yml: 定义安装脚本, Daemon中实现安装方法
- nav@bks.yml: 导航模块@布科思无线导航版本配置
- nav@jjmgt.yml: 导航模块@磁导航版本配置
- nav@jjbase.yml: 导航模块@简易底盘版本配置
- ${service}@${module}.yml: ...模块@版本.yml

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

#### install_script.yml更新安装脚本
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
      image: registry.cn-shenzhen.aliyuncs.com/jjrobot/jjrobotd:2.3.0-rc2
      rename: jjrobot/jjrobotd
  - release:
      version: v1.0.0-rc3
```
上述script中描述了4个resolver
- file类型将由src下载内容至dst, 若x_perm==true将更新权限为0744，否则0644
- image类型将更新一个服务镜像并更名为jjrobot/jjrobotd
- release类型将获取[srv-builder](https://srvbuilder.jj-robot.com?sn=test)服务指定version的构建并将内容同步至Daemon本地执行更新
