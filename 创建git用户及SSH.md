脚本

```shell
#创建SSH私钥
ssh-keygen -t rsa -C "1009274113@qq.com"
#将/home/用户/.ssh/id_rsa.pub中的公钥复制到Github中
#测试
ssh -T git@github.com
```

