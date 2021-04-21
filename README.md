## 1、对象进行读写监听

```js
const obj = {
  name: 'layouwen',
}
Object.defineProperty(obj, 'name', {
  configurable: true,
  enumerable: true,
  get() {
    console.log('触发get')
    return 'layouwen'
  },
  set(newValue) {
    console.log('触发set')
    return newValue
  },
})
obj.name // 触发get
obj.name = 'yuouwen' // 触发set
```

## 2、观察者模式

下面有个场景。当儿子说要出去玩的时候，爸爸告诉孩子不能出去玩。

```js
const father = {
  eat() {
    console.log('不给出去玩！准备吃饭了')
  },
}

const son = {
  play() {
    console.log('爸，我出去玩会~')
  },
}

son.play()
```

我们新建一个事件触发器

```js
const EventObj = new EventTarget()
```

在执行 play 方法是，通知爸爸

```js
EventObj.addEventListener('callFather', father.eat)

const son = {
  play() {
    console.log('爸，我出去玩会~')
    EventObj.dispatchEvent(new CustomEvent('callFather'))
  },
}
```

## 3、发布订阅模式

跟上面的场景一致，我们换个思路实现。首先创建一个保存需要执行函数的队列。提供两个方法：添加新的任务 `addSub`，执行所有任务 `notify`。

```js
class Dep {
  constructor() {
    this.subs = []
  }
  addSub(sub) {
    this.subs.push(sub)
  }
  notify() {
    this.subs.forEach(item => item.update())
  }
}
```

创建一个用于新建任务的类，提供一个方法：执行自己的任务 `update`。

```js
class Watcher {
  constructor(callback) {
    this.callback = callback
  }
  update() {
    this.callback()
  }
}
```

实例化 Dep，用于将新建的任务加入到队列中。

```js
const dep = new Dep()
```

创建两个对象，模拟妈妈和爸爸。

```js
const father = {
  eat() {
    dep.addSub(
      new Watcher(() => {
        console.log('爸爸：不给出去玩！准备吃饭了')
      })
    )
  },
}

const mother = {
  eat() {
    dep.addSub(
      new Watcher(() => {
        console.log('妈妈：不给出去玩！准备吃饭了')
      })
    )
  },
}
```

执行里面的方法，使其加入到等待任务中

```js
father.eat()
mother.eat()
```

此时创建一个儿子对象

```js
const son = {
  play() {
    console.log('儿子：爸，我出去玩会~')
    dep.notify()
  },
}
```

我设置一个延迟，在 2 秒回触发儿子的 `play` 方法。看看是否会将爸爸和妈妈中的等待任务给执行。

```js
setTimeout(son.play, 2000)
```

结果

```
儿子：爸，我出去玩会~
爸爸：不给出去玩！准备吃饭了
妈妈：不给出去玩！准备吃饭了
```

## 4、观察模式模拟 Vue 的数据监听响应

先实现通过正则表达式，将`{{value}}`内的值替换成，data 中的数值

```js
class LVue {
  constructor(option) {
    this.$option = option
    this._data = option.data
    this.compile()
  }
  compile() {
    const el = document.querySelector(this.$option.el)
    this.compileNodes(el)
  }
  compileNodes(el) {
    /* 获取所有子节点 */
    const childNodes = el.childNodes
    childNodes.forEach(node => {
      /* 如果为元素节点，并且该节点内部还有内容，就继续进行遍历编译 */
      if (node.nodeType === 1 && node.childNodes.length > 0) {
        /* 元素节点 */
        this.compileNodes(node)
      } else if (node.nodeType === 3) {
        /* 文本节点 */
        const textContent = node.textContent
        // 创建正则。匹配{{}}中的内容
        const reg = /\{\{\s*([^\{\}\s]+)\s*\}\}/g
        /* 匹配成功的就是我们需要的内容 */
        if (reg.test(textContent)) {
          // 获得匹配到的内容
          const name = RegExp.$1
          // 替换内容
          node.textContent = node.textContent.replace(reg, this._data[name])
        }
      }
    })
  }
}

new LVue({
  el: '#app',
  data: {
    name: 'layouwen',
    age: 22,
    address: '广州市荔湾区',
  },
})
```

我们实现了简单的替换后，我们开始实现数据劫持监听

