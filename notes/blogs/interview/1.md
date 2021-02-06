---
title: vue是如何实现数据响应的？
categories:
 - 面试相关
tags:
 - vue
---

vue2.x使用 `Object.defineProperty()`，vue3.0 `Proxy`

# vue2.x

递归遍历data中的数据，使用`Object.defineProperty()`劫持。

::: details 查看部分源码
``` js
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
```
:::

## 语法

`Object.defineProperty(obj, prop, descriptor)`

`obj`: 要定义属性的对象

`prop`: 要定义或修改的属性的名称或Symbol。

`descriptor`: 要定义或修改的属性描述符。

返回被传递给函数的对象

### descriptor可选参数

`configurable`: 是否可以删除目标属性

`enumerable`: 是否可以枚举

`writable`: 属性的值是否可以被重写 *不能和get，set同时存在*

`get`

`set`


## 为什么vue2.x中不能给监听到新属性的变化，需要用到`$set`

自己实现一个Observer试试
``` js
class Observer {
    constructor(data) {
        // 遍历参数data的属性,给添加到this上
        for (let key of Object.keys(data)) {
        if (typeof data[key] === "object") {
            data[key] = new Observer(data[key]);
        }
        Object.defineProperty(this, key, {
            enumerable: true, 
            configurable: true, 
            get() {
            console.log("get", key, data[key]);
            return data[key]; 
            },
            set(newVal) {
            console.log("set", key, newVal);
            if (newVal === data[key]) {
                return;
            }
            data[key] = newVal;
            }
        });
        }
    }
}

const obj = {
    name: "app",
    age: 18,
    a: {
        a: 1,
        b: 2
    }
};

let app = new Observer(obj);

// get set 监听到
app.name = "123";
console.log(app.name);

// set 监听不到
app.custom = "custom";
console.log(app.custom);

app.a.b = "ab";
console.log(app.a.b);
```

打印结果如下：

```
// set name 123
// get name 123
// 123

// custom

// get a Observer {}
// set b ab
// get a Observer {}
// get b ab
// ab
```

可以看出对`custom`的复制，并没有监听到。这也就是为什么在vue2.x中给对象添加新的属性需要用到[$set](https://juejin.cn/post/6844904142507360269)的原因了。

大致逻辑，判断传入的target是否合法，然后判断target对象上面是否有key，并且key不是Object原型对象的属性。说明这个key已经被监听到了，直接返回val。
否则去执行[defineReactive](https://juejin.cn/post/6844903760771153933)方法

# vue3.0 使用了proxy

使用`proxy`实现Observer

## 语法
代理Object对象，用于控制和修改Object的基本行为，比如：属性调用，赋值，删除属性等

`const p = new Proxy(target, handler)`

target: 被代理的对象
handler: 代理`p`的行为

## 使用proxy实现Observe

``` js
class Observe {
    constructor(obj) {
        return new Proxy(obj, {
        get(target, propKey, receiver) {
            console.log("get", propKey, receiver);
            return Reflect.get(target, propKey, receiver);
        },
        set(target, propKey, value, receiver) {
            console.log("set", propKey, value);
            Reflect.set(target, propKey, value, receiver);
        }
        });
    }
}

const obj = {
    name: "app",
    age: 18,
    a: {
        a: 1,
        b: 2
    }
};

const ob = new Observe(obj);

ob.age = "20";
console.log(ob.age);

ob.custom = "custom";
console.log(ob.custom);

ob.a.b = "ab";
console.log(ob.a.b);
```

结果如下：

``` js
// set age 20
// get age Proxy {name: "app", age: "20", a: {…}}
// 20

// set custom custom
// get custom Proxy {name: "app", age: "20", a: {…}, custom: "custom"}
// custom

// get a Proxy {name: "app", age: "20", a: {…}, custom: "custom"}
// get a Proxy {name: "app", age: "20", a: {…}, custom: "custom"}
// ab
```

可以看出，新增的属性`proxy`一样能监听到

# 总结

| Object.defineProperty()                                                                  | Proxy                                              |
| ---------------------------------------------------------------------------------------- | -------------------------------------------------- |
| 会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性，并返回此对象。不能直接监听 | 创建一个对象的代理，从而实现基本操作的拦截和自定义 |
| 不能直接监听新增属性                                                                     | 可以监听新增属性                                   |

- [proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
- [Object.defineProperty()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)