# 0x01 问题
最近同事反馈一个问题，说POST请求时出现404 Bad Request，在Linux下必现，但是使用POSTMAN模拟请求时没有问题。他跟我说了问题的缘由：本来该接口是GET请求的，也不需要什么参数，但是现在使用的是POST的时候就出现错误了。其代码如下：
```php
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, 'http://api.xxx.com/all-tags');
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
curl_setopt($ch, CURLOPT_POST, 1);
curl_setopt($ch, CURLOPT_HTTPHEADER, array(
    'Content-Type:application/json; charset=UTF-8', 
    'Authorization:Bearer h-VEqtiblL693QZFQoTHT-SUooMF1OTLFtNbExGs'
));
$result = curl_exec($ch);
print_r($result);
```

执行结果：

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/1/20170706145243.png)

我之前也遇到过这个问题，就是本来用GET请求的用POST方式去请求，但是那时候忙，没有空去找原因。今天不是那么忙，找找这个原因。

# 0x02 分析过程
#### 1.会不会环境问题，试试Windows下
Windows环境：5.6.30-nts-Win32，curl:7.51.0。如下图：

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/1/20170706150315.png)

执行结果：

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/1/20170706150618.png)

也没有出现像上面的400错误结果，说明Windows没有问题，也不是代码差异问题。

#### 2.这段错误怎么来的？
我惯用的手段就是抓包了，抓包一看一切就很清晰了，不过后面还有其他方法，先看看抓包分析。

先看看Linux环境：7.0.16-NTS，curl:7.29.0，如下图：

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/1/20170706151422.png)

在Linux和Windows都抓包了，发现其响应都不相同，Windows的是200 OK，Linux的是400 Bad Request。如下图：

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/1/20170706152007.png)

错误是服务器返回的，那问题来了？代码是一样的，为什么会有不同的结果？难道curl的问题，于是把POST请求报文拿出来，对比了一下：

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/1/20170706152510.png)

结果真的跟我预想的一样，请求头信息果然不一样，Linux下多了一行：Content-Length: -1，所以服务器就报了400 Bad Request错误，可以参见[rfc2616的Content-Length](https://tools.ietf.org/html/rfc2616#page-119)和[4.4 Message Length](https://tools.ietf.org/html/rfc2616#section-4.4)。多加的一行头应该跟PHP的curl扩展有关系，这也可能是curl的一个bug，其原因不深究，感兴趣的可以去研究一下。

#### 3.怎么修改呢？
1.本来就是GET请求的，非要用POST请求方式，所以直接用GET请求即可。如下：
```php
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, 'http://api.xxx.com/all-tags');
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
curl_setopt($ch, CURLOPT_HTTPHEADER, array(
    'Content-Type:application/json; charset=UTF-8', 
    'Authorization:Bearer h-VEqtiblL693QZFQoTHT-SUooMF1OTLFtNbExGs'
));
$result = curl_exec($ch);
print_r($result);

```
2.非要用POST也可以，POST访问方式问题是由于Content-Length为-1时出现400错误的，所以要指定Content-Length，如下：
```php
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, 'http://api.xxx.com/all-tags');
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
curl_setopt($ch, CURLOPT_POST, 1);
curl_setopt($ch, CURLOPT_HTTPHEADER, array(
    'Content-Type:application/json; charset=UTF-8', 
    'Authorization:Bearer h-VEqtiblL693QZFQoTHT-SUooMF1OTLFtNbExGs',
    'Content-Length:0' // 设置了Content-Length为0
));
$result = curl_exec($ch);
print_r($result);
```
3.POST访问方式页可以设置CURLOPT_POSTFIELDS让curl自己计算长度，如下：
```php
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, 'http://api.xxx.com/all-tags');
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
curl_setopt($ch, CURLOPT_POST, 1);
curl_setopt($ch, CURLOPT_POSTFIELDS, ""); // 设置了CURLOPT_POSTFIELDS
curl_setopt($ch, CURLOPT_HTTPHEADER, array(
    'Content-Type:application/json; charset=UTF-8', 
    'Authorization:Bearer h-VEqtiblL693QZFQoTHT-SUooMF1OTLFtNbExGs'
));
$result = curl_exec($ch);
print_r($result);
```

#### 还有什么方法找出错误呢？

上面已经说过了，不用抓包的形式，还有其他方法找出错误，这就是curl自带的verbose。例如上面的错误，其实可以这样子找差异：
```php
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, 'http://api.xxx.com/all-tags');
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
curl_setopt($ch, CURLOPT_POST, 1);
curl_setopt($ch, CURLOPT_VERBOSE, true); // 这里加了verbose
curl_setopt($ch, CURLOPT_HTTPHEADER, array(
    'Content-Type:application/json; charset=UTF-8', 
    'Authorization:Bearer h-VEqtiblL693QZFQoTHT-SUooMF1OTLFtNbExGs'
));
$result = curl_exec($ch);
print_r($result);
```

执行结果：

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/1/20170707104458.png)

由此看出，可以用设置curl选项CURLOPT_VERBOSE来看出请求的整个过程，可以不用抓包形式获取报文。

# 0x03 结论

1. 正确的使用GET和POST等请求方式，避免不必要的麻烦。

2. Content-Length是个好东西，但是要慎用，可以参见上面所说的[rfc2616的Content-Length](https://tools.ietf.org/html/rfc2616#page-119)和[4.4 Message Length](https://tools.ietf.org/html/rfc2616#section-4.4)。
3. 抓包分析还是不错的，底层嘛，拿到了数据啥都可以分析出来。