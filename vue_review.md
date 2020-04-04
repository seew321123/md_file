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

#### 3.非父子关系组件传递信息

