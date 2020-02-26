---
title: vue3.0监听之proxy
date: 2020-02-26 15:35:51
tags:
---
## 什么是 Proxy？

[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy) 上是这么描述的——`Proxy`对象用于定义基本操作的自定义行为（如属性查找，赋值，枚举，函数调用等）。

其实就是在对目标对象的操作之前提供了拦截，可以对外界的操作进行过滤和改写，修改某些操作的默认行为，这样我们可以不直接操作对象本身，而是通过操作对象的**代理对象**来间接来操作对象，达到预期的目的~

Proxy 可以理解成，在目标对象之前架设一层“拦截”，外界对该对象的访问，都必须先通过这层拦截，因此提供了一种机制，可以对外界的访问进行过滤和改写。Proxy 这个词的原意是代理，用在这里表示由它来“代理”某些操作，可以译为“代理器”。



```javascript
    let obj = {
        a : 1
    }
    let proxyObj = new Proxy(obj,{
        get : function (target,prop) {
            return prop in target ? target[prop] : 0
        },
        set : function (target,prop,value) {
            target[prop] = 888;
        }
    })
    
    console.log(proxyObj.a);        // 1
    console.log(proxyObj.b);        // 0

    proxyObj.a = 666;
    console.log(proxyObj.a)         // 888
```

ES6 原生提供 Proxy 构造函数，用来生成 Proxy 实例。



```jsx
var proxy = new Proxy(target, handler);
```

> Proxy 对象的所有用法，都是上面这种形式，不同的只是handler参数的写法。其中，new Proxy()表示生成一个Proxy实例，target参数表示所要拦截的目标对象，handler参数也是一个对象，用来定制拦截行为。

简单demo（拦截读取属性行为）：



```jsx
var proxy = new Proxy({}, {
  get: function(target, property) {
    return 35;
  }
});

proxy.time // 35
proxy.name // 35
proxy.title // 35
```

> 上面代码中，作为构造函数，Proxy接受两个参数。第一个参数是所要代理的目标对象（上例是一个空对象），即如果没有Proxy的介入，操作原来要访问的就是这个对象；第二个参数是一个配置对象，对于每一个被代理的操作，需要提供一个对应的处理函数，该函数将拦截对应的操作。比如，上面代码中，配置对象有一个get方法，用来拦截对目标对象属性的访问请求。get方法的两个参数分别是目标对象和所要访问的属性。可以看到，由于拦截函数总是返回35，所以访问任何属性都得到35。

> 注意，要使得Proxy起作用，必须针对Proxy实例（上例是proxy对象）进行操作，而不是针对目标对象（上例是空对象）进行操作。

> **如果handler没有设置任何拦截，那就等同于直接通向原对象。**

demo2:



```jsx
var target = {};
var handler = {};
var proxy = new Proxy(target, handler);
proxy.a = 'b';
target.a // "b"
```

> 上面代码中，handler是一个空对象，没有任何拦截效果，访问proxy就等同于访问target。

## 技巧：一个技巧是将 Proxy 对象，设置到object.proxy属性，从而可以在object对象上调用。

如下所示：



```javascript
var object = { proxy: new Proxy(target, handler) };
```

Proxy 实例也可以作为其他对象的原型对象。



```javascript
var proxy = new Proxy({}, {
  get: function(target, property) {
    return 35;
  }
});

let obj = Object.create(proxy);
obj.time // 35
```

上面代码中，proxy对象是obj对象的原型，obj对象本身并没有time属性，所以根据原型链，会在proxy对象上读取该属性，导致被拦截。

同一个拦截器函数，可以设置拦截多个操作。



```javascript
var handler = {
  get: function(target, name) {
    if (name === 'prototype') {
      return Object.prototype;
    }
    return 'Hello, ' + name;
  },

  apply: function(target, thisBinding, args) {
    return args[0];
  },

  construct: function(target, args) {
    return {value: args[1]};
  }
};

var fproxy = new Proxy(function(x, y) {
  return x + y;
}, handler);

fproxy(1, 2) // 1
new fproxy(1, 2) // {value: 2}
fproxy.prototype === Object.prototype // true
fproxy.foo === "Hello, foo" // true
```

对于可以设置、但没有设置拦截的操作，则直接落在目标对象上，按照原先的方式产生结果。
 下面是 Proxy 支持的拦截操作一览，一共 13 种。

