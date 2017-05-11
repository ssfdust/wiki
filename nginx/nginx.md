# 负载均衡

## 现实情况
　现在存在多个后端服务器：
   - 运行在80端口的nginx的http服务
   - 运行在443端口的nginx的https服务
   - 运行在8080端口的jira服务
   - 运行在8090端口的confluence服务
   - 运行在8888端口的业务后端服务
   - 运行在10080端口的gitlab服务

## 预计的效果
完成分配任务，服务器正常运行。设计适当的权重分配

## 实现方案
   1. 利用nginx自身的分组功能
   2. 利用iproute2+iptables

## nginx的负载实现
### nginx的upstream的结构以及参数解析
1. 第一种情况对于产品业务运行的后端服务器
    upstream的定义：
    ``` nginx
            upstream tomcat {
                ip_hash;
                server 192.168.1.1:8080 weight=10;
                server 192.168.1.2:8080 weight=5;
                server 192.168.1.3:8080 backup;
                server 192.168.1.4:8080 down;
                server 192.168.1.5:8080 max_fails=3,fail_timeout=8s;
            }
    ```
    * ip_hash表示按照访问的ip地址生成的hash来分配访问，以解决不同用户产生的session问题。
    * weight表示对不同服务器的负载不同 ，weight值越大，负载的权重越大，不设置则为1
    * backup 只有非backup机器都为忙或者down的时候会访问backup机器
    * down 表示这台机器暂时停用
    * max_fails 允许的最大请求失败的次数，默认为1
    * fail_timeout 当请求多次失败后，失败次数达到max_fails，暂停的时间，默认为10s

2. 第二种情况对于缓存服务器来说，将根据每个url的hash将url定向到一个后端服务器。
    ```
            upstream resinserver {
                server 192.168.2.1:8888;
                server 192.168.2.2:8080;
                hash $request_uri;
                hash_method crc32;
            }
    ```
    * hash 以url为hash因子
    * hash_method hash方式

### nginx使用hash_ip的不足与解决方案

#### 无法对同一局域网内的不同ip起到均衡作用
   　　如果一群主机都是在一个局域网内，假设这群主机对应的公网IP都为114.123.233.112。那对于nginx来说这就是一个ip，不能起到负载均衡的作用
   　　换句话来讲，就是nginx不是最前端服务器，在上面这个例子中，还有一台交换机服务器。所以nginx得到的不是真实的ip，无法实现负载均衡。

#### nginx的后端还有其他的分流方式
   　　如果nginx后端还有自己的分流方式，那样也不能保证客户端的session对应到同一台session应用服务器上。那
#### ip_hash部分问题的解决方案
   1. 改写nginx的源码的
   ``` C
        for (;;) {
            for (i = 0;i < 3;i++) {
                hash = (hash * 113 + iphp->addr[i]) % 6271;
   ```
       * 给hash的计算采用新的算法，比如从session中获取新的数值。但是此步骤维护的代价可能很大，nginx需要更新的时候，也有可能涉及到代码的兼容性问题，故不予考虑。
   2. 利用第三方hash模块
    ```
        upstream api {
            server host1:8080;
            server host2:8080;
            hash $cookie_jsessionid;
            hash_method crc32;
        }
    ```
        * 此处利用cookie作为hash因子，来进行分布。
