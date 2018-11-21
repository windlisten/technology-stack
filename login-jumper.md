# 基于 expect 实现跳板机一键登录
> **expect**是一个自动交互功能的工具。**expect**原理是开了一个子进程，通过spawn来执行shell脚本，监测到脚本的返回结果，通过expect判断要进行的交互输入内容（send）。

## 安装expect 
- mac brew insatll expect
- liunx 先安装tcl: apt-get install tcl     
再安装expect: apt-get install expect

## 编辑登录跳板机脚本 login_jumper.sh
```shell
#!/usr/bin/expect

# 配置信息
set jumper_user             用户名
set jumper_server_host      跳板机地址 
set jumper_server_port      跳板机端口
set jumper_key_path         私钥文件地址
set jumper_password         私钥密码

# 接收脚本传入参数：登录目标服务器地址（可以是IP地址，也可以是：t、r、x、p...）
set to_env [lindex $argv 0 ]
switch $to_env {
    "t" { set ip "*.*.*.*"  }
    "r" { set ip "*.*.*.*"  }
    "x" { set ip "*.*.*.*"  }
    "p" { set ip "*.*.*.*" }
    "p1" { set ip "*.*.*.*" }
    "p2" { set ip "*.*.*.*" }
    default { set ip $to_env }
}

# 开启子进程ssh 登录跳板机
spawn ssh -i $jumper_key_path $jumper_user@$jumper_server_host -p$jumper_server_port

# 捕获返回信息，自动输入密码
expect {
    "Enter passphrase for key '/xxx/xxxx.pem':" {
        send "$jumper_password\r"
     }
}

# 自动跳转到目标服务器
expect {
    "Opt or ID>:" {
        send "$ip\r"
    }
}

# 把控制权交给控制台
interact
```
## 将shell脚本添加可执行权限，并起别名
```
$ chmod 700 login_jumper.sh 
$ vim ~/.bash_aliases
alias jump='/your_path/login-jump.sh'
```
使别名生效
```
$ source ~/.bash_aliases
```

## 验收,直接输入 jump t 命令即可登录t环境
```
$ jumpe t
```