```js
// 继承 EventTarget 实现监听事件
/* new content start */
class LVue extends EventTarget {
  /* new content end */
  constructor(option) {
    super()
    this.$option = option
    this._data = option.data
    this.observe(option.data)
    this.compile()
  }
  observe(data) {
    const keys = Object.keys(data)
    const that = this
    keys.forEach(key => {
      let value = data[key]
      Object.defineProperty(data, key, {
        configurable: true,
        enumerable: true,
        get() {
          return value
        },
        set(newValue) {
          /* new content start */
          // 触发对应 key 事件
          that.dispatchEvent(new CustomEvent(key, { detail: newValue }))
          /* new content end */
          // 更新闭包中缓存的 value
          value = newValue
        },
      })
    })
  }
  compile() {
    const el = document.querySelector(this.$option.el)
    this.compileNodes(el)
  }
  compileNodes(el) {
    /* 获取所有子节点 */
    const childNodes = el.childNodes
    childNodes.forEach(node => {
      /* 如果为元素节点，并且该节点内部还有内容，就继续进行遍历编译 */
      if (node.nodeType === 1 && node.childNodes.length > 0) {
        /* 元素节点 */
        this.compileNodes(node)
      } else if (node.nodeType === 3) {
        /* 文本节点 */
        const textContent = node.textContent
        // 创建正则。匹配{{}}中的内容
        const reg = /\{\{\s*([^\{\}\s]+)\s*\}\}/g
        /* 匹配成功的就是我们需要的内容 */
        if (reg.test(textContent)) {
          console.log(this, 'this1')
          // 获得匹配到的内容
          const name = RegExp.$1
          // 替换内容
          node.textContent = node.textContent.replace(reg, this._data[name])
          /* new content start */
          // 监听 name 对应的事件
          this.addEventListener(name, e => {
            console.log(this, 'this2')
            const newValue = e.detail
            const oldValue = this._data[name]
            node.textContent = node.textContent.replace(oldValue, newValue)
            console.log('页面数据刷新了')
          })
          /* new content end */
        }
      }
    })
  }
}

const lvue = new LVue({
  el: '#app',
  data: {
    name: 'layouwen',
    age: 22,
    address: '广州市荔湾区',
  },
})
setTimeout(() => {
  lvue._data.name = '梁又文'
  setTimeout(() => {
    lvue._data.age = 23
  }, 1000)
}, 2000)
```

## 5、发布订阅模式版本

通过依赖收集，对用notify触发所有收集的依赖实现响应式。简单模拟了一下v-model、v-text以及v-html

```html
<div id="app">
  <h1>{{name}}</h1>
  <input type="text" v-model="name" />
  <h2>v-text</h2>
  <div v-text="name"></div>
  <div v-html="name"></div>
  <div>
    <p>年龄:{{age}}</p>
    <p>地址:{{address}}</p>
  </div>
</div>
<script>
  class LVue2 {
    constructor(option) {
      this.$option = option
      this.$data = option.data
      this.observer()
      this.compile()
    }
    observer() {
      const keys = Object.keys(this.$data)
      keys.forEach(keyName => {
        const dep = new Dep()
        let value = this.$data[keyName]
        Object.defineProperty(this.$data, keyName, {
          configurable: true,
          enumerable: true,
          get() {
            if (Dep.target) {
              dep.addSub(Dep.target)
            }
            return value
          },
          set(newValue) {
            dep.notify(newValue)
            value = newValue
          },
        })
      })
    }
    compile() {
      const el = document.querySelector(this.$option.el)
      this.compileNodes(el)
    }
    compileNodes(el) {
      const childNodes = el.childNodes
      childNodes.forEach(node => {
        if (node.nodeType === 1) {
          /* new content start */
          // 新增 v-model 属性监听
          let attrs = node.attributes
          ;[...attrs].forEach(attr => {
            const attrName = attr.name
            const attrValue = attr.value
            if (attrName === 'v-model') {
              node.value = this.$data[attrValue]
              node.addEventListener('input', e => {
                this.$data[attrValue] = e.target.value
              })
            } else if (attrName === 'v-text') {
              node.innerText = this.$data[attrValue]
              new Watcher(this.$data, attrValue, newValue => {
                node.innerText = newValue
              })
            } else if (attrName === 'v-html') {
              node.innerHTML = this.$data[attrValue]
              new Watcher(this.$data, attrValue, newValue => {
                node.innerHTML = newValue
              })
            }
          })
          /* new content end */
          if (node.childNodes.length > 0) {
            this.compileNodes(node)
          }
        } else if (node.nodeType === 3) {
          const textContent = node.textContent
          const reg = /\{\{\s*([^\{\}\s]+)\s*\}\}/g
          if (reg.test(textContent)) {
            const valueName = RegExp.$1
            node.textContent = node.textContent.replace(reg, this.$data[valueName])
            new Watcher(this.$data, valueName, newValue => {
              const oldValue = this.$data[valueName]
              node.textContent = node.textContent.replace(oldValue, newValue)
            })
          }
        }
      })
    }
  }

  class Dep {
    constructor() {
      this.subs = []
    }
    addSub(sub) {
      this.subs.push(sub)
    }
    notify(newValue) {
      this.subs.forEach(sub => {
        sub.update(newValue)
      })
    }
  }

  class Watcher {
    constructor(data, key, cb) {
      this.cb = cb
      // 保存实例对象到 Dep 中的 target 中
      Dep.target = this
      // 为了触发 get 收集依赖
      data[key]
      Dep.target = null
    }
    update(newValue) {
      this.cb(newValue)
    }
  }

  const lvue2 = new LVue2({
    el: '#app',
    data: {
      name: '梁又文',
      age: 23,
      address: '广州市荔湾区',
    },
  })
  setTimeout(() => (lvue2.$data.name = '梁文文'), 1000)
  setTimeout(() => (lvue2.$data.name = '我是v-text的内容'), 2000)
  setTimeout(() => (lvue2.$data.name = '<h3>我是v-html的内容</h3>'), 3000)
</script>
```
