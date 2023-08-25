# 自定义shell函数

# 1、is_in() : 相当于python中的 in

```shell
#判断一个对象是否在一个迭代对象中
function is_in {            
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
```



# 2、**join(): 相当于python中的str.join()**

```shell
function join {
     local var=$1
     local res=""
     shift
     for item in $@
     do
         res=$res"$var"$item
     done
     res=$(echo $res| cut -c2-)
     echo $res
 }
```

* 运行效果

  ```shell
  join "=" 1 2 3 4
  >> 1=2=3=4
  ```



# 3、自定义return 集获取

* shell 中的return 只能够返回数字类型的值，并且不能够大于256.所以不能够返回其他数据数据类型，比较麻烦。

* 所以自定义了一个my_return 关联数组，用于存储某个函数的返回值。

* 为了避免，取值之后，该函数的返回值依然存在，不仅占用空间，而且后续可能存在某种风险。所以要删掉这个key-value.

* 为了方便，有自定义了一个get_return 函数，用来获取某个函数的返回值。获取之后，会将这个值删除掉。只能获取一次

```shell
declare -A my_return            #用于存储函数的返回的结果
function get_return {            #用于获取返回值
         echo ${my_return[$1]}   #输出返回值
         unset my_return[$1]     #删除这个返回值
         return 0
 }
 
 
function fun {
    ........
    my_return["fun"]=value   #存储此函数函数的返回结果。key值，与函数名相同    
}

res=$(get_return fun)    #这里的fun只作为一个字符参数，不会被当成函数执行
echo $res
```



# 4、split(): 相当于python中的split

* 有两种实现方式，任选其一

```shell
function split {
        str=$1
        fs=$2
        echo $str | sed 's#'''$fs'''# #g'
}


function split {
        str=$1
        fs=$2
        echo ${str//$fs/" "}
}
```





