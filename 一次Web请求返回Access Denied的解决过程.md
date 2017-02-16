# 0x01 事情缘由
最近部门来了两个PHP开发的小伙伴儿，那得有人去带他们啊，装环境什么的。A同事把之前的备份的虚拟机文件都拷过去，用虚拟机安装的CentOS。网络配好了，开发环境（Nginx+PHP-FPM）已装好，也配好vhost，改一下路径，貌似一切没有问题了。宿主机外配好host打开Chrome，输入test.php文件的地址，结果出现Access Denied!。于是鼓捣了下，还是不行，于是叫我帮看看。

# 0x02 排查过程
##### 1. 查看虚拟域名是否访问正确
ping一下域名，结果发现是正确，能ping通（当然服务器没有禁ping）。ps查看nginx加载的配置文件，cat查看vhost是否配置好，发现没有问题，那么久查看访问日志，发现日志没开。开了access log，另开窗口tailf access.log查看，刷新浏览器，access log有访问记录，状态码依然403。

##### 2. 文件是否有权限
ps aux|grep php查看到php-fpm的worker是www用户，ps aux|grep nginx查看到nginx的worker也是www用户，ls -l查看vhost的root文件夹/htdocs，发现也是www用户的，这说明权限应该是没有的。创建一个index.html，权限跟test.php一样的，访问之可以正常访问，这样说明权限没有问题。于是我怀疑他们的配置文件有问题，应该是php-fpm出问题了，但是用户权限都是www，这就矛盾了，查看nginx的error log，居然没有错误信息（这...让我怀疑人生了，那时候真的没有看到有错误日志输出）。

##### 3. 看来要放大招
上面的方法不行，看来要放大招了，tcpdump抓包，tcpdump -ieth0 tcp。访问test.php，抓到了，Windows用wireshark分享，查看nginx 80端口到php-fpm 9000的有报文，分析之，查看nginx的参数什么的也没什么明显错误。查看反应报文发现php-fpm返回给nginx的是Access to the script '/htdocs/test.php' has been denied (see security.limit_extensions) ，于是查看php-fpm.conf，发现没有配置security.limit_extensions，应该不是这个问题。但是为什么会提示这个错误呢？而且为什么nginx错误日志居然没有，至今百思不得其解，明天要去看看。

##### 4. 控制变量法，访问我的php-fpm
我们是用桥接方式的（IP多，人少，不怕），都在一个局域网下。于是把我的php-fpm（首先我的是正常的），修改为外网能访问，listen = 127.0.0.1:9000改成listen = 9000。访问之，返回404 Not Found，说明请求正常（因为我的不存在/htdocs/test.php），他的nginx也没有问题。那就奇怪了，把他的php-fpm.conf备份，拷我的php-fpm.conf过去，结果还是报403的错。这就里，那只剩php.ini了。

##### 5. php.ini的错
首先查看跟cgi相关的，和我的一对比，发现他的cgi.fix_pathinfo为0，我的为注释掉的，注释之，保存php.ini。重启php-fpm，访问成功！

# 0x03 查看手册
查看php帮助手册，有如下：

cgi.fix_pathinfo boolean 
Provides real PATH_INFO/ PATH_TRANSLATED support for CGI. PHP's previous behaviour was to set PATH_TRANSLATED to SCRIPT_FILENAME, and to not grok what  PATH_INFO is. For more information on PATH_INFO, see the CGI specs. Setting this to 1 will cause PHP CGI to fix its paths to conform to the spec. A setting of zero causes PHP to behave as before. It is turned on by default. You should fix your scripts to use SCRIPT_FILENAME rather than PATH_TRANSLATED. 

下面是中文：

cgi.fix_pathinfo boolean

 对 CGI 提供了真正的 PATH_INFO/PATH_TRANSLATED 支持。以前 PHP 的行为是将 PATH_TRANSLATED 设为 SCRIPT_FILENAME，而不管 PATH_INFO 是什么。有关 PATH_INFO 的更多信息见 cgi 规格。将此值设为 1 将使 PHP CGI 修正其路径以遵守规格。设为 0 将使 PHP 的行为和从前一样。默认为1。用户应该修正其脚本使用 SCRIPT_FILENAME 而不是 PATH_TRANSLATED。
 
什么是PATH_TRANSLATED，也有下面一段：
 
'PATH_TRANSLATED'当前脚本所在文件系统（非文档根目录）的基本路径。这是在服务器进行虚拟到真实路径的映像后的结果。  

Note: 自 PHP 4.3.2 起，PATH_TRANSLATED 在 Apache 2 SAPI 模式下不再和 Apache 1 一样隐含赋值，而是若 Apache 不生成此值，PHP 便自己生成并将其值放入 SCRIPT_FILENAME 服务器常量中。这个修改遵守了 CGI 规范，PATH_TRANSLATED 仅在 PATH_INFO 被定义的条件下才存在。  Apache 2 用户可以在 httpd.conf 中设置 AcceptPathInfo 

其中，PATH_TRANSLATED 仅在 PATH_INFO。

# 0x04 总结
没事不要乱改东西，保持默认就好。除非你很有把握。（先占个位置，以后再写）
