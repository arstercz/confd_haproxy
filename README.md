## confd_haproxy

proxy tcp port in a dynamic way with confd and haproxy, it can be used to proxy alive tcp port based on the confd read.

### current support

confd_haproxy support tcp/ip layer's proxy, which based on `haproxy`, currently we just support mysql slave port proxy. 

### How does confd_haproxy work

We use [consul](https://github.com/hashicorp/consul) as the [confd](https://github.com/kelseyhightower/confd) backend. Currently, you can use `bin/checkmysqlslave` to check alive mysql slave, such as the following workflow: 
```
                             +---------+    <update>  +-------------+
                             |  consul |   <--------- | bin/check.. |
                             +---------+              +-------------+
                                |                              |
                                | <confd update/reload>        | <check servers> 
                                |                              |
                                |                     +---------------+
       +---------+          +---------+               |               |
       | request |   ---->  | haproxy |   --------->  | mysql slave   |
       +---------+          +---------+               |               |
                                                      +---------------+
```

`checkmysqlslave` check and update `consul` entry when current MySQL slave server is abnormal, then `confd` capture the `consul` entry change, update the haproxy.cfg based on `consul` entry, and execute `reload_cmd` in `confd/conf.d/haproxy_mysql.toml` if `check_cmd` is successed.

### How to use

#### start confd

1. first of all, you must export consul token if your `consul` have set [consul_acl](https://www.consul.io/api/acl.html):
```
export CONSUL_HTTP_TOKEN=e95597e0-4045-11e7-a9ef-b6ba84687927
```
2. start confd:
```
# confd -onetime -backend consul -node localhost:8500
2017-10-16T16:06:06+08:00 cz-test1 confd[16454]: INFO Backend set to consul
2017-10-16T16:06:06+08:00 cz-test1 confd[16454]: INFO Starting confd
2017-10-16T16:06:06+08:00 cz-test1 confd[16454]: INFO Backend nodes set to localhost:8500
2017-10-16T16:06:06+08:00 cz-test1 confd[16454]: INFO /etc/haproxy/haproxy.cfg has md5sum af246453ab8f8e3c168dd63f055e9183 should be 0c5f58a6e13d6f2b45292b88d554b7f5
2017-10-16T16:06:06+08:00 cz-test1 confd[16454]: INFO Target config /etc/haproxy/haproxy.cfg out of sync
2017-10-16T16:06:07+08:00 cz-test1 confd[16454]: INFO Target config /etc/haproxy/haproxy.cfg has been updated
```
or the `watch` option if you want to continuous check consul entries:
```
confd -onetime -backend consul -node localhost:8500 --watch
```

#### run `checkmysqlslave`

you can run `checkmysqlslave` to check and update the alive mysql slave info:
```
# perl checkmysqlslave --conf db.conf --tag slave3301 --consul localhost:8500 --token e95597e0-4045-11e7-a9ef-b6ba84687927
[2017-10-16T16:03:33] connect to 10.0.21.5, 3301, monitor, xxxxxxxx ...
[2017-10-16T16:03:34] slave3301 in consul is ok
[2017-10-16T16:03:35] connect to 10.0.21.5, 3301, monitor, xxxxxxxx ...
[2017-10-16T16:03:35] slave3301 in consul is ok
....
....
```

### toolkit

the `bin` dir include some useful script to check the backend servers which behind at the haproxy:
```
checkmysqlslave - check mysql whether is slave or not, set new slave info to consul when current slave is abnormal.
...
...
```

TODO list:

  * mysql slave proxy (bin/checkmysqlslave)
  * redis master proxy (TODO)
  * memcached proxy (TODO)

