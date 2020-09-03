# idp-slo

由于浏览器安全性的提升，在iframe中嵌套sp的slo页面会因为跨域问题导致无法只能注销掉sp侧无法注销idp侧登录状态，所以采用同域名下部署一个页面用来控制登出并且额外提供了跳转回调地址的功能

## shibboleth-sp 设置

如需使用此方法，需要设置sp为仅注销本地，不通过saml2注销。

修改/etc/shibboleth/shibboleth2.xml

```xml
<!-- SAML and local-only logout. -->
<Logout>SAML2 Local</Logout>
```

删除如上位置的saml2，最终解决应该如下

```xml
<!-- SAML and local-only logout. -->
<Logout>Local</Logout>
```

## Idp 设置

仅需部署一个网页即可，可以使用apache，nginx等反代服务

我们这里仅提供tomcat的部署方法，无需安装任何东西。

### 复制文件

在tomcat/webapp 下新建文件夹logout

将网页拷贝至tomcat/webapp/logout下

### 修改 server.xml

修改tomcat/conf/server.xml

在文件末尾</Host>的上方添加如下代码

```xml
<Context path="logout" docBase="logout" debug="0" reloadable="true" crossContext="true"/>
```

其中docBase为webapp目录下对应的文件夹，path为访问的路径

如果您的server.xml没做过太大的改动，那么最后他会看起来像这样

```xml
<Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">

<!-- SingleSignOn valve, share authentication between web applications
             Documentation at: /docs/config/valve.html -->
<!--
        <Valve className="org.apache.catalina.authenticator.SingleSignOn" />
        -->

<!-- Access log processes all example.
             Documentation at: /docs/config/valve.html
             Note: The pattern used is equivalent to using pattern="common" -->
<Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
  <!-- slo网页部署 -->
<Context path="logout" docBase="logout" debug="0" reloadable="true" crossContext="true"/>

</Host>
    </Engine>
  </Service>
</Server>
```

修改完server.xml后重启tomcat服务即可完成部署，同时别忘了防火墙或者反代将路径暴露出去

## 使用方法

在slo登出的时候，原本是访问sp侧路径 https://sp.xxxxxx.cn/Shibboleth.sso/Logout 完成slo，此方法内置了一个return参数协助跳转，我们仅仅需要在访问时加上一点参数，例如

https://sp.xxxxxx.cn/Shibboleth.sso/Logout?return=https://idp.xxxx.com/logout/logout.html?redirect_url=https://www.baidu.com

return后面的一整个url是sp侧完成注销之后进行的跳转，这时候直接去访问了对应的idp域名下我们刚刚部署的网页（示例中是https://idp.xxxx.com/logout/logout.html?redirect_url=https://www.baidu.com）

网页会通过iframe同域名完成idp侧的注销，成功后跳转至redirect_url中的网址（示例中是https://www.baidu.com）

