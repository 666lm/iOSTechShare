ViewController Router
---

# 资料

* [JLRoutes](https://github.com/joeldev/JLRoutes)
* [Routable iOS](https://github.com/clayallsopp/routable-ios)
* [HHRouter](https://github.com/Huohua/HHRouter)
* [WAAppRouting](https://github.com/Wasappli/WAAppRouting)
* [如何优雅的进行页面间的跳转](http://gaonan.me/2015/07/23/%E5%A6%82%E4%BD%95%E4%BC%98%E9%9B%85%E7%9A%84%E8%BF%9B%E8%A1%8C%E9%A1%B5%E9%9D%A2%E9%97%B4%E7%9A%84%E8%B7%B3%E8%BD%AC/)
* [URLManager](https://github.com/gaosboy/urlmanager)
* [iOS应用2.0版怎么做](http://gaosboy.com/2013/03/16/ios_vesion_2_0.html) 里面有介绍URL Scheme的原理

# 总结

ViewController Router常用于解耦、服务器来控制跳转(推送跳转、webview跳转)等。

1. 字典直接映射URL和Controller，并 做某些处理 来跳转（如返回controller，或者传入nav等），如 Routable iOS、 HHRouter、URLManager。
2. 做个 中介模式, 如 "如何优雅的进行页面间的跳转"描述。
3. 实现路由, 包括 增加路由表、URL模式匹配、增加Handler等，如JLRoutes。

# 思考

* 常规的，就是字典一一映射。优点是: 简单; 缺点是: 有些需要中介者来集中处理，并且对处理web页面不是很 理想。
* 传统的url路由，能满足常见的需求，但是需要覆盖所有，并且扩展比较 差。
* 元编程路由，完全满足所有需求，针对业务任意玩，或者说换js来做也都okay。缺点是安全性，命名问题，规范，无线代码统一问题等。

处理的场景有：web，tab，业务controller。优点 随意route。