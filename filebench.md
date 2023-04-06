# FileBench

​	GitHub地址：[filebench/filebench: File system and storage benchmark that uses a custom language to generate a large variety of workloads. (github.com)](https://github.com/filebench/filebench)

## 编译安装

第一步：生成自动编译脚本

```shell
cd ./filebench
# 在./filebench中运行以下指令，需要安装 automake 
libtoolize
aclocal
autoheader
automake --add-missing
autoconf
```

第二步：编译并安装

```shell
# 需要安装 bison，不然会出现 yacc 命令缺失
# 需要安装 flex，不然会出现 parser_lex.c 文件找不到的问题
./configure
make
# 一定要安装，不然使用不了filebench的用户变量(如cvar)
sudo make install
```



## 使用filebench

```shell
cd ~/bench/my-filebench-workloads
sudo echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
filebench -f mywebserver.f
```

问题一：

​	webserver.f 出现segment fault

```shell
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

问题二：

​	跑不了filebench！出现以下问题！

​	跑不了多线程！

```shell
[FEMU] Err: *********ZONE WRITE FAILED*********
[FEMU] Err: Error IO processed!
```

