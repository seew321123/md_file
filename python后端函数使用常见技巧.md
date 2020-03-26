# python后端函数使用常见技巧
### 1.快速构造字典
```
a = 123
b = 456
return dict(a=a,b=b)
```
 __以上内容返回一个字典__

### 2.格式化字符串访问接口
> 如用户的主域名为'http://www.baidu.com'
> 接口名为'test'
> 则拼接方式如下
```
a = 'http://www.baidu.com'
b = 'test'
c = f'{a}/{b}'
```
> 这也支持一些复杂的计算，如下
```
a = 123.123456
b = f'{round(a,2)}元'
print(b)
```
> 通过以上的操作可以快速将a保留小数点后两位输出带价格的字符串


### 3.根据当前siteuser_id获取该用户所有信息
> 接口：post/psiteuser/get_siteuser_data
```
user =await context.async_call('post/psiteuser/get_siteuser_data'，data)
return user
```
> 我个人认为这个功能非常鸡肋，因此我推崇一种功能比较全面的请求方式。
```
from shared.sitez.repos import SiteUserRepo
user_id =await SiteUserRepo().filter(nickname='herojery').async_first()
```
> 对于以上代码的解释,获取siteuser数据表中，nickname为herojery的所有数据的第一个条目，可以直接使用to_dict()查看其中数据
>

```python
if not user_id:
    return 0
else:
    return 1
```
> 如果你为了将客户服务器中的用户信息与我方siteuser中的信息绑定，可以将手机号储存到siteuser数据中，然后当用户登录时，我们马上根据当前手机号判断siteuser表中是否有相关数据，用这个方法比较稳。

### 4.获取当前登录用户的siteuser_id
```python
siteuser_id = await context.async_get_siteuser_id()
```
> 即便在前端很容易获得，但这个东西可能还是需要记一下。
>
> 

### 5.编辑数据常规操作

```python
user_repo = context.get_rows_repo('user')
entity =await user_repo.active().filter(id=16909021).async_first()
entity.data['nick_name'] = '复杂男孩'
entity =await user_repo.async_save(entity)
```
> 总的来说，修改少量数据可以通过这种方式，但如果需要整体修改，则可以先创建一个字典
```python
data = dict(
	shop_level = '〇级商家',
	nick_name = '贼复杂男孩'
 )
entity = await user_repo.async_from_dict(data)
entity = await user_repo.async_save(entity)
```

### 6.两种模式的关联查询

> 1.假如遍历申诉表中所有申诉订单，同时显示每个申诉订单的订单号以及申诉人的信息，可是订单号储存在order表中申诉人的信息在user表中，因此同时需要查询order表中的数据的id

```python
async def test10(event, context):
    complain_repo = context.get_rows_repo('complain')
    select_related = {
        'toone': {
            'order_uuid__order': {},
            'Plaintiff_uuid__user': {}
        }
    }
    complain_entity = complain_repo.active().apply_pagination(select_related=select_related)
    return complain_entity['objects']
```

> 上文返回的是一个数组，可以随意遍历，比较舒服。


> 2.某种情况中，我已知当前订单编号，希望通过订单编号查询申诉表中的数据，可是申诉表中根本没有订单编号这个字段，因此这种以外键数据为查询条件的方式，需要通过如下方法。

```python
@app.register_func()
async def test11(event, context):
    complain_repo = context.get_rows_repo('complain')
    entity = await complain_repo.active().async_filter_by_toone('data__order_uuid', 'order', id=19415414)
    a = []
    for i in entity:
        a.append(i.to_dict())
    return a
```

> 需要注意的是，这里的entity变量并不能像以往那样通过async_first获取第一个对象，也不能通过values()获取其值，因此我特意将得到的结果一个一个存储至数组中，并返回这个数组。
> 实际上这种查询条件的应用场景应该是比较少的，但是知识这种东西，只能说多多益善。

### 7.将query_set转化为列表返回

> 如上文的情况，这个entity虽然可以遍历，但却不是数组，因此我们可以通过下文的语法糖完成

```python
a = await [item.to_dict() async for item in repo]
return a
```

> 快速创建数组并遍历queryset,将所有所有条目都赋值给列表a的子项。

### 8.云函数内支持的系统事件

```
on_order_submit
on_order_pay
on_order_success
on_order_cancel
on_order_user_cancel
on_order_system_cancel
```

### 9.某些场景可能需要我们计算查询结果的数量
```
order_repo = context.get_rows_repo('order')
entity = order_repo.count()
```
> 上面这个方法将返回一个数量。
> 与之对应的还有sum、max、min、avg等，功能不言而喻。

### 10.推送微信消息
> 通过认证的微信公众号，可以在公众号管理页面设置模板消息。模板消息可以理解为一个一个的表单，表单数据中的key不变，而value可以动态赋值。首先应该将需要发送的若干种信息，转化为若干个消息模板。请求发送模板消息时，还有一些例如行业信息等参数，具体可以参考微信官方的讲解内容https://developers.weixin.qq.com/doc/offiaccount/Message_Management/Template_Message_Interface.html
> 在担路后端调用发送模板消息的接口示例如下
```python
python
@app.register_func()
async def func_name(event, context):
    # 以下为调用例子
    params = {
        'company_id': 500, # 当前公司的id
        'industry_name': 'IT科技-互联网|电子商务', # 行业的第一个选项 + '-' 行业第二个选项 这个是模板所属的行业
        'template_id': "TM00002", # 模板的id
        'data': {
            "touser": "wqeqrqwrhiuryierr", # appuser的openid
            "miniprogram": { # 如果有小程序的话
                "appid": 'wx1asdasdasd',
                "pagepath": 'home/index',
            },
            "url": 'ww.baidm.com', # 网页地址
            "topcolor": '#ffff', hex 格式的颜色, 标题颜色
            "data": { # 具体的内容
                "keyword1": {
                    "value": "微信影城影票",
                    "color": '#ffff' # hex 格式的颜色,
                },
            }
        },
    }
    await context.call("post/wechattemplatecode/send_wechat_template_message", params)
```
> 和微信官方的文档不同的时，在这个示例中我们有看到带有access_token的请求过程，我的猜测是，后台调用了company_id对应的商家事先提交的许可证。
> 结合实际使用情况，例如交易市场中，如果订单状态发生变化，比如卖家接单，此阶段应该通过公众号发送给卖家相关信息。调用一个比如叫做send_massage的后端函数，传入的参数可以是与订单相关的所有信息，以及接受信息一方用户的open_id。将信息按照模板传入字典数据，最后发起请求。