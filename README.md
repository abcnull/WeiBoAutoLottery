@[toc]

# 大致流程与思路

我目前想做到 jmeter 接口访问微博进行自动抽奖，我需要大致以下流程

- 访问关注人的微博接口

  每天运行 jmeter 脚本，访问搜索关注的人的微博的接口，注意筛选日期当天，并且返回每页微博数量最大

- 匹配具体信息拿到 mid

  我们通过某种匹配方式获取前面接口响应的微博信息为抽奖或者其他之类的，然后再找到对应的 mid 号码

- 然后我们循环访问发送

稍微详细的思路如下：

- 通过接口搜索不是转发的，关键字是@微博抽奖平台的并且日期是指定日期的微博数据
- 通过搜索接口的返回数据正则提取所有 mid 微博号及其具体信息，并且获取第一个 mid 和最后一个 mid 号（这个接口最多返回 15 个 mid！），并且提取一个总共搜索的微博条数计算后面要刷新几次（正常没刷新一次即执行某个接口，然后多返回 15 条 mid）
- 通过循环控制器，前面已经拿到循环几次了，然后控制器中循环刷新即访问某个接口，然后接口内拿到 mid，不断累加在最前面的搜索接口的 mid 中
- 循环最后结束，有一个 beanshell 将所有 mid 分为两大类，一类需要评论，一类不需要，并且在其中做基本的筛查工作，得到需要评论的 mid 保存为数组变量，还得到不需要评论的 mid 保存为数组变量
- 有两个循环控制器，一个为需要评论的 mid，其中做循环点赞和转发，另一个为不需要评论的 mid，其中做循环点赞和转发，而且其中都可以写上请求前 beanshell，用来做更为精准的筛查工作，之所以将评论和不评论的微博区分，正式因为需要评论的微博更需要智能识别要评论什么，这一部分可以在前置 beanshell 中写，这部分 beanshell 还未做，以后可以补充

前置条件：

- 你需要关注很多人，最好都是大 V
- 你需要至少三位互关好友

# sina 登陆接口介绍

经过我自己的观察尝试，发现登陆是没法做的，因为即使 UI 正常登陆之后也是要通过手机接收验证码的，如果非要用手机的验证码我想到用以下几种方式：

- 手机接到验证码自动传到自己的邮箱，然后 jmeter 从邮箱爬取验证码
- 找到 sina 遗留后者隐藏的获取验证码的接口

这几种方法，第一种我嫌麻烦，第二种我没找到，与其非要登录之后在操作，那我们为什么不直接使用 token 或者 cookies 呢？

下面我会介绍下微博的 cookies，当然我这里还是要放上微博登录接口

**发送验证码 POST**
```http
https

passport.weibo.com

/protection/mobile/sendcode?token=2YzFfPTUsAFCaPhBHrAEVEZj0oddUxPXICnByb3RlY3Rpb24.

encrypt_mobile:\u624b\u673a\u53f7\u683c\u5f0f\u9519\u8bef
```

**验证验证码 POST**

```http
https

passport.weibo.com

/protection/mobile/confirm?token=2YzFfPTUsAFCaPhBHrAEVEZj0oddUxPXICnByb3RlY3Rpb24.

encrypt_mobile: 9f6354c5a451
code: 771723
```

**登录 GET**

```http
https

weibo.com

/u/5240852795/home?wvr=5
```

# sina 转发接口介绍（可以连带评论）

后来我发现 sina 没有 token 只有 cookies，只要将 cookies 附在请求头中即可，我通过请求几个 XHR 接口，抓取 cookies，然后在在线的字符串比较网站比较了一下，删去不同的字符串即可

## 转发接口

```
https

POST

weibo.com

/aj/v6/mblog/forward?ajwvr=6&domain=100505&__rnd=1597901313249
```

rnd 未知

## 必填的请求头

转发微博的时候一共有 5 个必填的信息头

**本人 cookie**

不太清楚之后会不会变，但是我第二天再次使用是没有问题的

