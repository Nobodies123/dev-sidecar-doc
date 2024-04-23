# 部署到Cloudflare免费Workers上

## 1、 注册Cloudflare账号

https://www.cloudflare.com/

验证完邮箱

## 2、创建Worker

* 点击左侧Workers菜单
* 点击右边创建服务按钮
* 服务名称随意填写（`YourWorkerName`）
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
        const self_domain_name = env.SELF_DOMAIN_NAME;
        const self_path = env.SELF_PATH;

        let targetUrl = '';
        if (request.url.startsWith('https://')) {
            targetUrl = request.url.replace(`https://${self_domain_name}/${self_path}/`, "");
        }
        else if (request.url.startsWith('http://')) {
            targetUrl = request.url.replace(`http://${self_domain_name}/${self_path}/`, "");
        }

        let host = '';
        if (targetUrl.match(/^\w*(:\/\/)/)) {
            host = (new URL(targetUrl)).hostname;
        }
        else {
            host = (new URL(`http://${targetUrl}`)).hostname;
        }
        if (!(targetUrl.match(/^\w*(:\/\/)/))) {
            if ((await env.XXXXX.get(host)) == 'HTTPS') {//把此处XXXXX改成随便一个名字，它就是你的KV实例名称，后面要用到
                targetUrl = `https://${targetUrl}`;
            }
            else {
                targetUrl = `http://${targetUrl}`;
            }
        }
        
        let internalResponse = await fetch(new Request(targetUrl, request));
        if (internalResponse.status >= 300 && internalResponse.status < 400) {
            // 获取重定向目标URL
            const redirectUrl = internalResponse.headers.get("Location");
            if (redirectUrl == targetUrl.replace("http://", "https://")) {
                let maxAgeMatchee = internalResponse.headers.get('Strict-Transport-Security');
                if (!maxAgeMatchee) {
                    internalResponse.headers.get('Cache-Control');
                    if (!maxAgeMatchee) {
                        maxAgeMatchee = "max-age=1200";
                    }
                }
                let maxAgeSeconds = parseInt(maxAgeMatchee.match(/max-age=(\d+)/)[1], 10);
                if (maxAgeSeconds < 1200) {
                    maxAgeSeconds = 1200;
                }
                await env.XXXXX.put(host, 'HTTPS', { expirationTtl: maxAgeSeconds });//把此处XXXXX改成随便一个名字，它就是你的KV实例名称，后面要用到
            }
        }
        return internalResponse;
    },
};
```

## 4、 配置KV数据库

添加一个KV命名空间，名字随便。再在Worker页面用前面的KV实例名称绑定实例。

## 5、 配置自定义域名

由于Cloudflare的Worker默认域名`*.workers.dev`被屏蔽，请配置一个自定义域名（不要长得太像各类屏蔽网站的域名）。

首先自行寻找域名提供商买一个。可以上[dynv6.com](dynv6.com) 拿一个免费的。**极不推荐使用各类公益二级域名。**

接下来分两种情况。

#### 5.1 域名可托管到Cloudflare的

自行搜索教程托管到Cloudflare，然后直接在Worker页面添加自定义域名即可。

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

##### 9.1.2 部分网站卡在谷歌人机验证

和人机验证中门对狙，或者过一段时间再访问（基本都能成功）。

##### 9.1.3 部分网站卡在Cloudflare人机验证

不要和人机验证中门对狙（因为Cloudflare显然认识自家服务器，从那里代理的请求会被Cloudflare死心塌地地认定你是机器人）。对于可以不走Worker的网站，就不要走Worker。至于其他的网站没有好办法。（也许在Worker后面再加一个反向代理可行？）

##### 9.1.4 Steam相关网页返回403 Forbidden或黑屏

关掉ds（不必退出），并另行使用加速器。实测ds开启过增强模式后无法与Watt Toolkit或UU加速器的Steam加速同时使用（即使再次调回默认模式也可能不行），否则Steam相关网页有概率返回403 Forbidden（网页黑屏是因为Steam自身问题）。

##### 9.1.5 部分网站显示流量异常，拒绝访问

没有好办法，如果刷新几次之后都不行建议破口大骂或者与网站提供方客服中门对狙。

#### 9.2 npm加速无法开启，报错“Error invoking remote method 'apiInvoke': Error: 'npm' 不是内部或外部命令，也不是可运行的程序 或批处理文件。”

ds的问题，找ds提issue。

#### 9.3 浏览器间歇性出现“TOO MANY REDIRECTS”

刷新一下，或者等几秒再访问，这是Cloudflare的KV数据库写入延迟问题。
