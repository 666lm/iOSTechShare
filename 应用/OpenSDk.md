OpenSDK
---

# 资料

## OpenSDK的第三方 Dependency Collisions

解决方案：首先在pod里面通过nm找出对应repo的特殊symbol，然后宏替换对应的symbol，最后工程里面用同样的宏替换。

开发过程不用替换；发布版本的时候，注意替换即可。

实施过程中，因为Cocoapods版本更新导致pch文件变更，可将宏转换文件加入到pod中对应repo的pch中。

参考：（里面也有rename等其他方案的比较）

* [Prefix Static Library iOS](http://stackoverflow.com/questions/11512291/prefix-static-library-ios/19341366#19341366)
* [generate namespace header](https://github.com/jverkoey/nimbus/blob/master/scripts/generate_namespace_header)
* [Avoiding Dependency Collisions in an iOS Library](http://pdx.esri.com/blog/namespacing-dependencies/)
* [Avoiding Dependency Collsions in iOS Static Library Managed by CocoaPods](http://blog.sigmapoint.pl/avoiding-dependency-collisions-in-ios-static-library-managed-by-cocoapods/)

# 总结

# 思考

