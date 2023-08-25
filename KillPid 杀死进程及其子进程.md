# KillPid 杀死进程及其子进程

# killPid.sh

```shell
function killPid(){
    pids=$(ps -ef|awk '{if($3=='$1'){print $2} }');
        kill -9 $1  &>/dev/null
    if [ -n "$pids" ]; then
        for pid in $pids
        do
        killPid $pid
        done
    fi
}

killPid  $Pid
```



