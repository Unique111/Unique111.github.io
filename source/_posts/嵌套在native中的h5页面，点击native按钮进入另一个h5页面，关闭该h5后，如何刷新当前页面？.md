---
title: 嵌套在native中的h5页面，点击native按钮进入另一个h5页面，关闭该h5后，如何刷新当前页面？
date: 2017-10-30
categories: ['javascript']
tags:
---
### 背景
通常，嵌套在native中的 i 版页面有这样的需求：点击页面 1 右上角的按钮（native的按钮），进入了页面 2（一个表单），那么当表单填写完毕，提交请求后，在页面 1 如何监听到请求的提交是否成功，然后刷新当前页面或做其他操作。

<!-- more -->

### 原理
在页面 1 进入页面 2 时，初始化 localStorage 的值，在页面 2 操作后，修改 localStorage 的值，然后在页面 1 轮询 localStorage 里面的值，当监听到指定值变化后，停止轮询，然后就可以做页面 1 指定的操作了。

### 实现
在页面 1 中调用如下方法：
``` javascript
/**
 * A 页面打开新的 webview 页面 B，A 在自己的页面中监听 B 对某些页面的改动
 * Android: 监听信息编辑页面对 localStorage 的改变
    $(window).on('storage',function(event) {
      if (event.key === 'project-content') {
        var content = JSON.parse(event.newValue).data;
        updateForm({content: content});
      }
    });
 * iOS: 轮询 localStorage 里的 project-content 值，如果其 status 变为了 2，那么就说明数据就绪了
    这么做是因为 iOS 的 UIWebview 有 BUG，无法监听到 storage 事件……
    参考1：https://bugs.webkit.org/show_bug.cgi?id=145565
    参考2：https://zhuanlan.zhihu.com/p/22476760
 */
  Tool.openAndPollingWebview = (function() {
    // 全局共用一个 _timerId
    var _timerId = null;
    return function(options) {
      var key = options.listenTo || defaultLocalStorageKey;
      options = {
        url: options.url,
        assertion: options.assertion || function(event) {return event.type === 'close';},
        onComplete: options.onComplete || function() {}
      };
      clearTimeout(_timerId);
      localStorage.setItem(key, JSON.stringify({data: '', status: 1, type: ''}));
      var storagePoll = function() {
        var target = JSON.parse(localStorage.getItem(key));
        console.log('polling...key: ', key, ' | target: ', target);
        if (target && options.assertion(target)) {
          clearTimeout(_timerId);
          options.onComplete(target);
        } else {
          _timerId = setTimeout(function() {
            storagePoll();
          }, 400);
        }
      };
      storagePoll();
      window.location.href = options.url;
    };
  })();
```

页面 1 中的调用方式如下：
``` javascript
xnb.addButtons([{
  type: 'image',
  url: base64Image,
  callback: function() {
    var url = pageData.proSchema + '://www.meituan.com/web?url=' + location.origin + '/connect/project/create';
    tools.openAndPollingWebview({
      listenTo: '[buffer]/connect/project/create',
      url: url,
      assertion: function(event) {
        return event.status === 2;
      },
      onComplete: function(event) {
        if (event.type === 'saved') {
          window.location.reload();
        } else if (event.type === 'cancel') {
          console.log('用户可能做了修改，但是没有提交，不必刷新本页面');
        }
      }
    });
  }
}]);
```

页面 2 中点击取消按钮：
``` javascript
Tools = {
  // 发布订阅模式
  publishStorageEvent: function(event, key) {
    key = key || ('[buffer]' + location.pathname);
    localStorage.setItem(key, JSON.stringify(event));
  },
};
function cancelFormEdit() {
  // 修改 localStorage
  tools.publishStorageEvent({data: '', status: 2, type: 'cancel'});
  xnb.closeWebview();
}
```

页面 2 中点击发起请求按钮：
``` javascript
ajax({
  url: 'xxx',
  type: 'POST',
  contentType: 'application/json',
  data: JSON.stringify(data),
  success: function(res) {
    toast('项目提交成功<br/>24小时内审核完毕');
    setTimeout(function() {
      if (isCreatingNewProject) {
        window.location.href = pageData.projectDetailURL + res.data;
      } else {
        // 修改 localStorage
        tools.publishStorageEvent({data: '', status: 2, type: 'saved'});
      }
      xnb.closeWebview();
    }, 2000);
  }
});
```









