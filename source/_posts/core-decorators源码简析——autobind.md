---
title: core-decorators源码简析——autobind
date: 2018-04-12 20:22:06
tags: [core-decorators, decorator, autobind, js]
---
## 入口函数
先看入口函数，通过`export default`暴露的autobind函数，通过`...args`形式的rest参数获取函数的多余参数，这儿是所有参数。通过decorator形式有两种调用方式：
- @autobind()，会返回一个匿名函数`function (...argsClass) {return handle(argsClass);}`，argsClass为最终decorator传入的参数，函数内部返回了handle函数；在修饰器外面再封装一层函数的方式，可以用于接收额外的参数；
- @autobind，args即为decorator传入的参数，直接调用handle函数

``` js
/**
 * 两种调用方式
 *
 * 2. @autobind
 * @param  {...[type]} args [description]
 * @return {[type]}         [description]
 */
export default function autobind(...args) {
  if (args.length === 0) {
    return function (...argsClass) {
      return handle(argsClass);
    };
  } else {
    return handle(args);
  }
}
```
## handle函数
handle函数根据参数的长度去判断修饰类型，当args参数长度为1时，是对类的修饰，该参数即所要修饰的目标类；当长度为3时，是对方法的修饰，参数和Object.defineProperty的参数相对应`target, name, descriptor:{value, configurable, enumerable, writable}`。
``` js
function handle(args) {
  if (args.length === 1) {
    return autobindClass(...args);
  } else {
    return autobindMethod(...args);
  }
}
```
## 对类的修饰
- `Object.getOwnPropertyDescriptors`会以键值对的形式返回某对象属性的描述对象descriptor；
- `getOwnKeys`方法会返回一个数组，包含目标类原型对象上所有的属性的键名，包括不可枚举属性以及Symbol属性
`autobindClass`遍历目标类原型上的所有方法，若某个属性对应的value类型是`function`并且属性不是`constructor`，则用`Object.defineProperty`对该属性进行重新赋值，调用`autobindMethod`实现对该方法的autobind；
<!-- more -->
``` js
function autobindClass(klass) {
  const descs = getOwnPropertyDescriptors(klass.prototype);
  const keys = getOwnKeys(descs);

  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i];
    const desc = descs[key];

    if (typeof desc.value !== 'function' || key === 'constructor') {
      continue;
    }
    defineProperty(klass.prototype, key, autobindMethod(klass.prototype, key, desc));
  }
}
```
`getOwnKeys`方法的实现如下所示：
``` js
export const getOwnKeys = getOwnPropertySymbols
    ? function (object) {
        return getOwnPropertyNames(object)
          .concat(getOwnPropertySymbols(object));
      }
    : getOwnPropertyNames;
```
其中`getOwnPropertySymbols`会返回包含对象所有Symbol属性的数组，`getOwnPropertyNames`会返回包含自身所有属性（不包括Symbol属性，但包括不可枚举属性）的数组。
此外，`Object.assign`拷贝只会拷贝属性的值，并不会拷贝它的赋值方法set和取值方法get，结合`Object.defineProperties(target, Object.getOwnPropertyDescriptors(source))`可以实现对象属性取值、赋值方法的拷贝。

