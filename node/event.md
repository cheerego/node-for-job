## Event循环引用源码解析

```javascript
const EventEmitter = require('events');
let emitter = new EventEmitter();
emitter.on('myEvent', () => {
  console.log('hi');
  emitter.emit('myEvent');
});
emitter.emit('myEvent');
```



这段代码会死循环吗?

```javascript
const EventEmitter = require('events');
let emitter = new EventEmitter();
emitter.on('myEvent', function sth () {
  emitter.on('myEvent', sth);
  console.log('hi');
});
emitter.emit('myEvent');
```
这段代码会死循环吗?

```
第一段代码:RangeError: Maximum call stack size exceeded
第二段代码:hi (输出一次)
```



### EventEmitter.prototype.on 解析
```javascript
EventEmitter.prototype.addListener = function addListener(type, listener) {
  return _addListener(this, type, listener, false);
};

EventEmitter.prototype.on = EventEmitter.prototype.addListener;
```
由此可见 on 方法是 addListener 的别名

```javascript
function _addListener(target, type, listener, prepend) {
  var m;
  var events;
  var existing;
  // 传入的 listener 不是 function 的时候抛出异常
  if (typeof listener !== 'function') {
    const errors = lazyErrors();
    throw new errors.TypeError('ERR_INVALID_ARG_TYPE', 'listener', 'Function');
  }
  // target 传入的是 this(当前类对象),取this._events
  events = target._events;
  // 如果不存在就创建空对象赋值给this,并将_eventsCount
  // 感觉这一步有点多余,因为在 EventEmitter 的构造函数中就进行了初始化
  if (events === undefined) {
    events = target._events = Object.create(null);
    target._eventsCount = 0;
  } else {
    // To avoid recursion in the case that type === "newListener"! Before
    // adding it to the listeners, first emit "newListener".
    if (events.newListener !== undefined) {
      target.emit('newListener', type,
                  listener.listener ? listener.listener : listener);

      // Re-assign `events` because a newListener handler could have caused the
      // this._events to be assigned to a new object
      events = target._events;
    }
    existing = events[type];
  }
  // 代码从一个if else分支执行到这里
  // 当 events === undefined 为真时,existing === undefined为真,说明可以直接将事件名和事件函数绑定到events上
  // 当 events === undefined 为假时,existing = events[type];用来进行判断这个事件是否有绑定,如果 existing === undefined 为假
  // 说明该事件已经绑定过了
  if (existing === undefined) {
    // Optimize the case of one listener. Don't need the extra array object.
    existing = events[type] = listener;
    ++target._eventsCount;
  } else {
    // 这个判断说明绑定过了,且绑定过一次  
    if (typeof existing === 'function') {
      // Adding the second element, need to change to array.
      // 进行第二次绑定,就将时间function改成一个function数组
      existing = events[type] =
        prepend ? [listener, existing] : [existing, listener];
      // If we've already got an array, just append.
    } else if (prepend) {
      existing.unshift(listener);
    } else {
      existing.push(listener);
    }

    // Check for listener leak
    if (!existing.warned) {
      m = $getMaxListeners(target);
      if (m && m > 0 && existing.length > m) {
        existing.warned = true;
        // No error code for this since it is a Warning
        const w = new Error('Possible EventEmitter memory leak detected. ' +
                            `${existing.length} ${String(type)} listeners ` +
                            'added. Use emitter.setMaxListeners() to ' +
                            'increase limit');
        w.name = 'MaxListenersExceededWarning';
        w.emitter = target;
        w.type = type;
        w.count = existing.length;
        process.emitWarning(w);
      }
    }
  }

  return target;
}

```

在这里代码中,进行了:event 的初始化,没有绑定过直接绑定,绑定过了会将其转换成数组

### EventEmitter.prototype.emit 解析

```javascript
EventEmitter.prototype.emit = function emit(type, ...args) {
  let doError = (type === 'error');

  const events = this._events;
  if (events !== undefined)
    doError = (doError && events.error === undefined);
  else if (!doError)
    return false;

  // If there is no 'error' event listener then throw.
  if (doError) {
    let er;
    if (args.length > 0)
      er = args[0];
    if (er instanceof Error) {
      try {
        const { kExpandStackSymbol } = require('internal/util');
        const capture = {};
        Error.captureStackTrace(capture, EventEmitter.prototype.emit);
        Object.defineProperty(er, kExpandStackSymbol, {
          value: enhanceStackTrace.bind(null, er, capture),
          configurable: true
        });
      } catch (e) {}

      // Note: The comments on the `throw` lines are intentional, they show
      // up in Node's output if this results in an unhandled exception.
      throw er; // Unhandled 'error' event
    }
    // At least give some kind of context to the user
    const errors = lazyErrors();
    const err = new errors.Error('ERR_UNHANDLED_ERROR', er);
    err.context = er;
    throw err; // Unhandled 'error' event
  }
  // 通过事件名去取出 handler,这里的handler就是事件函数,也有可能是事件函数数组
  const handler = events[type];

  if (handler === undefined)
    return false;

  if (typeof handler === 'function') {
    Reflect.apply(handler, this, args);
  } else {
    const len = handler.length;
    // 对事件函数数组进行拷贝
    // 这个 arrayClone 是循环调用问题的核心解决方案
    // 在思考题的第二个例子中,当触发了第一个on的时候,已经拿到了handler的拷贝,此时第二个on并不在里面
    const listeners = arrayClone(handler, len);
    //遍历调用事件函数
    for (var i = 0; i < len; ++i)
      Reflect.apply(listeners[i], this, args);
  }

  return true;
};

// 对事件函数数组进行拷贝
function arrayClone(arr, n) {
  var copy = new Array(n);
  for (var i = 0; i < n; ++i)
    copy[i] = arr[i];
  return copy;
}
```


## 总结

on 添加绑定事件的时候，会根据是否有过绑定，如果有过绑定，就会将其改成一个Array<Function>
2. emit 的时候，会先用 arrayClone 拿到 Array<Function> 的拷贝，遍历调用，如果在 on 里面写多个监听，此时是监听不到的，因为没有被注册进去。
