#  Postman+Newman+Jenkins给API测试

对API进行自动化测试，选择了Postman和Jenkins对API进行持续集成自动化测试。


## 第一步，新建一个Collection
 collection--也就是将多个接口请求可以放在一起，并管理起来。

## 第二步，在Collections里添加请求
 在设置中添加请求URL、请求方式、请求参数等，并加入测试断言
 可以从右侧中添加，并设置测试数据

```tsx
//test
//状态码
pm.test("Status code is 200", function () {
pm.response.to.have.status(200);
});
//Time
pm.test("Response time is less than 2000ms", function () {
pm.expect(pm.response.responseTime).to.be.below(2000);
});
//response
pm.test("Body is correct", function () {
pm.response.to.have.body("response_body_string");
});

```

 在右侧设置完接口请求信息后点击保存--save

 ![pic1](https://raw.githubusercontent.com/ximeijie/blogs/master/images/2/pic1.png)

## 第三步，在postman中进行测试
 在postman中添加完需要测试的请求后，可以进行合并测试

![pic2](https://raw.githubusercontent.com/ximeijie/blogs/master/images/2/pic2.png)

## 第四步，导出到json文件
 通过export导出为json文件到本地，选择collection v2

![pic3](https://raw.githubusercontent.com/ximeijie/blogs/master/images/2/pic3.png)

## 第五步，搭建Jenkins+newman环境
 1、搭建Jenkins环境
 2、安装newman并安装HTML报告

 cmd输入
```tsx
npm install newman 

```
 安装完成后，输入newman -v 查看版本，检测是否安装成功。 
 3、安装html报告 

```tsx
 npm install -g newman-reporter-html 

```

## 第六步，Jenkins设置
 1、新增自由构建

![image4](https://raw.githubusercontent.com/ximeijie/blogs/master/images/2/pic4.png)

 2、构建触发器，为了定时运行
 H/5  * * * * 

![image5](https://raw.githubusercontent.com/ximeijie/blogs/master/images/2/pic5.png)

 3、添加构建命令，运行相应json文件

```tsx
#!/bin/sh -l
cd /usr/local
newman run octopus_API.postman_collection.json

```
![image6](https://raw.githubusercontent.com/ximeijie/blogs/master/images/2/pic6.png)

 4、设置邮件接收错误构建信息
    构建不成功的以邮箱的形式通知。

![image7](https://raw.githubusercontent.com/ximeijie/blogs/master/images/2/pic7.png)

## 第七步，控制台查看输出文件

![image8](https://raw.githubusercontent.com/ximeijie/blogs/master/images/2/pic8.png)
