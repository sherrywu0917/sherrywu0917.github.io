---
title: core-decorators源码简析——readonly
date: 2018-04-17 15:40:08
tags: [core-decorators, decorator, readonly, js]
---
直接看源码，`readonly.js`代码非常简短，核心函数是`handleDescriptor`，通过修改描述对象descriptor的writable属性为false，将目标属性置为不可修改。
``` js
import { decorate } from './private/utils';

function handleDescriptor(target, key, descriptor) {
  descriptor.writable = false;
  return descriptor;
}

export default function readonly(...args) {
  return decorate(handleDescriptor, args);
}

```
其中`decorate`函数定义在utils文件中，具体如下:
<!-- more -->

``` js
export function isDescriptor(desc) {
  if (!desc || !desc.hasOwnProperty) {
    return false;
  }

  const keys = ['value', 'initializer', 'get', 'set'];

  for (let i = 0, l = keys.length; i < l; i++) {
    if (desc.hasOwnProperty(keys[i])) {
      return true;
    }
  }

  return false;
}

export function decorate(handleDescriptor, entryArgs) {
  if (isDescriptor(entryArgs[entryArgs.length - 1])) {
    console.log(entryArgs);
    console.log(entryArgs[entryArgs.length - 1].initializer);
    return handleDescriptor(...entryArgs, []);
  } else {
    return function () {
      return handleDescriptor(...Array.prototype.slice.call(arguments), entryArgs);
    };
  }
}
```
首先判断传过来的最后一个参数是否是descriptor对象，若该参数有`['value', 'initializer', 'get', 'set']`属性中的任一个，则认为是descriptor对象，直接调用`handleDescriptor`去处理。
若所传最后一个参数不是descriptor对象，则返回一个匿名函数，可以接收额外的参数。

#### descriptor对象
基本内容如下：
- configurable控制是否能删、修改descriptor本身。
- writable控制是否能修改值value。
- enumerable控制是否能枚举出属性。
- value是该属性对应的值，可以是任何有效的JavaScript值（数值，对象，函数等）。
- get和set控制访问属性的读和写逻辑。

其中，前三个属性是一定存在的，`value`和`get/set`属性不会并存。当装饰器作用于类属性时，`descriptor`将变成一个叫“类属性描述符”的东西，其区别在于没有`value`和`get或set`，且多出一个`initializer`属性，类型是函数，***在类构造函数执行时，`initializer`返回的值作为属性的值***，***在类构造函数执行后，`initializer`属性被替换为value属性***。

#### 使用
``` js
class Meal {
  @readonly // 或者@readonly()
  entree = 'steak';
}

var dinner = new Meal();
console.log('--------对象创建后----------')
console.log(Object.getOwnPropertyDescriptors(dinner));
dinner.entree = 'salmon';
```
输出的descriptor前后对比：
<img src="/image/readonly_console.png" width="400px">
其中`initializer`函数内部即为`return 'steak'`。
同时会报错<font color=#F44336>Uncaught TypeError: Cannot assign to read only property 'entree' of object '#<Meal>'</font>，提示该属性只读。
