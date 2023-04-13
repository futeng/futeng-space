# clashX

本文介绍如何使用 Clash 配合浏览器插件 SwitchyOmega 来配置科学上网。

## macOS 

步骤：

1. 下载最新版本 [ClashX.dmg](https://github.com/yichengchen/clashX/releases) ；

2. 安装并新增订阅（订阅地址即为付费购买的机场提供的密钥 URL），安装时需要给予一定权限；

   ![image.png](http://cdn.wejoin365.com/img/2023/ob/20230413145112.png)
   ![image.png](http://cdn.wejoin365.com/img/2023/ob/20230413145307.png)

3. 可以测速并设置为系统代理，此时访问 https://www.google.com.hk/ ，应该已经可以访问外网。注意此时是全局代理，后期配置好浏览器的 SwitchyOmega 插件后可以取消掉。

![image.png](http://cdn.wejoin365.com/img/2023/ob/20230413145520.png)

4. 安装浏览器插件  [SwitchyOmega](chrome-extension://padekgcemlokbadohgkifijomclgjgif/options.html#!/about)，支持 Chrome、Safari等。

5. 在插件的导入/导出界面，选择在线恢复，填入以下地址（国内常见站点的白名单），配置自动代理：

   ```shell
   https://raw.githubusercontent.com/futeng/futeng-space/main/conf/OmegaOptions.bak
   ```

   ![image.png](http://cdn.wejoin365.com/img/2023/ob/20230413150209.png)

6. 将SwitchyOmega 配置为 auto switch。此时可取消 Clash 的全局代理。浏览器应可正常访问外网。

   ![image.png](http://cdn.wejoin365.com/img/2023/ob/20230413150331.png)

7. 对于 GPT、new bing 等使用，切记固定一个美区节点，不要轻易变动 IP。

![image.png](http://cdn.wejoin365.com/img/2023/ob/20230413150428.png)