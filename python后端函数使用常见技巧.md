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
>
### 11.在订单列表中显示优惠券信息
> 有时我们希望在订单列表中显示优惠信息，但实际情况是，即便你将优惠券字段存入了订单表中，但是最后遍历订单信息时，通过关联查询能查询到一系列该订单优惠券的相关信息，但是完全找不到有关reduction的内容。因此我的思路是，加载订单列表时，并不马上将查询结果直接赋值给state.items。而是先声明一个局部变量，将查询结果赋值给局部变量，通过数组循环的方法，循环这个局部变量，在循环内部通过当前订单的coupon_card_id的coupon_id字段，也就是优惠券模板id，调用系统接口查询优惠券详情信息。最后将查询到的优惠券内的reduction通过赋值给局部变量循环条目的一个属性，这样一来，这个局部变量比以往的items来说，其子项多了一个reduction属性。最后将其赋值给state.items
```

```

### 12.前端find()的巧用

> 某种场景你希望从一个已经存在的数组中，查询一个标识值已知的条目，例如你的items状态中，你希望能查到id为101的条目信息。你可以在前端执行如下代码
>
> ```javascript
> a = items.find((item)=>{return{item.id == 101})
> ```
>
> 以上方法比较高效。

### 13.吸顶tab切换

> 过去我凭借自己的直觉，将tab的状态与原子组件tab的值双向绑定，最后考虑到有一个全部选项，其实在前端控制器中根据tab筛选不同订单状态的订单时，可以先做个判断，土一点的方法就是if 判断。
>
> 但如果希望在一条代码中完成判断并赋值，可以如下操作
>
> ```javascript
> 订单状态精确等于 = state.activetab === 'default' ? undefined : state.activetab 
> ```
>
> 当tab原子组件处于default时，数据查询筛选的条件就灵活切换到undefined

### 14.创建新数据行默认字段配置

> 起初如果遇到部分字段，在每次存储值阶段，有时有有时没有。在存值的时候，每次都只能判断一次，有就存，没有就不存。其实在前端就可以把这个问题解决掉。当你在前端传值时，可以这样赋值
>
> ```javascript
> sku = item.data.sku != null ? item.data.sku : {}
> ```
>
> 当然在后端函数，from_dict()方法中，需要多一个参数fill_default=true

### 15.前端对象数组针对某个属性排序

> 例如地理位置场景，每个区域对于用户的distance属性各有不同，如果需要根据这个属性对整个数组重新排序，则可以在前端通过如下方式,执行代码
>
> ```javascript
> area.sort((a,b)=>{return a.distance - b.distance})
> ```
>
> 当相减时，如果结果大于零，则将会把小的对象放在更前面，如果小于零，则符合条件不改变。最后能够根据distance属性将整个数组重新由小到大排序。

### 16.mutation外部修改状态

> 部分场景需要通过执行代码的方式修改状态变量。如果直接执行代码，会报错。因此比较正规的方式，是先声明局部变量，通过深度拷贝的方式将状态的值赋值给局部变量。然后局部变量执行代码，最后将局部变量的值赋值给状态变量即可。

### 17.监听用户注册事件参数

> 当用户注册完成后，你希望能够马上执行一些操作，例如在自己的user表中增加一行数据，初始化信息。可以通过如下方式实现。
>
> ```python
> @app.register_event()
> async def on_siteuser_create(event, context):
>     user_repo = context.get_rows_repo('user')
>     user_data = {
>         'siteuser_id': event['siteuser'].id,
>         'is_position': False,
>         'level': '一级',
>         'discount': '110',
>     }
>     user_entity = await user_repo.async_from_dict(user_data, fill_default=True)
>     user_entity = await user_repo.async_save(user_entity)
>     return user_entity
> ```
>
> 其中event['siteuser']是这个系统级事件的参数，通过获取其id属性可以直接将其赋值给字典中的属性。需要注意的是这里的id总是很容易以这种形式发生错误：event['siteuser'] ['id']

### 18.充值功能实现

