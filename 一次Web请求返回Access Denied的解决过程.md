# 0x01 事情缘由
最近部门来了两个PHP开发的小伙伴儿，那得有人去带他们啊，装环境什么的。A同事把之前的备份的虚拟机文件都拷过去，用虚拟机安装的CentOS。网络配好了，开发环境（Nginx+PHP-FPM）已装好，也配好vhost，改一下路径，貌似一切没有问题了。宿主机外配好host打开Chrome，输入test.php文件的地址，结果出现Access Denied!。于是鼓捣了下，还是不行，于是叫我帮看看。

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/20170217223448.png)

# 0x02 排查过程
##### 1. 查看虚拟域名是否访问正确
ping一下域名，结果发现是正确，能ping通（当然服务器没有禁ping）。ps查看nginx加载的配置文件，cat查看vhost是否配置好，发现没有问题，那么久查看访问日志，发现日志没开。开了access log，另开窗口tailf access.log查看，刷新浏览器，access log有访问记录，状态码依然403。

##### 2. 文件是否有权限
`ps aux|grep -E php\|nginx|grep -v grep`查看到php-fpmh和nginx的worker进程都是www用户。

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/20170217224845.png)

`ll /htdocs/` 查看vhost的root文件夹/htdocs，发现也是www用户的，这说明权限应该是没有问题的。

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/20170217224326.png)

创建一个index.html，权限跟test.php一样的，访问之可以正常访问，这样说明权限没有问题。于是我怀疑他们的配置文件有问题，应该是php-fpm出问题了，但是用户权限都是www，这就矛盾了，查看nginx的error log，居然没有错误信息（这...让我怀疑人生了，那时候真的没有看到有错误日志输出，里面只有nginx的一些初始化信息，以为没有错误）。

**下面是知道有错误日志后补充的内容，你可以先调到第3点再回来读：**

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/20170217230026.png)

从上面就可以看出：php-fpm返回了错误信息给nginx，错误信息显示，执行的脚本名为/htdocs，而不是/htdocs/test.php。所以提示see security.limit_extensions，因为默认的只能执行.php后缀的。

##### 3. tcpdump分析是否有通信
上面的方法不行，也没有日志输出，真是百思不得其解。从应用层分析不出，那就从底层来分析。要放大招了，tcpdump抓包，`tcpdump -ienp0s10f0 -w /htdocs/test.pcap`。访问test.php，抓到了。

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/5C3DD9FC-2C05-4F80-BBA4-4E2A932BD641.jpeg)

在Windows用wireshark分析，查看nginx 80端口到php-fpm 9000的有报文，分析之，查看nginx的参数什么的也没什么明显错误。查看反应报文发现php-fpm返回给nginx的是Access to the script '/htdocs' has been denied (see security.limit_extensions)，这说明执行的脚本名为/htdocs，而不是/htdocs/test.php，这说明是php-fpm解析错的文件了。查看FCGI报文，看到传的参数如下：

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/D4BC8E16-2A1B-4C34-9A93-CD944CD2A6FC.jpeg)

SCRIPT_FILENAME参数没错，但是传了PATH_INFO和PATH_TRANSLATED。

查看nginx.conf，有如下：

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/20170217232431.png)

##### 4. php.ini配置错了
尽然传了PATH_INFO和PATH_TRANSLATED以及SCRIPT_FILENAME。首先查看php.ini跟cgi相关的配置，发现他的cgi.fix_pathinfo为0，原来是这里被改了，默认为注释的，就算没有注释掉，默认也为1。**重启php-fpm，访问成功！**



那为什么会是这样的呢？

查看php帮助手册，有如下：

> cgi.fix_pathinfo boolean 

> Provides real PATH_INFO/ PATH_TRANSLATED support for CGI. PHP's previous behaviour was to set PATH_TRANSLATED to SCRIPT_FILENAME, and to not grok what  PATH_INFO is. For more information on PATH_INFO, see the CGI specs. Setting this to 1 will cause PHP CGI to fix its paths to conform to the spec. A setting of zero causes PHP to behave as before. It is turned on by default. You should fix your scripts to use SCRIPT_FILENAME rather than PATH_TRANSLATED. 

下面是中文：

> cgi.fix_pathinfo boolean

> 对 CGI 提供了真正的 PATH_INFO/PATH_TRANSLATED 支持。以前 PHP 的行为是将 PATH_TRANSLATED 设为 SCRIPT_FILENAME，而不管 PATH_INFO 是什么。有关 PATH_INFO 的更多信息见 cgi 规格。将此值设为 1 将使 PHP CGI 修正其路径以遵守规格。设为 0 将使 PHP 的行为和从前一样。默认为1。用户应该修正其脚本使用 SCRIPT_FILENAME 而不是 PATH_TRANSLATED。
 
什么是PATH_TRANSLATED，也有下面一段：
 
> 'PATH_TRANSLATED'当前脚本所在文件系统（非文档根目录）的基本路径。这是在服务器进行虚拟到真实路径的映像后的结果。  

> Note: 自 PHP 4.3.2 起，PATH_TRANSLATED 在 Apache 2 SAPI 模式下不再和 Apache 1 一样隐含赋值，而是若 Apache 不生成此值，PHP 便自己生成并将其值放入 SCRIPT_FILENAME 服务器常量中。这个修改遵守了 CGI 规范，PATH_TRANSLATED 仅在 PATH_INFO 被定义的条件下才存在。  Apache 2 用户可以在 httpd.conf 中设置 AcceptPathInfo 

其中，PATH_TRANSLATED 仅在 PATH_INFO。

# 0x04 总结
> 没事不要乱改东西，保持默认就好。除非你很有把握。

> 别人的“成功”未必都能复制的，但是失败肯定是有原因的。