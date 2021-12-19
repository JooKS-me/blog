---
title: "Shell脚本，从小白到入门"
date: Thu Feb 18 18:48:28 CST 2021
categories: ["Linux"]
tags: ["Linux"]
draft: false
---

## 格式

- 后缀名：建议使用`.sh`(建议是因为在Unix环境中是不需要拓展名的，加上后缀名一方面可以方便我们认出来，另一方面是告诉vim这是个shell脚本)

- 首行设置解析器类型：`#!/bin/bash`

- 注释：

  - 单行：`# 注释内容`

  - 多行：

    ```shell
    :<<!
    注释内容
    !
    ```

## 执行方式

- sh命令执行：`sh 文件名 `
- bash命令执行：`bash 文件名`
- 输入路径：`.\文件名`，这种需要执行的权限，会调用首行指定的解析器
- source命令：`source 文件名`，前面都会创建子进程，而这个不会，是直接在当前shell执行命令。

## 变量

环境变量：`env`命令即可查看

- 全局配置文件
  - /etc/profile，当shell环境初始化时，会加载全局配置文件/etc/profile里面的环境变量，供给所有shell程序使用
  - /etc/profile.d/*.sh
  - /etc/bashrc
- 个人配置文件
  - 当前用户目录/.bash_profile
  - 当前用户目录/.bashrc
- 环境变量加载初始化过程

![image-20210218175451569](https://image-1301164990.cos.ap-shanghai.myqcloud.com/img/20210218175451.png)

环境变量+自定义变量+函数：`set`命令即可查看

显示变量：`echo`

取消设定变量：`unset`

设置成环境变量：`export`，父进程给子进程传值

#### 各种环境变量

- HOME：代表用户家目录

- SHELL：当前环境使用哪个SHELL

- HISTSIZE：`history`命令记录的记录最大条数

- MAIL： 当我们使用`mail`指令收信时，系统会去读取的邮件信箱文件(mailbox)

- PATH：执行文件搜寻的路径，用冒号(:)分割

- LANG：语系数据

- RANDOM：随机数。目前大多数发行版都会有随机数生成器，即`/dev/random`，详细见下图

  ![image-20210218155640326](https://image-1301164990.cos.ap-shanghai.myqcloud.com/img/20210218155640.png)

#### 字符串

可以选择带单引号，双引号，或者啥都不带。

单引号不会显示出$变量，啥都不带不能带空格。

```shell
#左侧开始截取
${变量名:start:length} #第一个索引为0
${变量名:start}

#右侧开始截取
${变量名:0-start:length} #第一个索引为1，0后面是杠
${变量名:0-start}

#从左侧开始查找字符串，找到后，截取这个字符串后面的所有字符串
${变量名#*chars} #找第一个
${变量名##*chars} #找最后一个
#从右侧开始查找
${变量名%chars*}
${变量名%%chars*}
```

#### 自定义常量

`readonly 变量名(=value)`

`readonly -p`查看当前shell的所有常量

#### 特殊变量

- `${n}`

  当输入`sh 脚本文件名 参数1 参数2 ...`时，`$n`，获取第n个输入参数(从1开始，0为脚本文件名)

  n为二位及以上数字时需要改为`${n}`

  脚本中，echo语句输出参数时，在引号里面和外面都行

- `$#`或`${#}`

  获取所有输入参数的个数

- `$*`与`$@`，不加引号两个都一样，加了引号`$*`把参数作为一个字符串整体(单字符串)返回，`$@`把每个参数作为一个字符串返回

- `$?`返回上一条语句的执行状态，0为正常，非0表示出错了

- `$$`获取当前shell的pid，获取所有pid可以用`ps -aux`

#### 索引数组

- 定义

  ```shell
  array_name=(item1 item2 ...) #等号旁边不能有空格
  array_name=([index1]=item1 [index2]=item2)
  ```

- 获取数组的数据

  ```shell
  ${array_name[索引下标]}
  ${array_name[*]} #所有数据
  ${array_name[@]} #所有数据
  ```

- 拼接

  ```shell
  array_newname=(${array_name1[*]} ${array_name2[*]})
  ```

- 数组的删除

  ```shell
  unset array_name[索引]
  unset array_name #删掉整个数组
  ```

## 循环

```shell
for var in 参数列表
do
		# 循环体
done
```

## Shell工作环境

交互式shell与非交互式shell

```shell
交互式shell：需要用户参与互动的shell环境。
非交互式shell：只执行命令，不需要用户参与。
```

登录shell与非登录shell

```shell
登录shell环境：需要输入用户名和密码才能进入的shell环境
`sh(或bash) -l(或--login) 脚本文件名`
`或者直接键入bash -l，exit退出时会显示logout，logout退出正常`
非登录shell环境：不使用用户名和密码就能进入的shell环境
`sh(或bash) 脚本文件名`
`或者直接键入bash，exit退出时会显示exit，logout退出报错`
```

网上有个**错误**的说法，说是终端输入`echo $0`，如果是-bash表示登录环境，如果不是bash表示非登录环境。

其实并不是这样的！我在/etc/profile中新定义一个环境变量，`bash -l`可以读出这个变量，说明`bash -l`进入的的确是登录shell环境。但在`bash -l`中输入`echo $0`却显示bash，这与上面矛盾了！

所以不要简单地认为`echo $0`就能判断出当前是否为登录环境，`echo $0`在终端中只是输出当前shell解释器的名字罢了！

如果真的有需要去查询当前shell是否已经登陆的话，我会输入`ps -aux|grep bash`自上而下看，这些都是shell的进程，如果只有你只开了一个窗口，那就好说了；如果是多个窗口，或者是多个人先后登录，那就不好说了（可以让别人先退出去）。


---
2021/02/23未完待续。。。



