## 单例模式作业

### 熟练掌握单例模式的常见写法。
1、饿汉

2、懒汉，同步，双重校验

3、容器

4、枚举

### 思考破坏单例模式的方式有哪些？并且归纳总结。
1、反射

2、序列化

### 梳理内部类的执行逻辑，并画出时序图。
1、首先jvm层面类的初始化只会在主动调用该类时执行，且全局只执行一次。

2、在调用getInstance方法时会主动调用LazyHolder的LAZY静态成员，这时会触发LazyHolder的初始化

3、LAZY会被初始化为声明的单例对象，此后不再需要初始化，依靠JVM保证了单例的实现。