> 虽然绝大多数情况客户不会有这个要求，但不可避免的我还是遇到了。
>
> 首先充值以后的余额是保存在siteuser分销余额当中。充值余额的页面可以类似于各大移动运营商的充值话费的页面布局。用户发起充值以后，通过后台接口快速提交一个订单，其中的参数product_name可以指定为"余额充值"，price可以是其充值的金额。其他的就不细说了。根据提交的订单返回_pay_url跳转到指定页面让用户完成付款，最后在后端云函数中声明系统事件on_order_pay.
>
> 在该事件中的order参数中的几个数据提取出来，调用后端接口充值该siteuser的分销余额（因为历史原因将用户的余额储存在这个位置）
>
> ```python
> @app.register_event()
> async def on_order_pay(event, context):
>     # 如果是余额充值类订单支付，则直接在用户的分销余额中增加指定的余额
>     if event['order']['product_name'] == '余额充值':
>         siteuser_id = event['order']['siteuser_id']
>         money = event['order']['price'] * 100
>         data = {
>             'siteuser_id': siteuser_id,
>             'money': money,
>             'remark': f'用户余额充值{money}(单位：分)'
>         }
>         a = await context.async_call('post/pshopwalletitem_admin/do_pshop_recharge', data, as_admin=True)
>         return dict(status='success', msg='余额充值成功！', object=a)
> ```
>
> 

### 19.判断查询集是否为空

> 有些场景需要判断查询集是否为空，不为空才执行一系列动作。因此查询后可以通过exists()方法进行如下代码判断,如果查询结果为entity，则可以直接通过if else判断是否为空
>
> ```python
> async def test3(event, context):
>  user_repo = context.get_rows_repo('user')
>  user_entity = user_repo.active().filter(data__siteuser_id=123)
>  if user_entity.exists():
>      return '有'
>  else:
>      return '无'
> 
> async def test4(event, context):
>  user_repo = context.get_rows_repo('user')
>  user_entity = await user_repo.active().filter(data__siteuser_id=123).async_first()
>  if user_entity():
>      return '有'
>  else:
>      return '无'
> ```
>
> 

### 20.获取company的entity对象

> 安全性高的接口，调用时可能需要参数as_admin_company参数及其对应的entity，例如处理分销余额、确认、取消订单等场景。因此需要获取到company的entity对象。代码如下
>
> ```python
> from shared.company.repos import CompanyRepo
> company_id = context.get_company_id()
> company = await CompanyRepo().async_get(id=company_id)
> ```
>
> 

### 21.滑动功能的实现

> 滑动功能主要监听两个事件。
>
> 1.触摸移动
>
> 2.触摸结束
>
> 通常购物车条目时一个数组，需要给每个条目增加两个属性，
>
> 1.class(用于动态绑定每个子项的类，通过改变类从而改变样式实现滑动)
>
> 2.slide(切换滑动状态显示删除按钮)
>
> 在触摸移动时，设置开始位置横坐标。对比当前位置位移差超过一个固定的数值则改变该条目的类。
>
> 当触摸结束时，将开始位置的横坐标清空。
>
> 但滑动的容器应该是一个绝对容器，且应该放在一个相对容器当中。
>
> ```javascript
> console.log('exec', event)
> let clientX
> if (dw.platform == 'dwapp') {
>   clientX = event.data.touches[0].clientX
> } 
> let moveX = clientX - startX
> if (!start) {
>   startX = clientX
>   start = true
> } else {
>   // trigger
>   if (moveX < -60) {
>     console.log(' move left')
>     return true
>   }
> }
> ```
>
> 

### 21.常量访问循环条目

> bagForItem

### 22.自定义字符串模板

> 有些场景需要使用一些具有功能性的原子组件，例如价格格式化，限制长度字符串等等。但最终的需求可能在此基础上还要在前后增加简单的字符串。因此最简单的方法就是克隆你需要的原子组件，在其中wxss文件中，首先找到该组件对应的类名，然后如下执行
>
> ```css
> .youClassName::before{
>     context:"your_content"
> }
> 
> .youClassName::after{
>     context:"your_content"
> }
> ```
>
> 

### 23.es6对象重构赋值

> 对象赋值快捷方式。代码如下
>
> ```javascript
> { a } = {a:1,b:2}
> console.log(a)
> //打印结果为：1
> c = "helloworld"
> console.log({ c })
> //打印结果：{c:"helloworld"}
> 
> ```

### 24.getProps()不支持循环

> 因为组件在循环容器中不支持getProps()获取所有属性，因此，因此在模板中的监听的方法触发后传入的context参数，通常在wjs层中触发emit事件时，会将这个context以字典的形式带出
>
> ```javascript
> import { dw } from 'dstore/papp'
> 
> export default {
>   data: {
>     //
>   },
>   methods: {
>     updateField (arg1, arg2) {
>       const context = dw.platform === 'web' ? arg1 : arg1.target.dataset.bindValue
>       let value = dw.platform === 'web' ? arg2 : arg1.detail.value
>       if (context.type === 'digit') {
>         value = parseInt(value)
>       } 
>       dw.emit('input', value, { context }) // 这里的 context 能够被事件接收方通过 event.context 获取到
>     }
>   }
> }
> ```

