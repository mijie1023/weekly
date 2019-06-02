# Webpack
工程化 1，提升开发效率；2，工程可以复杂，最终服务于业务场景；

## 公共模块抽取：common /vendor
1.考虑变更频率：第三方依赖因为业务无关可抽取；
2.考量被引用次数：使用 minChunks 支持；
3.webpack4 支持抽取公共模块时设置 priority；可以根据业务需求设置依赖层级；

## Happypack 提升编译速度
1.利用多线程解决node cpu密集型任务性能瓶颈；
2.多线程可能导致多 entry 执行顺序不确定，对于有执行顺序要求的case不正常work；

## UglifyPlugin
1.uglify是生产环境编译最耗时动作；一般生成环境编译频率较低，因此优化不是很紧迫；
2.webpack-uglify-parallel 多核并行压缩的方式来提升压缩速度；

## devtool
开发环境 推荐： cheap-module-eval-source-map

https://techblog.toutiao.com/2017/02/28/webpack/