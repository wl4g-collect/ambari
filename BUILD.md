# ambar 编译构建

- 构建下包太慢
    - 建议买海外 US 的机器：[https://ecs-buy.aliyun.com/ecs?spm=5176.ecscore_server.0.0.23544df5ZQ5GEX#/custom/prepay/us-west-1?orderSource=buyWizard-console-list](https://ecs-buy.aliyun.com/ecs?spm=5176.ecscore_server.0.0.23544df5ZQ5GEX#/custom/prepay/us-west-1?orderSource=buyWizard-console-list)
    - 推荐选：
        - 抢占实例 - 美国 - 西部/硅谷等 `8C 8G 40GiB 100Mbps Ubuntu 20.04`
        - `ping downloads.apache.org`   能接近 100ms
        - `ambari` 下载`hadoop/hbase`相关包的地址源码如：https://github.com/wl4g-collect/ambari/blob/release-2.7.8/ambari-metrics/pom.xml#L43

- 安装 git、unzip、lrzsz、jdk

```bash
apt update
apt install git unzip lrzsz openjdk-8-jdk-headless -y
```

- 安装 python

```bash
apt install python2 -y # ambari-2.7.8 只能用 >= python2.6
ln -snf /usr/bin/python2 /usr/bin/python
```

- 安装 maven

```bash
curl -OL 'https://dlcdn.apache.org/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz'
tar -xf apache-maven-3.9.6-bin.tar.gz
echo 'PATH=$PATH:/root/apache-maven-3.9.6/bin' >> .bashrc
source .bashrc
```

- 安装 node

```bash
curl -OL 'https://nodejs.org/dist/v4.5.0/node-v4.5.0-linux-x64.tar.gz'
tar -xf node-v4.5.0-linux-x64.tar.gz
echo 'PATH=$PATH:/root/node-v4.5.0-linux-x64/bin' >> .bashrc
source .bashrc
```

- 安装 node yarn
    - 由于是老版本 node-4.5.0 安装对应 yarn-0.23.0 可能在线包已不存在，应该手动安装

```bash
curl -OL 'https://github.com/yarnpkg/yarn/releases/download/v0.24.6/yarn-v0.24.6.tar.gz'
tar -xf yarn-v0.24.6.tar.gz
echo 'PATH=$PATH:/root/dist/bin' >> .bashrc
source .bashrc
```

- 安装 python2 依赖库
    - 解决报错1：`[ERROR] around Ant part ...... @ 4:241 in /root/ambari/ambari-metrics/ambari-metrics-host-monitoring/target/antrun/build-psutils-compile.xml` 即：`No module named setuptools (将python命令拉出来测试就能看到)`
    
    ```bash
    sudo apt install python-setuptools -y # python3-setuptools
    ```
    
    - 解决报错2：`mvn install-f ambari-metrics/ambari-metrics-host-monitoring/` 指定模块构建就能看到如下错误堆栈：`[exec] building '_psutil_linux' extension  #include  [exec] |  ^~~~iasing -Wdate-time -D_FORTIFY_SOURCE=2 -g -fdebug-prefix-map=/build/python2.7-1x6jhf/python2.7-2.7.18~rc1=. -fstack-protector-st~~~~~~  error: command 'x86_64-linux-gnu-gcc' failed with exit status 1`
    
    ```bash
    sudo apt install python2-dev python-is-python2 -y
    ```
    

- 构建 ambari
    - 参考（官方未提供二进制包?）：[https://cwiki.apache.org/confluence/display/AMBARI/Installation+Guide+for+Ambari+2.7.8](https://cwiki.apache.org/confluence/display/AMBARI/Installation+Guide+for+Ambari+2.7.8)

```bash
#git clone git@github.com:wl4g-collect/ambari.git
git clone https://github.com/wl4g-collect/ambari
cd ambari/
git checkout ref-release-2.7.8 # 构建修复版

# 方案1: 设置忽略证书校验无效?
# bugs: https://issues.apache.org/jira/browse/WAGON-452 or https://github.com/apache/maven-wagon/pull/36
#export MAVEN_OPTS="-Dmaven.wagon.http.ssl.insecure=true \
#-Dmaven.wagon.http.ssl.allowall=true \
#-Dmaven.wagon.http.ssl.ignore.validity.dates=true"

# 方案2: 使用 javaagent 动态替换 sslCerts 校验代码
export MAVEN_OPTS="-noverify -javaagent:/root/playground-agent-modifier-all.jar"
export AGENT_DEBUG=true
export AGENT_CONFIG_PATH="/root/agent-modifier.yaml"

cat <<EOF >/root/agent-modifier.yaml
# jdk1.8 replace to: sun.security.x509.X509CertImpl#checkValidity
modifiers:
  transformers:
    - name: sun.security.x509.X509CertImpl
      classes: []
      constructors: []
      methods:
        - !INSERT_AT_METHOD
          name: checkValidity
          paramTypes: [ 'java.util.Date' ]
          line: 339
          src: |
            System.out.println("Welcome to Cracked X509Cert Trusted !");
            return;
EOF

mvn \
-B install jdeb:jdeb \
-DnewVersion=2.7.8.0.0 \
-DbuildNumber=da8f1b9b5a799bfa8e2d8aa9ab31d6d5a1cc31a0 \
-Dpython.ver="python >= 2.6" \
-Drat.skip=true \
-DskipTests \
-DskipFindbugs \
-rf :ambari-infra-manager-it \
-T 4C
```

## FAQ

- 构建报错1：`/root/ambari/ambari-logsearch/ambari-logsearch-appender/target/package does not exist`

```bash
sed -i -p 's#<delete dir="target/package" />#<!--<delete dir="target/package" />-->#g' \
/root/ambari/ambari-logsearch/ambari-logsearch-appender/build.xml
```

- 构建时可能报错的模块，可暂注释

```bash
# 可暂时注释 web 模块 (MacOS下必须要加 -p)
#sed -ip 's#<module>ambari-web</module>#<!--<module>ambari-web</module>-->#g' pom.xml
#sed -ip 's#<module>ambari-views</module>#<!--<module>ambari-views</module>-->#g' pom.xml
#sed -ip 's#<module>ambari-admin</module>#<!--<module>ambari-admin</module>-->#g' pom.xml

#sed -ip 's#<module>ambari-metrics-assembly</module>#<!--<module>ambari-metrics-assembly</module>-->#g' ambari-metrics/pom.xml
#sed -ip 's#<module>ambari-metrics-host-monitoring</module>#<!--<module>ambari-metrics-host-monitoring</module>-->#g' ambari-metrics/pom.xml

#sed -ip 's#<module>ambari-logsearch-it</module>#<!--<module>ambari-logsearch-it</module>-->#g' ambari-logsearch/pom.xml
#sed -ip 's#<module>ambari-logsearch-assembly</module>#<!--<module>ambari-logsearch-assembly</module>-->#g' ambari-logsearch/pom.xml

#sed -ip 's#<module>ambari-infra-assembly</module>#<!--<module>ambari-infra-assembly</module>-->#g' ambari-infra/pom.xml
#sed -ip 's#<module>ambari-infra-solr-plugin</module>#<!--<module>ambari-infra-solr-plugin</module>-->#g' ambari-infra/pom.xml
```
