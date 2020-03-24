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