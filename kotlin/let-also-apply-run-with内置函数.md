## kotlin 语言 let、also、apply、run、with 内置函数

主要区别：

| 函数  	| 内部参数 	| 返回值          	|
|-------	|----------	|-----------------	|
| let   	| it       	| 最后一行/return 	|
| also  	| it       	| 对象本身        	|
| apply 	| this     	| 对象本身        	|
| run   	| this     	| 最后一行/return 	|
| with  	| this     	| 最后一行/return 	|

inline结构：
- ``let``
  ```kotlin
    fun <T, R> T.let(block:(T)->R): R = block(this)
  ```
- ``also``
  ```kotlin
  fun T.also(block:(T)->Unit): T { block(this); return this }
  ```
- ``apply``
  ```kotlin
  fun T.apply(block: T.()->Unit): T { block(); return this }
  ```
- ``run``
  ```kotlin
  fun <T, R> T.run(block: T.()->R): R = block()
  ```
- ``with``
  ```kotlin
  fun <T, R> with(receiver: T, block: T.()->R): R = receiver.block()
  ```


---
参考：https://cloud.tencent.com/developer/article/1591238

