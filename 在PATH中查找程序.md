# 在PATH中查找程序

# inpath.sh 

```shell
 
#!/bin/bash

set -u

function in_path() {    
    #在给定的path环境变量中找到给定的命令，如果找到返回0；
    #如果没有找到，返回1
    #参数：  $1: 需要查找的命令；
    #       $2: 指定查询的路径

    local cmd=$1
    local ourpath=$2
    local oldIFS=$IFS        #保存原有的内部字段分隔符
    local result=1            #返回值默认为1
    IFS=$':'                #重新指定内部字段分隔符
    for directory in $ourpath
    do
        if [ -x $directory/$cmd ];then    #如果存在这样一个命令，并且可以执行，返回值改变为0
            result=0
            break        #找到，就结束循环。
        fi
    done
    IFS=$oldIFS            #还原IFS的值。
    return $result
}

function checkForCmdInpath() {    #检查命令的形式是一个单独的指令还是一个绝对路径
    local var=$1
    if [ "$var" != "" ];then      #命令不为空
        if [ "${var:0:1}" = "/" ];then  #符合，则是一个绝对路径
            if [ ! -x "$var" ];then  #符合则表示不是一个可执行的命令
                return 1
            fi
        elif ! in_path $var "$PATH";then  #符合，则表示也不是一个可执行的命令
                return 2
        fi
    fi    
}


if [ $# -gt 1 ];then  #参数的个数大于1
    echo "Usage:$0 command 参数过多,只能有一个" >&2
    exit 1
fi

if [ $# -eq 0 ];then     #没有参数
    echo "Usage:$0 command 未传入参数" >&2
    exit 1
fi


checkForCmdInpath $1
case $? in
    0)    echo "$1 found in PATH" ;;
    1)    echo "$1 not found or not executable" ;;
    2)    echo "$1 not found in PATH"   ;;
esac

exit 0


```

# 执行结果

```python
[root@server1 ~]# ./inpath.sh echo
echo found in PATH
[root@server1 ~]# ./inpath.sh ls
ls found in PATH
[root@server1 ~]# ./inpath.sh ls -a
Usage:./inpath.sh command 参数过多,只能有一个
[root@server1 ~]# ./inpath.sh 
Usage:./inpath.sh command 未传入参数
[root@server1 ~]# ./inpath.sh djskdw
djskdw not found in PATH
[root@server1 ~]# ./inpath.sh /bin/ls
/bin/ls found in PATH
[root@server1 ~]# ./inpath.sh /bin/lssed
/bin/lssed not found or not executable
[root@server1 ~]# 
```

