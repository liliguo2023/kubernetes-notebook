~~~~
个人学习使用，如有错误请及时指正,谢谢

假设已经安装好golang环境，并且下载好代码并切换到想要调试的tag
调试方式：
	1.kubernetes/hack/local-up-cluster.sh 启动之前要修改下列内容
		1.安装对应版本的containerd,如果cgroup使用systemd则containerd和local-up-cluster.sh都需要修改
		2.安装对应版本的etcd
		3.修改kubernetes/cluster/addons/dns/coredns/coredns.yaml.in  coredns镜像地址
		4.修改kubernetes/hack/lib/golang.sh 增加DBG=1
		5.修改kubernetes/hack/local-up-cluster.sh  增加GO_OUT 指定编译出来的相关组件位置
		6.自行下载所需的cni并替换local-up-cluster.sh中的install_cni函数的逻辑(默认是curl，如果能下载下来可以不改)
	2.kill掉对应组件
	3.使用dlv启动响应组件
	4.goland远程调试

~~~~

