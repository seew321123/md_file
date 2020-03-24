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

### 3.根据当前siteuser_id获取该用户所有信息
> 接口：post/psiteuser/get_siteuser_data
```
user =await context.async_call('post/psiteuser/get_siteuser_data')
return user
```
> 我个人认为这个功能非常鸡肋，因此我推崇一种功能比较全面的请求方式。
```
from shared.sitez.repos import SiteUserRepo
user_id = siteuser = SiteUserRepo().filter(nickname='herojery').values()[0]['id']
```
> 对于以上代码的解释,获取siteuser数据表中，nickname为herojery的所有数据的values属性中的第一个符合条件的数据的id.通过这样的方式可以快速高效获取你想要的内容。
>
> 通过以上的代码，很显然values()后返回的是一个列表。通过判断列表长度可以万无一失的得知当前是否有符合条件的搜索结果。
>
```
siteuser = SiteUserRepo().filter(nickname='herojery').values()
if len(siteuser) == 0:
	your_code
```
> 如果你为了将客户服务器中的用户信息与我方siteuser中的信息绑定，可以将手机号储存到siteuser数据中，然后当用户登录时，我们马上根据当前手机号判断siteuser表中是否有相关数据，用这个方法比较稳。