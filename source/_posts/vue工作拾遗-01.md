---
title: vue工作拾遗-01
date: 2022-9-03 20:21:16
tags: [vue2]
categories: [vue2]
---
记录下工作中遇到的vue有关有意思的知识点，可能会有02 也可能没有。太懒了。

# vue 和 html页面挂载
这个主要是在项目中使用的两种方法。刚好负责的港股业务中使用到了php挂载vue节点。
php页面上使用div+id ， 然后正常写vue页面。在php页面上 请求vue页面的模块，然后挂载。
缺点会有闪动，但是防止了整个项目的重构，目前看来可接受。等待后续所有的都迁移成vue。

在这过程中尝试了 vue嵌入html
具体插件代码
```js
(function () {
  var vueAppend = {}
  var async = true;

  var fireEvent = function (element, event) {
      var evt = document.createEvent('HTMLEvents');
      evt.initEvent(event, true, true);
      return !element.dispatchEvent(evt);
  };


  var slice = [].slice,
    singleTagRE = /^<(\w+)\s*\/?>(?:<\/\1>|)$/,
    tagExpanderRE = /<(?!area|br|col|embed|hr|img|input|link|meta|param)(([\w:]+)[^>]*)\/>/ig,
    table = document.createElement('table'),
    fragmentRE = /^\s*<(\w+|!)[^>]*>/,
    tableRow = document.createElement('tr'),
    containers = {
      'tr': document.createElement('tbody'),
      'tbody': table,
      'thead': table,
      'tfoot': table,
      'td': tableRow,
      'th': tableRow,
      '*': document.createElement('div')
    };

  var fragment = function (html, name, properties) {
    var dom, container
    // A special case optimization for a single tag
    if (singleTagRE.test(html)) dom = document.createElement(RegExp.$1)

    if (!dom) {
      if (html.replace) html = html.replace(tagExpanderRE, "<$1></$2>")
      if (name === undefined) name = fragmentRE.test(html) && RegExp.$1
      if (!(name in containers)) name = '*'

      container = containers[name]
      container.innerHTML = '' + html
      dom = slice.call(container.childNodes).map(function (child) {
        return container.removeChild(child)
      })
    }

    return dom
  }

  function traverseNode(node, fun) {
    fun(node)
    for (var key in node.childNodes) {
      traverseNode(node.childNodes[key], fun)
    }
  }

  var append = function (nodes, target, cb) {
    var pendingIndex = 0;
    var doneIndex = 0;
    nodes.forEach(function (_node) {
      var node = _node.cloneNode(true)
      if (document.documentElement !== target && document.documentElement.contains(target)) {
        traverseNode(target.insertBefore(node, null), function (el) {
          if (el.nodeName != null && el.nodeName.toUpperCase() === 'SCRIPT' && (!el.type || el.type === 'text/javascript')) {
            pendingIndex++;
            if (el.src) {
              var http = new XMLHttpRequest();
              http.open('GET', el.src, async);
              http.onreadystatechange = function () {
                if (http.readyState === 4) {
                  // Makes sure the document is ready to parse.
                  if (http.status === 200) {
                    el.innerHTML = http.responseText;
                    var target = el.ownerDocument ?
                      el.ownerDocument.defaultView :
                      window;
                    target['eval'].call(target, el.innerHTML);
                    doneIndex++;
                    if (doneIndex === pendingIndex) {
                      cb();
                    }
                  }
                }
              };
              http.send(null);
            } else {
              var target = el.ownerDocument ? el.ownerDocument.defaultView : window
              target['eval'].call(target, el.innerHTML);
              doneIndex++;
              if (doneIndex === pendingIndex) {
                cb();
              }
            }
          }
        })
      }
    })
  }

  var exec = function (el, val) {
    if (val) {
      try {
        el.innerHTML = '';
        append(fragment(val), el, function cb() {
          fireEvent(el, 'appended');
        })
      } catch (e) {
        fireEvent(el, 'appenderr');
        console.error(e);
      }
    }
  }

  // exposed global options
  vueAppend.config = {};

  vueAppend.install = function (Vue) {
    Vue.directive('append', {
      bind: function (el, binding) {
        if (binding.modifiers && binding.modifiers.sync) {
          async = false;
        }
      },
      inserted: function (el, data) {
        exec(el, data.value);
      },
      componentUpdated: function (el, data) {
        if (data.value !== data.oldValue) {
          exec(el, data.value);
        }
      }
    })
  }

  if (typeof exports == "object") {
    module.exports = vueAppend;
  } else if (typeof define == "function" && define.amd) {
    define([], function () {
      return vueAppend
    });
  } else if (window.Vue) {
    window.VueAppend = vueAppend;
    Vue.use(vueAppend);
  }

})();
```

# vue 双向绑定
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230720015908.png)

new Vue()首先执行初始化，对data执行响应化处理，这个过程发生Observe中
同时对模板执行编译，找到其中动态绑定的数据，从data中获取并初始化视图，这个过程发生在Compile中
同时定义⼀个更新函数和Watcher，将来对应数据变化时Watcher会调用更新函数
由于data的某个key在⼀个视图中可能出现多次，所以每个key都需要⼀个管家Dep来管理多个Watcher
将来data中数据⼀旦发生变化，会首先找到对应的Dep，通知所有Watcher执行更新函数
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230720015956.png)

```js
function defineReactive(obj, key, val) {  
  this.observe(val);  
  const dep = new Dep();  
  Object.defineProperty(obj, key, {  
    get() {  
      Dep.target && dep.addDep(Dep.target);// Dep.target也就是Watcher实例  
      return val;  
    },  
    set(newVal) {  
      if (newVal === val) return;  
      dep.notify(); // 通知dep执行更新方法  
    },  
  });  
} 
```