p4_16_what_is_switch_statement_in_p4.md


see： 11.7 switch statement

![image](https://ws2.sinaimg.cn/large/005JrW9Kly1fzhslbyig7j30pm05dq38.jpg)

## 几个特性

- 无需break

- 只能在 control block 使用

- 如果没有 `{...}` 才会fall - through

值得注意的是  `default ` 这个标签的行为， 他不代表 `default_action` 会被执行



`default_action`  在 p4 16 spec 13.1 说： 是 table 的一个 可选属性



## 觉得这个 和  p4 14 里的 action_profile  类似



![image](https://ws3.sinaimg.cn/large/005JrW9Kly1fzhskoh9znj30q00573yp.jpg)

![image](https://ws3.sinaimg.cn/large/005JrW9Kly1fzhspcmr2cj30nk0dt0tk.jpg)

### 具体16 为何去掉了 action_profile 

仅仅在 16 spec的 13.2.1.6. Additional properties 说到 p4 14 提出 action_profile 是为了大容量的表，又包含一定范围内重复entry 的复用？？


![image](https://wx2.sinaimg.cn/large/005JrW9Kly1fzht3mm01fj30yg08xmyc.jpg)