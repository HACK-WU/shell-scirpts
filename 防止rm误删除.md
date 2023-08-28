# 防止rm 误删除

```tex
1、在Linux系统中，各个命令的优先级顺序是：
alias(别名) >> 复合命令（if,while）>> function(函数)>> build in(内建命令)>>hash>>$path>>error command
2、由此可见，如果我们想防止rm 误删除，最直接的方法就是不同要使用系统原来默认的删除命令 rm.
rm 命令一般存在于build in或者$path中
所以解决方案有一下两种：
    一、 编写专门属于自己的删除函数，比如叫saferm  执行此命令时，不会将文件删除，二十放入到回收站中。
然后再使用alias 定义别名为rm 这样我们调用rm 命令时，由于alias的优先级较高，会首先调用我们自己的函数。
    二、 重写rm 命令,主要思路也是利用函数，我们编写一个同名rm函数，然后在里面使用绝对路径的方式调用rm 指令，然后我们再调用系统的rm时，可以增加一些判断或者选项，以达到误删除的功能。 其他类似的情况也可使用指令重写的方式.
3、使用重写的方式更简单，不仅可以保留原有指令的执行方式，还可以增加限制和增加参数。
```



# 1、使用别名

原理为自己定义rm的操作：将要删除的目标文件或目录移动到自定义的回收站里面。

* saferm.sh

```shell
#!/bin/bash

set -u
#define the trash can path
TRASH_DIR="$HOME/.trash"
[ -d $TRASH_DIR ] || mkdir -p $TRASH_DIR



forbid=(   #禁止删除的目录或者文件
        "/"
        "/*"
)



function is_in {            #判断一个对象是否在一个迭代对象中
    local var=$1
    shift
    for item in $@
    do
    if [ "$item" == "$var"  ];then
            return 0
    fi
    done
    return 1
}

force=no
dir=no

while getopts "fr" opt
do
        [ "$opt" == "f" ]&& force="yes" &&continue
        [ "$opt" == "r" ]&& dir="yes" &&continue
        echo "ERROR: $opt 没有这个选项"
done

shift $(($OPTIND-1))


#rm count
count=0

#ranging all the targets
for i in $*
do
     if [[ ! "$i" =~ ^/ ]];then
        i=$root_path"/$i"
     fi
     is_in $i ${forbid[*]}
     if [ "$?" -eq 0 ];then
          echo "rm $i 禁止的操作"
          continue
     fi
    #if the current target does not exist, just ignore it
    if [ ! -f $i ] && [ ! -d $i ];then
        echo '---not found:'$i
    else
        #create a time stamp to indicate when the target is removed
        STAMP=`date +%s`
        fileName=`basename $i`
        #move the target to the trash dir to finish the fake rm action

                if [[ -d $i && $dir == "no" ]];then
                        echo "$i 是一个目录"
                        continue
                fi


                if [ "$force" == "no" ];then
                        read -p "$i 确定要删除么(yes/no)？" res
                        res=$(echo $res | tr 'A-Z' 'a-z')
                        [[ "$res" != "yes" && "$res" != "y" ]] && exit 1
                fi

        mv $i $TRASH_DIR/$fileName.$STAMP.$count
        #delete count
        count=`expr $count + 1`
    fi
done
echo "trash dir: $TRASH_DIR"
exit 0
```

这个脚本实现了基本功能：把每个移除目标移动到垃圾箱中，并根据时间戳重命名。

然后别忘了加可执行权限：

```shell
chmod +x saferm.sh
```

我们把脚本放到合理的地方，方便调用：

```bash
mv saferm.sh /home/hackwu/software/
```

也可以给脚本创建别名：

```bash
alias saferm="sh /home/hackwu/software/saferm.sh"
```

使用方法：

```bash
saferm ./a.txt temp_dir c.xlsx
```

---

# 2、rm 指令重写（推荐）

