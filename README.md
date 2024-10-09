
目录* [glance对接ceph](https://github.com)
	+ [1\. 上传镜像](https://github.com)
	+ [2\. 对接ceph](https://github.com)
		- [2\.1 创建池](https://github.com)
		- [2\.2 创建用户](https://github.com):[西部世界官网](https://tianchuang88.com)
		- [2\.3 下发ceph文件](https://github.com)
		- [2\.4 修改globals文件](https://github.com)
		- [2\.5 更新glance配置](https://github.com)
	+ [3\. 上传镜像到ceph](https://github.com)

# glance对接ceph


**基于前面的openstack环境**


glance的镜像默认情况下是存储在本地的，如果glance节点挂掉了，那么镜像也就不存在了，所以现在我们需要将glance的镜像存储在ceph集群上，这样即使glance节点挂掉了，其他的节点启动glance然后对接一下ceph，镜像依然是存在的


## 1\. 上传镜像


这里我们的glance目前还没有对接ceph，现在上传一个镜像来看看他是不是存在本地



```


|  | [root@openstack01 ~]# source .venv/kolla/bin/activate |
| --- | --- |
|  | (kolla) [root@openstack01 ~]# source /etc/kolla/admin-openrc.sh |


```

下载一个cirros镜像



```


|  | (kolla) [root@openstack01 ~]# wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img |
| --- | --- |
|  | (kolla) [root@openstack01 kolla]# openstack image create --disk-format qcow2 --container-format bare --progress  --public --file /root/cirros-0.4.0-x86_64-disk.img test_image |
|  |  |
|  | (kolla) [root@openstack01 kolla]# openstack image list |
|  | +--------------------------------------+------------+--------+ |
|  | | ID                                   | Name       | Status | |
|  | +--------------------------------------+------------+--------+ |
|  | | c5d3998d-51a7-4732-9cd9-fb34ff2d3e94 | cirros     | active | |
|  | | 1add255d-d797-4c5a-8e74-f902ca3c45b6 | test_image | active | |
|  | +--------------------------------------+------------+--------+ |


```

这样，一个镜像就上传好了。来看看他是不是将镜像存在本地了



```


|  | (kolla) [root@openstack01 kolla]# openstack image show test_image |grep file |head -1 |
| --- | --- |
|  | | file             | /v2/images/1add255d-d797-4c5a-8e74-f902ca3c45b6/file |


```

他的回显是`/v2/images/1add255d-d797-4c5a-8e74-f902ca3c45b6/file` 这个路径，直接进入glance容器内



```


|  | (kolla) [root@openstack01 kolla]# docker exec -it glance_api /bin/bash |
| --- | --- |
|  | (glance-api)[glance@openstack01 /]$ cd /var/lib/glance/images/ |
|  | (glance-api)[glance@openstack01 /var/lib/glance/images]$ ls |
|  | 1add255d-d797-4c5a-8e74-f902ca3c45b6  c5d3998d-51a7-4732-9cd9-fb34ff2d3e94 |


```

这个目录下有2个文件，每个文件都与我们的镜像ID是对应的，说明镜像就是存在本地的，并且还是存在容器内部的


## 2\. 对接ceph


现在我们开始操作ceph，在ceph上要先创建池，然后创建用户授权，最后改glance的配置文件


### 2\.1 创建池



```


|  | [root@ceph ~]# ceph osd pool create images |
| --- | --- |
|  | pool 'images' created |
|  | [root@ceph ~]# ceph osd pool application enable images rbd |
|  | enabled application 'rbd' on pool 'images' |
|  | [root@ceph ~]# rbd pool init -p images |


```

池就创建并初始化好了，接下来创建用户


### 2\.2 创建用户



```


|  | [root@ceph ~]# ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=images' -o /etc/ceph/ceph.client.glance.keyring |
| --- | --- |


```

glance用户的密钥环就输出到了/etc/ceph下


### 2\.3 下发ceph文件


先去管理主机创建一个glance目录


管理主机也就是当时部署openstack的那台机器



```


|  | [root@openstack01 config]# cd /etc/kolla/config/ |
| --- | --- |
|  | [root@openstack01 config]# mkdir glance |


```

将ceph.conf和glance用户的密钥环发送到管理主机的/etc/kolla/config/



```


|  | [root@ceph ~]# scp /etc/ceph/ceph.conf /etc/ceph/ceph.client.glance.keyring 192.168.200.130:/etc/kolla/config/glance/ |
| --- | --- |
|  | root@192.168.200.130's password: |
|  | ceph.conf                                                           100%  181   256.9KB/s   00:00 |
|  | ceph.client.glance.keyring                                          100%   64    67.6KB/s   00:00 |
|  |  |


```

**注意一个坑**，在管理主机上打开这2个文件，将里面的缩进给删除，或者将缩进改为空格键，不然会报错，因为等会升级glance是使用ansible读取配置，yaml是不允许tab键的



```


|  | [root@openstack01 glance]# cat ceph.conf |
| --- | --- |
|  | # minimal ceph.conf for 601f8e36-2faa-11ef-9c62-000c294ff693 |
|  | [global] |
|  | fsid = 601f8e36-2faa-11ef-9c62-000c294ff693 |
|  | mon_host = [v2:192.168.200.100:3300/0,v1:192.168.200.100:6789/0] |
|  |  |
|  | [root@openstack01 glance]# cat ceph.client.glance.keyring |
|  | [client.glance] |
|  | key = AQD+d5JmAtybHBAARluqjWc6/W4xYoWPC4VHXA== |


```

最后这2个文件应该变成这个样子


### 2\.4 修改globals文件



```


|  | [root@openstack01 kolla]# vim globals.yml |
| --- | --- |
|  | ceph_glance_user: "glance" |
|  | ceph_glance_keyring: "client.{{ ceph_glance_user }}.keyring" |
|  | ceph_glance_pool_name: "images" |
|  | glance_backend_ceph: "yes" |
|  | glance_backend_file: "no" |


```

这里的keyring不要在前边加上`ceph.`因为等会执行的时候ansible会自动给带上，写上了反而报错


将globals文件改为这样就可以了，然后我们执行升级glance


### 2\.5 更新glance配置



```


|  | [root@openstack01 kolla]# source /root/.venv/kolla/bin/activate |
| --- | --- |
|  | (kolla) [root@openstack01 ~]# kolla-ansible -i multinode -t glance upgrade |


```

等待ansible的剧本执行完，执行完之后我们再来上传一个镜像


## 3\. 上传镜像到ceph


现在我们的glance配置文件就已经自动更新好了，接下来我们上传一个镜像看看是不是存在ceph集群里面了



```


|  | (kolla) [root@openstack01 ~]# openstack image create --disk-format qcow2 --container-format bare --public --file ./cirros-0.4.0-x86_64-disk.img ceph_test_image |
| --- | --- |
|  | (kolla) [root@openstack01 ~]# openstack image list |
|  | +--------------------------------------+-----------------+--------+ |
|  | | ID                                   | Name            | Status | |
|  | +--------------------------------------+-----------------+--------+ |
|  | | cfe7ca03-896d-4020-90e8-bc45e71068aa | ceph_test_image | active | |
|  | | c5d3998d-51a7-4732-9cd9-fb34ff2d3e94 | cirros          | active | |
|  | | 1add255d-d797-4c5a-8e74-f902ca3c45b6 | test_image      | active | |
|  | +--------------------------------------+-----------------+--------+ |


```

还是进入容器去查看



```


|  | (glance-api)[glance@openstack01 /var/lib/glance/images]$ ls |
| --- | --- |
|  | 1add255d-d797-4c5a-8e74-f902ca3c45b6  c5d3998d-51a7-4732-9cd9-fb34ff2d3e94 |


```

通过这里可以看见，他的回显依然是2个，也就是本地只存了2个，第3个镜像存到ceph里去了，我们回到ceph集群来查看



```


|  | [root@ceph ~]# rbd ls -p images |
| --- | --- |
|  | cfe7ca03-896d-4020-90e8-bc45e71068aa |


```

他给我们的回显正好是镜像的ID，确实已经存储到ceph集群内了