### 25.小程序模板中属性的意义

> 首先看一下常规小程序组件的模板中有哪些属性，代码如下
>
> ```html
> <wx-view>
>     <wx-input 
>         class="b-input"
>         :value="context.value"
>         dw-event="bind:input:updateField"
>         data-bind-value="context"
>         @update:value="updateField(context, $event)"
>     ></wx-input>
> </wx-view>
> ```
>
> 1. value用于绑定初始化表单的数据，如果不绑定该属性，那么在初始化阶段表单将无法获得事先绑定传递的状态数据或者其他数据。当编辑其内容时，依然可以正常使用。
> 2. update:value,这是vue用于监听更新value事件绑定的属性，每当表单的value更新时，会触发该事件调用wjs中声明的method.其中需要注意的是，context,$event这两个参数都有很重要的意义。如标题24所述，循环容器中的组件无法获得getProps()中应有的组件属性。为此，每当事件触发调用方法时，都最好将完整的context做为参数传递给逻辑层。当逻辑层获取这个参数时，就可以大有一番作为。想象一个场景，当你循环一个数组，展示了很多条目，每个条目都有一个按钮。当你点击按钮时，需要能够分辨究竟点击了哪一个按钮。而实现传递过来的context则很好的解决了这个问题，其中的eventData属性可以帮你分辨循环条目。而第二个参数$event则是当前正在编辑的这个表单中的事件数据。当你输入"helloworld"时，$event就是该值，而此使的context仍然是改变之前的那个context，直至方法中的emit触发后，相关的值才会发生改变。
> 3. dw-event="bind:input:updateField"，该属性为了兼容小程序对于input事件的监听
> 4. data-bind-value="context"，该属性为了兼容小程序，触发事件时将context作为参数传递给wjs端。

### 26.巧用localStorage

> 想象一个场景，当一个页面存在广告信息需要让30分钟内没看过广告的用户看一次广告。或许你可以在用户表中增加一个字段，这个字段代表用户上一次看广告的时间。每次进入页面都对比时间判断用户是否需要看广告。但这样查表直接影响了app的运行速度。因此可以使用localStorage对象。每次进入页面都获取item，没看过广告的用户，对应的item字段为空，就执行看广告的动作。当30分钟内该用户再次进入该页面时，获取item对比时间，符合条件则不执行看广告的动作，避免了查表，提高了效率。代码如下
>
> ```javascript
> //进入页面后执行如下代码
> result = localStorage.getItem("ad_time")
> //获取当前时间与1970年相差的毫秒数赋值给now变量
> let now = Date.now()
> if(result == null){
>     localStorage.setItem("ad_time",now)
>     console.log("show ad")
> }
> else{
>     if((now - result) < 1800000){
>         localStorage.setItem("ad_time",now)
>         console.log("show ad")
>     }
> }
> ```
>
> 

### 27.前端数据映射

> 订单状态数据内容通常为英文，为了良好展示为中文需要将其进行映射。代码如下
>
> ```javascript
> map = (state)=>{
>     const _map = {
> 	  'withdraw':'提现',
>   	  'income':'进账',
>   	  'sell_back':'撤销买入',
>   	  'error_back':'订单过期但是支付回撤',
>       'pay':'支付订单',
>       'admin': '后台解冻',
>       'reversal': '冻结',
>       'recharge':'充值',
>       'reward_income':'悬赏收入',
>       'reward_expense':'悬赏支出',
>       'reward_withdraw':'悬赏提现',
>       'level_package':'购买升级包',
> 	}
>    return _map(state)
> }
> map("admin")
> //"后台解冻"
> ```
>
> 创建一个对象，返回对象中的指定字段，这个字段为传入的参数

### 28.小程序wxss样式不要用px

> 在wxss文件中使用pm作为单位，在web端会转化为rem,在小程序将转化为rpx.

### 29.避免在大量循环中查询数据

