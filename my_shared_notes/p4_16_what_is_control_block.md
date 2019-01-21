p4_16 中的control block

其实它和c 语言 中的 函数很像，

比如能声明变量，创建一些对象（在 P4 里就是table） 实例化外部对象。

但是 不能包含 loop， 所有的行为 都应该是DAG ：有向无环图

包含的有： match action , deparser， 以及其他形式的 包处理