- get(target, propKey, receiver)：拦截对象属性的读取，比如proxy.foo和proxy['foo']。
- set(target, propKey, value, receiver)：拦截对象属性的设置，比如proxy.foo = v或proxy['foo'] = v，返回一个布尔值。
- has(target, propKey)：拦截propKey in proxy的操作，返回一个布尔值。
- deleteProperty(target, propKey)：拦截delete proxy[propKey]的操作，返回一个布尔值。
- ownKeys(target)：拦截Object.getOwnPropertyNames(proxy)、Object.getOwnPropertySymbols(proxy)、Object.keys(proxy)、for...in循环，返回一个数组。该方法返回目标对象所有自身的属性的属性名，而Object.keys()的返回结果仅包括目标对象自身的可遍历属性。
- getOwnPropertyDescriptor(target, propKey)：拦截Object.getOwnPropertyDescriptor(proxy, propKey)，返回属性的描述对象。
- defineProperty(target, propKey, propDesc)：拦截Object.defineProperty(proxy, propKey, propDesc）、Object.defineProperties(proxy, propDescs)，返回一个布尔值。
- preventExtensions(target)：拦截Object.preventExtensions(proxy)，返回一个布尔值。
- getPrototypeOf(target)：拦截Object.getPrototypeOf(proxy)，返回一个对象。
- isExtensible(target)：拦截Object.isExtensible(proxy)，返回一个布尔值。
- setPrototypeOf(target, proto)：拦截Object.setPrototypeOf(proxy, proto)，返回一个布尔值。如果目标对象是函数，那么还有两种额外操作可以拦截。
- apply(target, object, args)：拦截 Proxy 实例作为函数调用的操作，比如proxy(...args)、proxy.call(object, ...args)、proxy.apply(...)。
- construct(target, args)：拦截 Proxy 实例作为构造函数调用的操作，比如new proxy(...args)。

# Proxy this 问题

> 虽然 Proxy 可以代理针对目标对象的访问，但它不是目标对象的透明代理，即不做任何拦截的情况下，也无法保证与目标对象的行为一致。主要原因就是在 Proxy 代理的情况下，目标对象内部的this关键字会指向 Proxy 代理。



```javascript
const target = {
  m: function () {
    console.log(this === proxy);
  }
};
const handler = {};

const proxy = new Proxy(target, handler);

target.m() // false
proxy.m()  // true
```

上面代码中，一旦proxy代理target.m，后者内部的this就是指向proxy，而不是target。
 下面是一个例子，由于this指向的变化，导致 Proxy 无法代理目标对象。



```javascript
const _name = new WeakMap();

class Person {
  constructor(name) {
    _name.set(this, name);
  }
  get name() {
    return _name.get(this);
  }
}

const jane = new Person('Jane');
jane.name // 'Jane'

const proxy = new Proxy(jane, {});
proxy.name // undefined

```

上面代码中，目标对象jane的name属性，实际保存在外部WeakMap对象_name上面，通过this键区分。由于通过proxy.name访问时，this指向proxy，导致无法取到值，所以返回undefined。
 此外，有些原生对象的内部属性，只有通过正确的this才能拿到，所以 Proxy 也无法代理这些原生对象的属性。



```javascript
const target = new Date();
const handler = {};
const proxy = new Proxy(target, handler);

proxy.getDate();
// TypeError: this is not a Date object.

```

上面代码中，getDate方法只能在Date对象实例上面拿到，如果this不是Date对象实例就会报错。这时，this绑定原始对象，就可以解决这个问题。



```javascript
const target = new Date('2015-01-01');
const handler = {
  get(target, prop) {
    if (prop === 'getDate') {
      return target.getDate.bind(target);
    }
    return Reflect.get(target, prop);
  }
};
const proxy = new Proxy(target, handler);

proxy.getDate() // 1

```

# 三、Proxy 与Object.defineProperty

## 使用Object.defineProperty

ES5 提供了 Object.defineProperty 方法，该方法可以在一个对象上定义一个新属性，或者修改一个对象的现有属性，并返回这个对象。

- Object.defineProperty(obj, prop, descriptor)

  参数：

  - obj: 要在其上定义属性的对象。
  - prop: 要定义或修改的属性的名称。
  - descriptor: 将被定义或修改的属性的描述符。

下面一个简单的输入框变化，数据展示



```javascript
var obj = {
    value:''
}
var value = '';
Object.defineProperty(obj,"value",{
    get:function(){
        return value
    },
    set:function(newVal){
        value = newVal
    }
})
document.querySelector('#input').oninput = function(){
    var value = this.value;
    obj.value = value;
    document.querySelector('#text').innerHTML = obj.value;
}

```

当然一般不会只是改变一个属性,如下所示，遍历劫持对象所有的属性



```javascript
//要劫持的对象
const data = {
    name:''
}
  //遍历对象,对其属性值进行劫持
  Object.keys(data).forEach(function(key){
    Object.defineProperty(data,key,{
        enumerable: true,
        configurable: true,
        get: function() {
            console.log('get');
          },
          set: function(newVal) {
            // 当属性值发生变化时我们可以进行额外操作
            console.log(`我修改了成了${newVal}`);           
          },
    })
  })
  data.name = 'gg'

```

这个只是简单的劫持

**缺点：**Object.defineProperty的第一个缺陷,无法监听数组变化

**区别**：

- Proxy可以直接监听对象而非属性
- Proxy直接可以劫持整个对象,并返回一个新对象,不管是操作便利程度还是底层功能上都远强于Object.defineProperty。
- Proxy可以直接监听数组的变化
- Proxy有多达13种拦截方法,不限于apply、ownKeys、deleteProperty、has等等是Object.defineProperty不具备的。