```
cookie: SINAGLOBAL=37089483263.85686.1597846118028; SUBP=0033WrSXqPxfM725Ws9jqgMF55529P9D9WFHe8sUxIJF5SEcjA7MyoiM5JpX5KzhUgL.Fo-ESh5RSKzN1K-2dJLoIXXLxKqLBonL1h-LxK.LBKeL12-LxKML1heL1hnLxK.L1h-L1KzLxK-LB-BLBKBLxKML12zLB-eLxKBLBonLB.2LxK-L12qL12zEe0e0e7tt; SUHB=0oUYMUGs8Zps8H; ALF=1629382997; wvr=6; Ugrow-G0=589da022062e21d675f389ce54f2eae7; SUB=_2A25yOnndDeRhGeNM71IZ9SzLwjmIHXVRTuwVrDV8PUNbn9ANLU_DkW9NTgPndp9fl94l7W5u784WanulVoyNJXZi; YF-V5-G0=b588ba2d01e18f0a91ee89335e0afaeb; wb_view_log_5240852795=1366*7681; _s_tentry=www.baidu.com; UOR=,,www.baidu.com; Apache=247952996886.66672.1597901228120; ULV=1597901228191:2:2:2:247952996886.66672.1597901228120:1597846118035
```

观察发现，cookies 中会变化的是 SUB，SUHB 和 ALF，这三个参数变化后就无法登陆以及转发信息了

**user-agent**

这个请求头信息也是必填的

```
user-agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.97 Safari/537.36
```

**origin**

这个也是必填否则无法转发。origin 用于 post 请求，用于说明最初请求从哪发起的

```
origin: https://weibo.com
```

**referer**

也是必填的，和 origin 类似

```
referer: https://weibo.com/5240852795/profile?rightmod=1&wvr=6&mod=personnumber&is_all=1
```

**x-requested-with**

必填，如果`request.getHeader("x-requested-with")`不为 null 则为 ajax 异步请求否则为同步

```
x-requested-with: XMLHttpRequest
```

## 填写请求体

我这里拿自己之前的一个转发接口的请求体举例

```
pic_src:
pic_id:
appkey:
mid: 4539989524753680
style_type: 1
mark:
reason: 哈哈哈
location: page_100505_home
pdetail: 1005055240852795
module:
page_module_id:
refer_sort:
rank: 0
rankid:
isReEdit: false
_t: 0
```

我通过转发不同微博，然后在在线字符串比较网站中比较请求体的时候发现如下几点信息：

**mid**

mid 指的是哪一条微博

**reason**

reason 是转发填写的信息

**location**

这个参数目前不是太清楚，但是昨天和今天没有变过，我们可以注意到其中有一个数字 100505

**pdetail**

这个参数目前作用也不明确，但是它是由两个数字拼接而成的 location 中的 100505 和 5240852795 这个数字，并且我们知道这个 5240852795  数字请求头信息的 referer 中是有的，而且在登录的 get 请求中接口后直接带的参数中也有这个数字

**style_type**

目前还未知，猜测 1 表示转发

**要注意**

要注意的是我搜索出关注的人的微博然后再转发，接口请求数据会有不同，location 从 page_100505_home 变成了 v6_content_home，pdetail 从有值变成了没有值，然后 group_source 变成了 group_all，rid 成为 0_0_0_1413343101395464761_0_0_0

如果要顺便转发时候评论，要加上`is_comment_base: 1`这样的条件

<u>但是经过我的尝试还是上面给出的样式请求体参数不变，然后就是把 mid 换成相应微博号即可</u>

## 成功样式

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200826212612837.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiY251bGw=,size_16,color_FFFFFF,t_70#pic_center)




# sina 点赞接口介绍

类比着转发接口

## 接口

```
https

weibo.com

POST

/aj/v6/like/add?ajwvr=6&__rnd=1597901313249
```

## 请求参数

```
location	page_100505_home
version	mini
qid	heart
mid	${myNoComMid}
like_src	1
cuslike	1
floating	0
_t	0
```

这是我的请求，前三个参数没变过，mid 表示要点赞的是哪个微博，后四个参数也没变过，如果执行两边这个接口，参数且不变，那么点赞就会被取消！



# sina 搜索所有关注人的当天微博

## 当日关注人微博搜索接口

```
https

GET

weibo.com

/u/5240852795/home
```

5240852795 这个和你的微博账号相关

## 请求体参数

```
pids: Pl_Content_HomeFeed
is_ori: 1
is_text: 1
is_pic: 1
is_video: 1
is_music: 1
is_article: 1
key_word: @微博抽奖平台
start_time: 2020-08-20
end_time: 2020-08-20
gid: 
is_new: 
is_search: 1
is_searchadv: 1
ajaxpagelet: 1
ajaxpagelet_v6: 1
__ref: /u/5240852795/home?is_ori=1&is_text=1&is_pic=1&is_video=1&is_music=1&is_article=1&key_word=%40微博抽奖平台&start_time=选择日期&end_time=2020-08-20&gid=&is_new=&is_search=1&is_searchadv=1#_0
_t: FM_1597929836756190
```

前面 6 个 is_ 是表示如下

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020082621262762.png#pic_center)


