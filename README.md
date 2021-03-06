2017-02-11 更新：   
1. 消息回复内容中增加了is_replay变量，用于解决replay时重复写入的问题   
2. 增加了一个自定义UnexpectAnswer异常，用于静默处理用户的不合法输入。   

具体请看下面的说明和代码注释

# wechat-dialogs
一个微信公众号处理复杂会话的轮子，使用python generator和redis管理对话上下文   
可以比较方便的用微信的自动回复功能与用户进行交互

下面是一个简单的累加器实现：

```python
def accumulator(to_user):
    yield None
    msg_content, is_replay = yield None
    
    num_count, is_replay = yield ('TextMsg', '您需要累加几个数字？')
    try:
        num_count = int(num_count)
    except Exception:
        return ('TextMsg', '输入不合法！我们需要一个整数，请输入"开始"重新开启累加器')
    res = 0
    for i in range(num_count):
        num, is_replay = yield ('TextMsg', '请输入第%s个数字, 目前累加和:%s' % (i+1, res))
        try:
            num = int(num)
        except Exception:
            return ('TextMsg', '输入不合法！我们需要一个整数，请输入"开始"重新开启累加器')
        res += num
        
    # 注意：最后一个消息一定要用return不要用yield！return用于标记会话结束。
    return ('TextMsg', '累加结束，累加和: %s' % res)
```

效果图：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/4610828-e4d47cdc45d03c89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 部署demo

代码中附带了一个Flask写的小demo，可以直接部署测试。另外代码是在python3下写的。

安装步骤如下：   
1.  根据requirement.txt安装依赖包（其实就redis和flask..）   
2.  安装一个Redis，知道它的IP地址，端口号，密码等信息   
3.  在demo_dialog.py的开头更改相应的REDIS配置信息   
4.  启动demo_server.py: python ./demo_server.py   
5.  去微信公众平台绑定公众号服务器   

目前有三个DEMO：   
1.  回复"累加器"玩一玩累加，这是基本功能   
2.  回复"会话记录"看一下如何防止数据重复写入   
3.  回复"会话菜单"了解如何静默切换会话逻辑   

# 关于UnexpectAnswer

想一下这个场景：当公众号让用户回复YES或者NO的时候，用户回复了一个START会怎么样呢？

现在有两种处理方式：   
1.  回复一个信息告诉用户他的输入有误，让他重新输入。 --- 这个通过return实现   
2.  直接找一下是否有逻辑处理START这条信息。 --- 这个通过raise UnexpectAnswer实现   

# 应用到其他项目

1. 编写自己的对话逻辑，类似demo_dialog.py   
2. 在服务器代码中用wechat.bot.answer(post_data, dialog_module)获取相应的回复

# 存在的问题

这里用重现消息的方式来保存会话状态，所以在会话过程中做数据改动要慎重，比如   

```python
answer = yield xxx
<write database>
answer = yield xxx
```

在重现过程中<write database>会被多次执行，可能导致重复数据插入。   

解决的办法：
*  写操作统一在return的时候做   
*  写操作尽量用UPDATE不要用INSERT，避免重复插入   
*  直接用singleton代替redis，在进程内存中存储generator，不过这样就不适用prefork多进程的服务器..   
*  新增加is_replay返回值。在代码中可以通过这个值来判断这次调用是否是replay造成的，避免重复写入。   

相关博客：   
*  http://www.jianshu.com/p/974cf38291ec   
*  http://blog.arthurmao.me/2017/02/python-redis-wechat-dialog   
