# Optional chaining 

Optional Chaining 的语法有三种使用场景：

```
   obj?.prop       // optional static property access
   obj?.[expr]     // optional dynamic property access
   func?.(...args) // optional function or method call
```

# sandbox
作为一种安全机制，严格控制其中程序所能访问的资源；

关于 <iframe> sandbox

沙箱属性为iframe中的内容启用了一组额外的限制；包含以下限制：
* 唯一源
* 阻止表单提交
* 阻止脚本执行
* 禁用API调用
* 阻止链接定位到其他浏览上下文
* 阻止内容使用插件（通过`<embed>，<object>，<applet>`或其他）
* 阻止内容导航其顶级浏览上下文
* 阻止自动触发的功能（例如自动播放视频或自动聚焦表单控件）

使用：
1.设置sandbox，包含以上所有限制
```
<iframe src="demo_iframe_sandbox.htm" sandbox></iframe>
```
2.设置为空格间隔list，其中值表示移除该限制
```
<iframe src="demo_iframe_sandbox_origin.htm" sandbox="allow-same-origin allow-scripts"></iframe>
```

移除规则约定值：
* allow-same-origin      允许同源，不是唯一源
* allow-forms
* allow-scripts
* allow-pointer-lock     允许APIs
* allow-top-navigation   允许iframe内容导航其顶级浏览上下文
* allow-popups           允许弹窗

sandbox可用来控制iframe交互，如链接跳转，访问顶级context，增加同源限制等；
https://www.w3schools.com/tags/att_iframe_sandbox.asp

# MouseMoveEvent 触发频率
事件频率由设备，平台来实现，并未统一规定；连续移动出发多个事件；最佳频率应平衡响应性和性能；
浏览器一般实现为每帧dispatch一个move事件；

https://www.w3.org/TR/uievents/#event-type-mousemove
