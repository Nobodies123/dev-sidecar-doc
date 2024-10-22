# 部署到Cloudflare免费Workers上

## 1、 注册Cloudflare账号

https://www.cloudflare.com/

验证完邮箱

## 2、创建Worker

* 点击左侧Workers菜单
* 点击右边创建服务按钮
* 服务名称随意填写（以下设为`YourWorkerName`）
* 点击右下角的创建服务，创建成功后会自动进入服务配置页面

## 3、部署代理脚本

* 点击快速编辑按钮
* 删除左侧原有的代码
* 将下方代码粘贴进去
* 按照注释修改
* 点击部署按钮并确认
```javascript
export default {
    /**
     * @param {Request} request
     * @param {any} env
     * @param {any} ctx
     */
    async fetch(request, env, ctx) {
        const self_domain_name = env.SELF_DOMAIN_NAME;//你猜这里要干啥
        const self_path = env.SELF_PATH;

        let targetUrl = '';
        // 构造实际的URL
        if (request.url.startsWith('https://')) {
            targetUrl = request.url.slice(`https://${self_domain_name}/${self_path}/`.length);
        }
        else if (request.url.startsWith('http://')) {
            targetUrl = request.url.slice(`http://${self_domain_name}/${self_path}/`.length);
        }

        let host = '';
        // 识别实际域名
        if (targetUrl.match(/^\w*(:\/\/)/)) {
            host = (new URL(targetUrl)).hostname;
        }
        else {
            host = (new URL(`http://${targetUrl}`)).hostname;
        }// hostname仅域名，host包含端口

        // 修补协议头，考虑是否使用HTTPS
        if (!(targetUrl.match(/^\w*(:\/\/)/))) {
            if ((await env.XXX_kv.get(host)) == 'HTTP') {//修改所有的XXX_kv为实际的KV数据库名称
                targetUrl = `http://${targetUrl}`;
            }
            else {
                targetUrl = `https://${targetUrl}`;
            }
        }

        let internalResponse = await fetch(targetUrl, request);

        if (internalResponse.status == 525 && targetUrl.startsWith('https://')) {
            await env.XXX_kv.put(host, 'HTTP', { expirationTtl: 60 });
            internalResponse = await fetch(`http://${targetUrl.slice('https://'.length)}`, request);
        }

        return internalResponse;
    },
};
```

## 4、 配置KV数据库

添加一个KV命名空间，名字随便。再在Worker页面用前面的KV实例名称绑定实例。

## 5、 配置自定义域名

由于Cloudflare的Worker默认域名`*.workers.dev`被屏蔽，请配置一个自定义域名（不要长得太像各类屏蔽网站的域名）。

首先自行寻找域名提供商买一个。可以上[dynv6.com](dynv6.com) 拿一个免费的（问题是不能部署到cf），现在也可以上[nic.us.kg](nic.us.kg)申请一个（能部署到CF，但是实际用的时候好像有点问题）。**极不推荐使用各类所谓的免费“公益”二级域名。**

接下来分两种情况。

#### 5.1 域名可托管到Cloudflare的

托管到Cloudflare，然后直接在Worker页面添加自定义域名即可。

#### 5.2 域名不能托管到Cloudflare的（未测试）

1.   添加一条CNAME记录，指向你的Worker默认域名（据说结尾需要有个点，即`YourWorkerName.workers.dev.`）。
2.   为你的域名申请一个证书并部署到Worker上（此处略）。

## 6、 配置环境变量

添加两个环境变量：`SELF_DOMAIN_NAME`为你的自定义域名，`SELF_PATH`设置为任何一段URL安全的字符串（它起到类似密码的作用，请自行保密）。

## 7、 配置DevSidecar功能增强的代理服务端

域名 = `SELF_DOMAIN_NAME`

路径 = `SELF_PATH`

配置你代码中的域名和路径（端口、密码留空），点击应用即可。

## 8、 测试访问

（悠着点用，低调）

## 9、 FAQs

#### 9.1 设置完成后无法访问外网

##### 9.1.1 任何常见外网都不行

检查以上各步骤是否正确（部分步骤故意写的有的模糊，但是都可以参考Cloudflare的技术文档完成。啥？你嫌弃文档是英文的？这点动手能力都没有就不推荐使用增强模式了）。

##### 9.1.2 部分网站卡在谷歌人机验证（谷歌真是大罪人）

和人机验证中门对狙，或者过一段时间再访问（基本都能成功）。如果确实上不了谷歌搜东西可以用duckduckgo

##### 9.1.3 部分网站卡在Cloudflare人机验证

不要和人机验证中门对狙（因为Cloudflare显然认识自家服务器，从那里代理的请求会被Cloudflare死心塌地地认定你是机器人）。对于可以不走Worker的网站，就不要走Worker。至于其他的网站没有好办法。（也许在Worker后面再加一个反向代理可行？）

更新：喜报！ds1.8.8自2024.10.23起的新远程配置允许直连某蓝色图片网站，不用再用增强模式才能上然后卡住了！该issue终于迎来了几年来最可喜可贺的进展！

##### 9.1.4 Steam相关网页返回403 Forbidden或黑屏

关掉ds的系统代理（不必退出），并另行使用加速器。实测ds开启过增强模式后无法与Watt Toolkit或UU加速器的Steam加速同时使用（即使再次调回默认模式也可能不行），否则Steam相关网页有概率返回403 Forbidden（网页黑屏是因为Steam自身对纯文本的显示问题）。

##### 9.1.5 部分网站显示流量异常，拒绝访问

没有好办法，如果刷新几次之后都不行建议破口大骂或者与网站提供方客服中门对狙。

#### 9.2 npm加速无法开启，报错“Error invoking remote method 'apiInvoke': Error: 'npm' 不是内部或外部命令，也不是可运行的程序 或批处理文件。”

ds的问题，找ds提issue。（说实话我没见到几次）

#### 9.3 浏览器间歇性出现“TOO MANY REDIRECTS”

--刷新一下，或者等几秒再访问，这是Cloudflare的KV数据库写入延迟问题。--

修复了！应该不会有这个毛病了。（代价是访问一些仅http站点可能）

#### 9.4 某些使用增强模式的网站打不开或加载缓慢

手动指明使用http协议，这样代理就不会为了判断能不能用http而纠结半天。（这是个理论上的问题，我其实没碰到过）