> 在数据量比较大的场景中，如果在循环块查询数据表可能会导致查询次数过多数据量过大导致任务超时失败。比如有一张user表，有100个用户，你需要遍历100个用户的查询其对应的age属性。这种情况下可以考虑一次查询获取列表数据，然后在for循环中找到对应的数据即可。例子如下：
>
> ```python
> user_repo = context.get_rows_repo('user')
> user_entities = user_repo.active().filter(data__field='your_field').to_dict_list()
> # some code
> for item in user_entities:
>     if item['data']['siteuser_id'] == your_object['data']['siteuser_id']:
>         # then you can use item to do something 
> ```

### 30.系统表的分组聚合

> 在普通数据表中的聚合查询示例如下：
>
> ```python
> # 查询不同名称的数据并分别计数
> result2 = (
>     repo
>     .annotate(name='data__name')
>     .group_by('name', sum=repo.Sum('data__price__float'))
> )
> ```
>
> 以上内容根据name字段分组，并且将price字段求和。需要注意的是，这里的字段名称以及双下划线标识要非常小心。聚合中的形参为字段名称，实参需要带有data及双下划线的样式,而分组字段却又只是字段名称，而最后需要求和的字段price还要写为上文的形式，稍有不注意就很容易搞错。
>
> 而系统表的分组聚合查询则需要通过以下代码实现
>
> ```python
> order_map = list(
>         OrderRepo()
>         .filter(
>             company_id=company_id,
>             papp_slug="product",
>             namespace="default",
>             states="success",
>             created__gte=start,
>             created__lte=end,
>         )
>         ._queryset.values("siteuser_id")
>         .order_by("siteuser_id")
>         .annotate(sum=Sum("price"))
>     )
> ```

### 31.查询字段为空

> 有些场景需要查询没有父级分销商的siteuser,因此根据pshop_parent_id为空作为条件筛选用户。查询条件如下即可：
>
> ```python
> entity = repo.active().filter(pshop_parent_id=None)
> ```
>
> 

### 32.通过查询系统用户表获取分销关系树

> ```python
> @app.register_func()
> async def get_full_tree(event, context):
>     # 首先找到顶级分销商，输出为列表top_list
>     company_id = await context.async_get_company_id()
>     top_siteuser_entitys = await SiteUserRepo().filter(object_id=company_id, pshop_parent_id=None).async_to_dict_list()
>     top_list = []
>     for item in top_siteuser_entitys:
>         a = {
>             'id': item['id'],
>             'children': [],
>         }
>         top_list.append(a)
>     # 为了减少不必要的数据查询次数，首先一次性找到所有用户id与pshop_parent_id对应关系，存放在列表my_map中
>     my_map = []
>     map_entitys = await SiteUserRepo().filter(object_id=company_id).async_to_dict_list()
>     for item in map_entitys:
>         my_object = {
>             'id': item['id'],
>             'pshop_parent_id': item['pshop_parent_id']
>         }
>         my_map.append(my_object)
>     full_tree = []
>     for item in top_list:
>         result = await find_children(event, context, item, my_map)
>         if len(result['children']) > 0:
>             full_tree.append(result)
>     return full_tree
> 
> 
> @app.register_func()
> async def find_children(event, context, my_dict, my_map):
>     result = {
>         'id': my_dict['id'],
>         'children': []
>     }
>     # 根据获取的映射关系，查找是否有对应的children,放入children这个临时变量
>     children = []
>     for item in my_map:
>         if item['pshop_parent_id'] == my_dict['id']:
>             param_dict = {
>                 'id': item['id'],
>                 'children': []
>             }
>             children.append(param_dict)
>     if len(children) > 0:
>         for item in children:
>             result['children'].append(await find_children(event, context, item, my_map))
>         return result
>     else:
>         return result
>     
>     
> async def func(event, context):
>     ctx = context
>     from shared.account.repos import StaffRepo, StaffPermissionRepo
>     taiji = StaffRepo.Model.objects.get(id=107)
>     permission = StaffPermissionRepo.Model.objects.get(
>         field='dissue.read_list_all'
>     )
>     taiji.staff_permission.add(permission)
>     return context.return_success()
> ```
>
> 思路如下：
>
> 1. 为了避免重复查询系统表，在函数开头完整的查询表中所有数据，将其中的id以及phop_parent_id映射关系存储在一个dict_list中。
> 2. 根据实现获取的顶级分销商dict_list,通过循环条目依次调用find_children函数。
> 3. find_children函数接收一个字典，该字典只有两个字段，分别是id以及children.
> 4. find_children函数最终需要根据接受的字典而返回一个字典。因此一开始可以直接声明一个用来返回的字典。
> 5. find_children接受的另一个参数则是分销关系map列表。遍历map列表查找是否有与参数字典id对应的pshop_parent_id，如果有则代表该用户有下级用户。存储到临时变量children中。通过判断children的长度分析该用户是否有下级用户。如果没有则直接返回开始声明的那个字典。如果有，则遍历children，循环中调用自己并把结果插入到实现声明的字典中的列表字段。最终返回这个字典。
> 6. 这种递归动作的核心在于你能否想象到最后一次递归的动作应该有什么要的结果。返回的值如何控制，列表字典嵌套的情况下如何通过for循环构造字典或者列表。