## 对方法的修饰
`autobindMethod`第一个参数是类的原型对象，第二个参数是要修饰的属性名，第三个参数是该属性的描述对象，它通过修改描述对象的get赋值方法，实现了对方法的自动绑定，返回修改后的descriptor。赋值方法内不同条件分别对应的调用场景为：
1. `this === target`: 调用方式`let method = Child.prototype.childMethod`，this就是`Child.prototype`，不绑定直接return。
2. `this.constructor !== constructor && getPrototypeOf(this).constructor === constructor`:  调用方式`let method = Child.prototype.parentMethod`，子类没有该方法，`this.constructor`是`Child.prototype.constructor`，`constructor`是`Parent.prototype.constructor`，二者不等，但是`this.__proto__.constructor`和`Parent.prototype.constructor`是相同的(在es6的继承中`Child.prototype.__proto__` === `Parent.prototype`)，不绑定直接return。
3. `this.constructor !== constructor && key in this.constructor.prototype`: 子类调用父类的方法，仍然绑定为子类的this；如果不做该判断，直接到下一步bind(fn, this);则父类的方法会覆盖子类，下次调用就只执行父类中的方法了。
4. 其他：调用`bind(fn, this)`绑定上下文。
``` js
function autobindMethod(target, key, { value: fn, configurable, enumerable }) {
  if (typeof fn !== 'function') {
    throw new SyntaxError(`@autobind can only be used on functions, not: ${fn}`);
  }

  const { constructor } = target;

  return {
    configurable,
    enumerable,

    get() {
      // Class.prototype.key lookup
      // Someone accesses the property directly on the prototype on which it is
      // actually defined on, i.e. Class.prototype.hasOwnProperty(key)
      if (this === target) {
        return fn;
      }

      console.log(target, key)
      // Class.prototype.key lookup
      // Someone accesses the property directly on a prototype but it was found
      // up the chain, not defined directly on it
      // i.e. Class.prototype.hasOwnProperty(key) == false && key in Class.prototype
      // Object.getPrototypeOf作用：
      // 1. 用于获取一个实例对象的原型对象
      // 2. Object.getPrototypeOf(ColorPoint) === Point 用来判断一个类是否继承了另一个类
      if (this.constructor !== constructor && getPrototypeOf(this).constructor === constructor) {
        return fn;
      }

      // Autobound method calling super.sameMethod() which is also autobound and so on.
      // this.constructor: Child constructor: Parent
      if (this.constructor !== constructor && key in this.constructor.prototype) {
        return getBoundSuper(this, fn);
      }

      const boundFn = bind(fn, this);

      defineProperty(this, key, {
        configurable: true,
        writable: true,
        // NOT enumerable when it's a bound method
        enumerable: false,
        value: boundFn
      });

      return boundFn;
    },
    set: createDefaultSetter(key)
  };
}
```
实际调用时的例子：
``` js
@autobind
class Person {
  constructor() {
  }

  getPerson() {
    console.log('parent called, AWESOME METHOD');
    return this;
  }
}

@autobind
class Man extends Person {
  constructor() {
    super();
  }

  getMan() {
    return this;
  }
}

//四种方法分别对应上面列出的四种场景
console.log('---Man.prototype.getMan---');
let getProtoMan = Man.prototype.getMan;
console.log(getProtoMan());

console.log('---Man.prototype.getPerson---');
let getProtoPerson = Man.prototype.getPerson;
console.log(getProtoPerson());

console.log('---man.getPerson---');
let man = new Man();
let getPerson = man.getPerson;
console.log(getPerson() === man);

console.log('---man.getMan---');
let man2 = new Man();
let getMan = man2.getMan;
console.log(getMan() === man2);
```
输出结果如下图所示：
<img src="/image/autobind_console.png" width="800px">
## getBoundSuper函数
### 作用
给子类Man添加一个方法`getPerson`，方法内通过super调用父类的getPerson方法，`super.getPerson`对应上述场景3；如果不调用getBoundSuper函数，子类的getPerson方法在执行`super.getPerson`时会被父类覆盖，会导致下面方法只有第一次会输出'parent called, AWESOME METHOD'和'child called, AWESOME CHILD METHOD'，其他四次则只输出'parent called, AWESOME METHOD'。
``` js
@autobind
class Man extends Person {
  constructor() {
    super();
  }

  getPerson() {
    console.log('child called, AWESOME CHILD METHOD');
    super.getPerson();
    return this;
  }

  getMan() {
    return this;
  }
}

for (var i = 0; i < 5; ++i) {
  console.log('invoking');
  man.getPerson();
  console.log('---');
}
```

### WeakMap的特性：
> 1. 只能用对象obj作为key值
> 2. WeakMap的键名对应的对象不计入垃圾回收机制，如果对应的对象被清除，垃圾回收机制就会释放该对象所占用的内存，即WeakMap中的对应的键名对象和键值也会自动消失，不用手动删除
适用场景：WeakMap的专用场合就是，它的键所对应的对象，可能会在将来消失。WeakMap结构有助于防止内存泄漏。

通过getBoundSuper可以将父类方法绑定到子类this上，并且使用WeakMap存储已经绑定的方法，在mapStore中key值对应子类实例，不同的子类实例对应不同的WeakMap，若该子类被删除，对应的WeakMap对象也会被清除。
``` js
let mapStore;
function getBoundSuper(obj, fn) {
  if (!mapStore) {
     mapStore = new WeakMap();
  }

  if (mapStore.has(obj) === false) {
    mapStore.set(obj, new WeakMap());
  }

  const superStore = mapStore.get(obj);

  if (superStore.has(fn) === false) {
    superStore.set(fn, bind(fn, obj));
  }

  return superStore.get(fn);
}
```