key_word 是关键信息，两个 time 是开始时间和结束时间，这个 _t 应该是时间戳，应该没用，以后可以考虑不传这个

并且发现如果 __ref 和 _t 不传参数也是可行的，那我们就不传

## 从响应体获取微博号 mid

筛选 Doc，响应体中会有 mid 微博号以及微博信息相关数据，可以从中爬取数据

**响应体内的文本**

返回的是 Doc，里头返回有页面微博信息以及页面微博 mid 号

具体截图信息如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020082621253875.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FiY251bGw=,size_16,color_FFFFFF,t_70#pic_center)



**层级分析**

具体我们可以找到 class 中有 WB_from 且依据层级中里头的 div 会有微博信息，以后根据微博信息是什么来匹配具体的 a 标签的 name

```html
<div class=""\"WB_cardwrap" mid="\"4540055177138080\"">
	...
	<div class="\"WB_from" s_txt2\">
        <a></a>
        <a suda-data="\&quot;key=tblog_home_new&amp;value=feed_time:4540055177138080:frommainfeed\&quot;"></a>
        <a></a>
        <div class="\"PCD_user_b">
            <a class="\"S_txt2\""></a>
            <div class="\"WB_text">
                <a class="\"S_txt2\"">
                    abccba
                </a>
                <div class="\"WB_tag"></div>
            </div>
        </div>
        </div>
    </div>
	...
</div>
```

# sina 下拉刷新接口介绍

## 刷新接口

由于下拉时候新的微博才会显现，我抓取了一下，发现了是这个接口起作用

```java
https

GET

weibo.com

/aj/mblog/fsearch
```

## 请求体与原理

我这里有个请求体样式

```java
ajwvr: 6
pre_page: 1
page: 1
end_id: 4539031667086562
min_id: 4538960766831898
is_ori: 1
is_text: 1
is_pic: 1
is_video: 1
is_music: 1
is_article: 1
key_word: @微博抽奖平台
start_time: 2020-08-17
end_time: 2020-08-17
gid: 
is_new: 
is_search: 1
is_searchadv: 1
pagebar: 0
__rnd: 1598202053308
```

需要详细说明的是，当我的微博搜索哪一天的数据时候，默认是显示 15 条，我们需要注意这几个参数：

- end_id
- min_id
- pagebar
- __rnd

其他参数像什么 is_ 打头的都和转发接口是一样的，是搜索条件，然后 key_word 是搜索关键字，还有搜索开始和结束时间也是搜索条件

要注意的是微博搜索接口传回的数据会拿到 15 条 mid，剩余的数据必须通过下来刷新才能获取，当我们下拉刷新时候就会执行`/aj/mblog/fsearch`这个接口，end_id 会传已有的第一个 mid 号码，min_id 会传拿到的 15 个数据中最后一个 mid 号，这时候此接口又返回了 15 个数据，由于一共有 36 条微博，所以执行一遍此接口此时一共拿到了 30 条微博，所以还需要再执行一遍`/aj/mblog/fsearch`，但是这时候 end_id 依然不变，min_id 变成了第 30 个 mid 号码。目前发现两次接口中的 pagebar 第一次时 0 第二次是 1 目前不知道原因，但是发现不传 pagebar 实际上默认是 0，以后必须得传每次加一的值否则不能回响正确响应！也不知道为什么两次接口中的 __rnd 是不一样的

## 获取当天具体多少条微博的接口

# 防止封 IP 措施

目前采用的策略是每个请求之间隔了 0.2 s

# Cookies 要修改的解决办法

目前的做法是隔几天 cookies 变动，再去手动更换

# 关于 unicode 响应结果

目前还不太明确为什么微博有的会返回正常的值，有的会返回 unicode 编码的值，如果使用 unicode 编码的响应会用正则失败，因此我采用一个方法先将结果中所有的 unicode 编码的字符串转为 utf-8，然后再从响应结果中去正则匹配

# 目前已知

- 转发接口中 cookies 以及请求体里以及搜索所有关注人的接口中有`5240852795`这个数据，这个数据是与账号有关，为微博 id 身份
- 转发接口请求体中的 mid 猜测是具体微博编号

# 目前未知

- 转发接口中的`domain=100505&__rnd=1597901313249`，可能与微博账号有关的随机数
- 转发接口中 cookie 中和 referer 中以及请求体中都有`5240852795`目前还不知道是什么，这个最可能与账号有关

# 目前技术瓶颈

技术瓶颈目前在 jmx 文件中，运行时有一个报红的接口，但不影响，其中写了有哪些瓶颈

