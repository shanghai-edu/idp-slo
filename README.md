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

### 下载文件

安装git服务

```bash
yum install git
```

 拉取项目

```bash
cd /opt
git clone https://github.com/shanghai-edu/idp-slo.git
```

### 修改 server.xml

修改tomcat/conf/server.xml

在文件末尾</Host>的上方添加如下代码

```xml
<Context path="logout" docBase="/opt/idp-slo/src/" debug="0" reloadable="true" crossContext="true"/>
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
<Context path="logout" docBase="/opt/idp-slo/src/" debug="0" reloadable="true" crossContext="true"/>

</Host>
    </Engine>
  </Service>
</Server>
```

修改完server.xml后重启tomcat服务即可完成部署，同时别忘了防火墙或者反代将路径暴露出去

## 使用方法
### 单点注销
由于取消了 SAML 的单点注销配置，现在 SP 在注销时，需要传入额外的参数来协助完成单点注销。

SP 在取消掉 SAML 的单调注销后，`return` 参数将会生效，可以使用该参数将注销后的 SP 重定向到一个新的地址，比如将他重定向到 IdP 自身的注销地址

https://sp.xxxxxx.cn/Shibboleth.sso/Logout?return=https://idp.xxx.edu.cn/idp/profile/Logout

这样就在不开启 SAML 注销的情况下实现了单点注销，当然还是没法办实现单点注销后的回调。

这对于任何的 IdP 都是通用的，不需要他们支持带回调参数。

### 带回调参数的单点注销

如果 IdP 已经支持了带回调参数的单点注销，那么此时可以将 `return` 的地址替换为 `https://idp.xxx.edu.cn/logout/logout.html`，例如:

https://sp.xxxxxx.cn/Shibboleth.sso/Logout?return=https://idp.xxx.edu.cn/logout/logout.html

此时 `logout.html` 在没有参数时，也会自动跳转至 `https://idp.xxx.edu.cn/idp/profile/Logout`，效果和上述的单点注销是一致的。

而当我们给他带上 `redirect_url` 参数以后，例如:

https://sp.xxxxxx.cn/Shibboleth.sso/Logout?return=https://idp.xxx.edu.cn/logout/logout.html?redirect_url=https://www.baidu.com

此时 `logout.html` 会通过 `iframe` 的方式完成 IdP 侧的注销，由于这里是同域的，因此不存在跨域的问题。

在完成注销以后，`logout.html` 就会将请求重定向到 `redirect_url` 地址，即实现了带回调参数的单点注销。

