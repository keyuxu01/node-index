# 双向绑定核心知识点

1、如果一个对象中有属性有方法，那么调用属性可以直接. 就可以调用，但是如果是调用方法的时候，是通过入参来决定key的值来调用的话，请用[]来表示：  
```html
<!DOCTYPE html>
<html lang="en" xmlns:v-on="http://www.w3.org/1999/xhtml">
  <head>
    <meta charset="UTF-8">
      <title>MVVM 双项绑定</title>
      <style>
        #app {
        text-align: center;
        margin-top: 100px;
        color: #888;
      }

        h1 {
        color: #aaa;
      }

        input {
        padding: 0 10px;
        width: 600px;
        line-height: 2.5;
        border: 1px solid #ccc;
        border-radius: 2px;
      }

        .bind {
        color: #766;
      }

        strong {
        color: #05BC00;
      }

        button {
        padding: 5px 10px;
        border: 1px solid #777777;
        border-radius: 5px;
        background: #ffffff;
        color: #777777;
        cursor: pointer;

      }
      </style>
  </head>
  <body>
    <div id="app">
      <h1>Hi，MVVM</h1>
      <input v-model="name" placeholder="请输入内容" type="text">
        <h1 class="bind">{{name}} 's age is <strong>{{age}}</strong></h1>
        <button v-on:click="sayHi">点击欢迎您</button>
    </div>
    <script>
      function observe(data) {
      //如果不是一个对象，直接终止程序
      if (!data || typeof data !== 'object') {
      return false;
    }
      for (let key in data) {
      let val = data[key];
      let subject = new Subject();
      Object.defineProperty(data, key, {
      enumerable: true,
      configurable: true,
      get: function () {
      if (currentObserver) {
      currentObserver.subscribeTo(subject)
    }
      return val
    },
      set: function (newVal) {
      val = newVal;
      subject.notify()
    }
    });
      if (typeof val === 'object') {
      observe(val)
    }
    }
    }

      let id = 0;
      let currentObserver = null;

      /**
      * 订阅者对象
      */
      class Subject {
      constructor() {
      this.id = id++;
      this.observers = []
    }

      addObserver(observer) {
      this.observers.push(observer)
    }

      removeObserver(observer) {
      let index = this.observers.indexOf(observer)
      if (index > -1) {
      this.observers.splice(index, 1)
    }
    }

      notify() {
      this.observers.forEach(observer => {
      observer.update()
    })
    }
    }

      /**
      * 观察者对象
      */
      class Observer {
      constructor(vm, key, cb) {
      this.subjects = {};
      this.vm = vm;
      this.key = key;
      this.cb = cb;
      this.value = this.getValue()
    }

      //如果新旧数据不相同，就直接调用cb方法
      update() {
      let oldVal = this.value;
      let value = this.getValue();
      if (value !== oldVal) {
      this.value = value;
      this.cb.bind(this.vm)(value, oldVal)
    }
    }

      //添加观察者
      subscribeTo(subject) {
      if (!this.subjects[subject.id]) {       //如果当前换擦着中不存在这个当前id的一个对象，那么吧这个对象添加为观察者
      subject.addObserver(this);
      this.subjects[subject.id] = subject     //放在观察者对象中，根据自增id来区分
    }
    }

      getValue() {
      currentObserver = this;
      let value = this.vm.$data[this.key];    //获取vm实例兑现中的data数据
      currentObserver = null;
      return value
    }
    }

      /**
      * 编译对象
      */
      class Compile {
      constructor(vm) {
      this.vm = vm; //vm对象
      this.node = vm.$el; //获取挂载的元素dom
      this.compile();//执行核心功能
    }

      compile() {
      this.traverse(this.node);//传入的参数是挂载元素dom
    }

      traverse(node) {
      if (node.nodeType === 1) {      //节点类型1：element元素
      this.compileNode(node);     //触发节点事件 双向绑定和事件触发
      node.childNodes.forEach(childNode => {
      this.traverse(childNode);       // 递归调用，如果是有子节点，重新递归
    })
    } else if (node.nodeType === 3) {       // 节点类型3： 文本元素
      this.compileText(node);     // 处理文本元素的编译
    }
    }

      // 文本编译入口
      compileText(node) {
      let reg = /{{(.+?)}}/g;
      let match;
      while (match = reg.exec(node.nodeValue)) {      //获取到文本内容
      let raw = match[0]
      let key = match[1].trim()
      node.nodeValue = node.nodeValue.replace(raw, this.vm.$data[key]);
      new Observer(this.vm, key, function (val, oldVal) {     // 订阅者核心方法
      node.nodeValue = node.nodeValue.replace(oldVal, val)
    })
    }
    }

      // 节点编译入口
      compileNode(node) {
      let attrs = [...node.attributes];//获取标签属性
      attrs.forEach(attr => {
      if (this.isModelDirective(attr.name)) { //截取是绑定数据的情况
      this.bindModel(node, attr); //绑定数据
    } else if (this.isEventDirective(attr.name)) { //截取是绑定事件的情况
      this.bindEventHander(node, attr); //触发事件
    }
    })
    }

      /**
       * 双向绑定数据
       * @param node  标签节点
       * @param attr  标签节点的属性名
       */
      bindModel(node, attr) {
      let key = attr.value;// 获取到传递过来的属性的key的值
      node.value = this.vm.$data[key]; //给节点绑定值，对应的值就是vm实例里面data对应key的值
      new Observer(this.vm, key, function (newVal) {
      node.value = newVal
    });
      node.oninput = (e) => { //监听节点的input事件
      this.vm.$data[key] = e.target.value //过去输入框中输入的value值，把这个值放入到vm的data实例中去
    }
    }

      /**
       *
       * @param node
       * @param attr
       */
      bindEventHander(node, attr) {
      let eventType = attr.name.substr(5); //获取节点属性,从第五个下标开始截取后面的字符串作为：key(事件类型)
      let methodName = attr.value; //获取节点的属性的value
      node.addEventListener(eventType, this.vm.$methods[methodName]);//通过事件类型，来触发事件，事件就是vm实例中方法
    }

      //赛选出传入的node属性是v-model的情况
      isModelDirective(attrName) {
      return attrName === 'v-model'
    }

      //赛选出传入的node属性是 v-on的情况
      isEventDirective(attrName) {
      return attrName.indexOf('v-on') === 0
    }
    }

      class mvvm {
      constructor(opts) {     //这里面的函数是实例化的时候执行的
      this.init(opts);
      observe(this.$data);
      new Compile(this);      //变异当前对象
    }

      init(opts) {
      if (opts.el.nodeType === 1) {
      this.$el = opts.el
    } else {
      this.$el = document.querySelector(opts.el)
    }

      this.$data = opts.data || {};
      this.$methods = opts.methods || {};
      //把$data 中的数据直接代理到当前 vm 对象
      for (let key in this.$data) {
      Object.defineProperty(this, key, {
      enumerable: true,
      configurable: true,
      get: () => {
      return this.$data[key]
    },
      set: newVal => {
      this.$data[key] = newVal
    }
    })
    }
      //让 this.$methods 里面的函数中的 this，都指向当前的 this，也就是 vm对象实例
      for (let key in this.$methods) {
      this.$methods[key] = this.$methods[key].bind(this);
    }
    }
    }


      /**
      * 实例化MVVM对象， 主入口
      * @type {mvvm}
      */
      let vm = new mvvm({
      el: '#app',
      data: {
      name: 'YanLe',
      age: 3
    },
      methods: {
      sayHi: function () {
      alert(`hi ${this.name}`)
    }
    }
    });

      let clock = setInterval(function () {
      vm.age++;  //等同于 vm.$data.age

      if (vm.age === 10) clearInterval(clock)
    }, 1000)
    </script>
  </body>
</html>
```

