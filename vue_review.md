## Vue组件的复习

#### 1.组件属性传递参数

```javascript
props:['inf'],
template:`<div>{{ msg }}</div>`
data:function(){
	return {
		msg:this.inf * 2
	}
}
```

这样可以将父组件传递过来的属性inf通过简单的处理赋值到组件的data中，然后直接操作data.如果data按照es6箭头函数的样子，则会出错。随后我试了一下计算属性同样不能使用类似于()=>{}这种模式

还有一个需要注意的点，当你通过属性内部的data处理传入的参数并返回给模板的情况下，这种数据只会在渲染阶段赋值一次，当你在父组件中通过v-model动态绑定数值，且数值发生变化时，子组件中并不会产生任何改变。	

#### 2.子组件向父组件传递信息(有更简单的语法糖)

通常的操作方法是，在子组件模板中添加普通的事件例如@click等事件，以这种方式出发一个自定义事件，也就是$emit()触发事件，其中可以带有参数，而参数可以是子组件中的data数据通过一定的处理得到的值，最终可以动态影响父组件中的内容显示。具体代码如下

```javascript
<div id="app">
    <div>{{ total }}</div>
    <my-niubi 
    @event_add="handle"
    @event_reduce="handle"></my-niubi>
</div>
<script>
    Vue.component('my-niubi', {
        props:{
            message:Number
        },
        template:`
            <div>
            <button @click="add">+1</button>
            <button @click="reduce">-1</button>
            </div>`,
        data:function(){
            return{
                counter: 0
            }
        },
        methods:{
            add:function(){
                this.counter++
                this.$emit('event_add',this.counter)
            },
            reduce:function(){
                this.counter--
                this.$emit('event_reduce',this.counter)
            },
        }
    })
var app = new Vue({
    el: '#app',
    data:{
        total: 0
    },
    methods:{
        handle:function(total){
            this.total = total
        }
    }
})
</script>
```

语法糖的使用前提，是父组件必须要绑定v-model属性。在父组件中监听事件，修改data数据，可以通过直接通过v-model绑定一个值，而子组件中出发的特殊的事件$emit("input",this.counter)，这就相当于你在输入框内直接输入了一个值，而这个值就是你通过emit事件传递过来的参数。通常我认为只有input标签才能绑定v-model属性，但是这种情况下似乎任何元素都可以。

也可以通过在子组件中的input输入框通过v-model动态绑定内部的data值，然后出发input事件以后，发起一个$emit（）事件，将动态绑定的data作为参数传入父组件，而出发的$emit实际上可以是一个input事件，也就自动将子组件中的数据直接动态绑定到父组件上了。

#### 3.通过子组件索引访问子组件信息

有些情景需要在主组件中主动访问子组件的数据，如果实现在子组件中配置其子组件索引则可以解决这个问题。代码如下

```javascript
<div id="app">
			<div @click="get_msg" >一个按钮</div>
			<my-niubi ref="comA"></my-niubi>
		</div>
		<script>
			Vue.component('my-niubi', {
				props:['value'],
				template:`
					<div>子组件</div>`,
				data:function(){
					return {
						msg: "从子组件中获取的信息"
					}
				},
			})
			var app = new Vue({
			  el: '#app',
			  data:{
				  total: 0
			  },
			  methods:{
				  get_msg:function(){
					  console.log("打印子组件的信息",this.$refs.comA.msg)
				  }
			  }
			})
		</script>
```

其中需要注意一点，访问子组件索引时，this.$refs ，这个s很容易忘记。

#### 4.slot插槽

在子组件模板中，可以增加一对<slot></slot>插槽标签，这样在父组件使用子组件时，可以在子组件标签中填入内容，当没有填入任何内容时，插槽内的默认内容将会显示。

```javascript
<div id="app">
			<my-niubi ref="comA">替换默认文本的内容</my-niubi>
		</div>
		<script>
			Vue.component('my-niubi', {
				props:['value'],
				template:`
					<div>
						<slot>
							<p>默认文本</p>
						</slot>
					</div>`,
			})
			var app = new Vue({
			  el: '#app',
			})
		</script>
```

#### 5.递归组件

你可以声明一个组件，并在组件的模板中调用自己，通过这种操作实现组件的递归。但前提是你必须要有限制条件，否则会产生类似于套娃的效应导致报错。如下

```javascript
<div id="app">
    <my-com :count="1"></my-com>
</div>
<script>
	Vue.component('my-com',{
        name:"my-com",
        props:['count'],
        template:`<div><button>
		<my-com 
		:count="count + 1"
		v-if="count < 3"></my-com></div>`
    })
	var app = New Vue({
        el: "#app"
    })
</script>
```

通过属性count的传递值以及if的限制，让这个组件只会被递归2次。