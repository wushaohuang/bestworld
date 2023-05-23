1. 不懂prototype是什么
    - [轻松理解 JS 中的面向对象，顺便搞懂 prototype](https://mp.weixin.qq.com/s?__biz=MzUxODI3Mjc5MQ==&mid=2247487985&idx=1&sn=c49a32e98aafa0f44cb943221d77f0bc&chksm=f98a3389cefdba9f3567e893a171e0d04c9d942f522d85a071c5565227ed24134a37d01ce15f&scene=27)

2. 不懂$()是什么
    - [js中的$()的用法](http://outofmemory.cn/zaji/7098637.html)

3. Vue.config.productionTip = false 是什么
    - productionTip设置为 false ，可以阻止 vue 在启动时生成生产提示开发环境下，Vue 会提供很多警告来帮你对付常见的错误与陷阱。而在生产环境下，这些警告语句却没有用，反而会增加应用的体积。此外，有些警告检查还有一些小的运行时开销，这在生产环境模式下是可以避免的

4. 不懂什么是VueX
    - [什么是vuex？vuex如何使用？](https://blog.csdn.net/m0_70477767/article/details/125155540)
    - vuex分为五个大块
    - state： 统一定义公共数据（类似于data(){return {a:1, b:2，xxxxxx}}）
    - mutations ： 使用它来修改数据(类似于methods)
    - getters： 类似于computed(计算属性，对现有的状态进行计算得到新的数据-------派生 )
    - actions： 发起异步请求
    - modules： 模块拆分
5. 不知道怎么从页面中获取当前路由
    - [vue3获取当前页面路由的四种方法](https://blog.csdn.net/qq_38974163/article/details/121762708?ydreferer=aHR0cHM6Ly93d3cuYmFpZHUuY29tL2xpbms%2FdXJsPTdDWnlXODUybW1Ka3Z0TXJ4Z0V3LVNabkFhTXlta0tfMEtCdWRDcmxoUlpxbEdBNWtyRE1uT0dCSHZWQ3Y2RVRGQTJEUFJWUVVTaVJtNWE1WDRKMWl6cTdVaUNfWHFVX05kcWM0eHR2b2ZLJndkPSZlcWlkPWE2NWRhNjVkMDAwMTkyYmYwMDAwMDAwNTY0NjM0YzJj)

6. 不懂git在.gitignore添加忽略文件不起作用
    - [git在.gitignore添加忽略文件不起作用](https://blog.csdn.net/qq_42937522/article/details/107744472?ydreferer=aHR0cHM6Ly93d3cuYmFpZHUuY29tL2xpbms%2FdXJsPTNOMUdoSUlwYW9ydFpheFk5OGxCQThsNkFPeTNyaWxORTU1eDZwTm1rLTVfY01saFlEMlJvbVIzempYSTJoNUR3UVVPT0dfTDZybmZveDhWX2haOWhNMmp4YmlobWlBc2N1bFoyMThXVm11JndkPSZlcWlkPWFkZmIwZGVhMDAxMGE0MDgwMDAwMDAwNTY0NjcwOTZj)
7. 如何将前端获取的日期转换为指定格式，并计算它与今天所差的天数
   ```java
        String strDate = (String) parameterMap.get("report1StartDate");
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy/MM/dd");
        Date report1StartDate = dateFormat.parse(strDate);
        Date currentTime = new Date();
        long dayLength = (report1StartDate.getTime() - currentTime.getTime()) / 1000 / 60 / 60 / 24;
   ```

8. data: image/png； base64 用法详解（是Data URI scheme 直译过来的意思是：URI 数据处理方案）
    - [data: image/png； base64 用法详解](https://blog.csdn.net/weixin_50339217/article/details/113387229)