### 33.快速格式化日期时间

> 过去常常使用以下方式格式化时间日期
>
> ```python
> date1 = '2019-06-05' #string格式
> date1 = datetime.strptime(date1, "%Y-%m-%d") #首先变成datetime格式
> date2 = date1+ datetime.timedelta(days=-10))# 然后就可以进行计算了，得到datetime格式
> date3 = date2.strftime("%Y-%m-%d") # 变成string格式
> ```
>
> 有些场景仅仅只是需要获取当前时间，或者当前时间的简单加减法。可以考虑使用isoformat快速格式化时间日期使得该数据类型能够被系统所接受。
>
> ```python
> import datetime()
> now = datetime.datetime.now()
> data = {
>     'time': now.isoformat()
> }
> # then you can save this data to your own dataset
> ```

### 34.事务装饰器

> 某些一连串的操作动作需要保持一致性，例如某个订单完成，首先改用户余额，然后改产品数量，然后改订单状态。可能修改产品数量的时候发现小于0报错返回。可是用户余额已经修改了，因此需要将整个动作放在一个事务装饰器下，保证操作的统一性，要么一起执行要么一起不执行。在函数的顶部增加一个如下的装饰器即可：
>
> ```python
> @app.async_transaction()
> async def your_function(event, context) 
> ```
>
> 但有一个问题，如何判定步骤执行失败，如果只是返回status=error显然不算失败，因此为了抛出一个错误，往往需要手动操作如下：
>
> ```python
> if result['status'] == 'error':
> 	raise Exception("状态错误")
> ```

### 35.获取用户siteuser_entity

> 通常我之前的习惯是直接获取siteuser_id，但是这样的操作如果需要用户其他的信息依然很麻烦。因此直接获取siteuser_entity即可。所有数据都能够继而得到。
>
> ```python
> siteuser_entity = await context.async_get_siteuser_entity()
> ```

### 36.重构代码由获取配置信息开始

> 有时配置表中的信息有很多，需要获取配置信息的场景也很多。可是每次只是需要获取到配置表中的一个字段。这种情况可以考虑新建一个setting文件，在该文件中实现一个普通函数，专门用于获取配置表信息。每个需要获取配置信息的场景中都import setting中的这个函数即可。
>
> ```python
> # my_setting文件中
> async def get_config(context, key):
>     repo = context.get_rows_repo('config')
>     entity = await repo.active().async_first()
>     if not entity:
>         return None
>     data = entity.data[key]
>     return data
> ```
>
> 在你需要获取配置信息的场景中，如下操作
>
> ```python
> from .my_setting import get_config
> your_data = await get_config(context, your_key)
> ```

### 37.获取字典中的字段设置默认值

> 假设有一个字典如下所示
>
> ```python
> dict = {
> 	'id': 1,
> 	'name': 'jerry'
> }
> ```
>
> 获取name可以通过另一种方法get(),从而当有可能不存在name字段时，可以为他准备一个默认值：
>
> ```python
> name = dict.get('name',tom)
> ```

### 38.通过schema数据校验

> 排除一些复杂的数据校验场景需要在前端通过正则表达式判断内容，为了友好的展示后端云函数所接收的参数以及描述性内容，建议使用SCHEMA参数数据校验具体实现方式如下
>```python
> YOUR_SCHEMA = {
>   'num':  app.value_object.Num(default=0, description='测试数字'),
>   'str': app.value_object.String(required=True,default='test', description='测试数字'),
> }
> @app,register_func(post_sechema=YOUR_SCHEMA)
> async def test(event, context):
>   '''
>   用于检测云函数
>
>   '''
>   data = app.value_object.Value.parse_data(event, YOUR_SCHEMA)
>   context.log(data)
>   return data
>```
> 当传入的参数不符合要求标准，函数将带错误类型返回。