```shel
#!/bin/bash

reF="[a-zA-Z0-9._-]*"   #用于匹配文件名
forbid=(                                                                #禁止删除的目录或者文件,绝对路径的正则表达式
        "^$HOME/$reF[/]?$"              #家目录下的，一级目录或者文件不可以删除
        "^[/]+$reF/$reF[/]?$"              #根下的二级目录或者文件不可以删除
        "^[/]+$reF[/]?$"                   #根目录的一级目录或者文件不可以删除
)




function is_in {            #判断一个对象是否在一个迭代对象中
    local var=$1
    shift
    for item in $@
    do
    if [[ "$var" =~ $item  ]];then
            return 0
    fi
    done
    return 1
}

function saferm {
        TRASH_DIR="$HOME/.trash"    #创建回收站
        [ -d $TRASH_DIR ] || mkdir -p $TRASH_DIR

        local force=no
        local dir=no
        local opt
        local tag=1
        local OPTIND
        while getopts "fri" opt
        do
                [ "$opt" == "f" ]&& force="yes" &&continue
                [ "$opt" == "r" ]&& dir="yes" &&continue
                [ "$opt" == "i" ]&& continue
                echo -ne "ERROR: 没有这个选项\n"
        done
        shift $(($OPTIND-1))

        count=0
        root_path=$(pwd)
        local i
        for i in $*
        do
                #if the current target does not exist, just ignore it
                if [ ! -f $i ] && [ ! -d $i ];then
                        echo '---not found:'$i
                else
                        #create a time stamp to indicate when the target is removed
                        STAMP=`date +%s`
                        fileName=`basename $i`
                        #move the target to the trash dir to finish the fake rm action

                        if [[ -d $i && $dir == "no" ]];then
                                echo "$i 是一个目录"
                                continue
                        fi


                        if [ "$force" == "no" ];then
                                read -p "$i 确定要删除么(yes/no)？" res
                                res=$(echo $res | tr 'A-Z' 'a-z')
                                [[ "$res" != "yes" && "$res" != "y" ]] && return 1
                        fi

                        mv $i $TRASH_DIR/$fileName.$STAMP.$count
                        tag=0
                        #delete count
                        count=`expr $count + 1`
                fi
        done
        [ "$tag" -eq 0 ] && echo "trash dir: $TRASH_DIR"
        return 0
}


function rm {
        local args="$*"     #接收参数
        local my_opt=p            #新增的参数
        local EXEC="/usr/bin/rm"
        if [ "$args" == "--help" ];then
                $EXEC --help
                return 0

        fi
        local permanent=no
        local root_path=$(pwd)
        local regex="(\s|^)-[a-zA-Z]*(\s|$)"  #匹配到选项的正则表达式
        local opts=$(echo "$args" | grep -Eo "$regex")  #获取选项
        echo $opts| grep "$my_opt" >/dev/null
        [ "$?" -eq 0 ] && permanent="yes"
    local target=$(echo "$args" | sed -r 's/'''$regex'''/ /g' )                 #获取到要删除的
        local i
        local new_i
        local new_target
        for i in ${target}            #过滤违规的目标
        do
        if [[ ! "$i" =~ ^/ ]];then
                [[ "$i" =~ ^\./  ]] && i=$(echo $i | sed 's/\.\///g')
                new_i=$root_path"/$i"
        else
                new_i=$i
        fi
        is_in $new_i ${forbid[*]}
        if [ "$?" -eq 0 ];then
                echo "rm $i 禁止的操作，强制删除请使用/bin/rm 指令"
                continue
        fi
                new_target="$new_target $i"
        done
        [ "$new_target" == "" ]  && return 1
        local opts=$(echo $opts | sed 's/'''$my_opt'''//g')     #删除新添加的选项
        [ "$opts" == "-" ]&& opts=""
        if [[ "$permanent" == "no"  && -n "$new_target"  ]];then
                saferm $opts $new_target                # 移动到回收站
        else
                $EXEC $opts $new_target   #正真的删除
                return $?
        fi
}



```



* 永久生效

```bash
grep "\sexit\s" .fun && echo -e"\033[0m.fun 文件中出现严重错误\033[0m"  || source .fun >> ~/.bashrc  #添加到.bashrc 文件中
#注意以上“.fun” 是脚本名称，如果你的脚本不叫.fun 就要更换为对应的名称
```

> \#注意以上“.fun” 是脚本名称，如果你的脚本不叫.fun 就要更换为对应的名称

