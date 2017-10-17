# confd_haproxy

proxy tcp port in a dynamic way with confd and haproxy, it can be used to proxy alive tcp port based on the confd read.

### current support

confd_haproxy support tcp/ip layer's proxy, which based on `haproxy`, currently we just support mysql slave port proxy. 

## Dependency

```
perl-libwww-perl
perl-JSON
perl-Config-IniFiles
perl-DBD-MySQL
perl-Cache-Memcached
```

## How does confd_haproxy work

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

## How to use

#### consul acl

`checkmysqlslave` use 'haproxy/mysqlslave/slavexxxx' as the key name, where `xxxx` is mysql port. if consul have set [consul_acl](https://www.consul.io/api/acl.html), you must enable permission to read key 'haproxy/mysqlslave/slavexxxx', such as acl rule:
```
{
  "ID": "anonymous",
  "Name": "Anonymous Token for haproxy",
  "Type": "client",
  "Rules": "{\"key\":{\"haproxy/\":{\"policy\":\"deny\"}},\"operator\":\"read\"}"
}
```

#### start confd

2. first of all you must export consul token if your `consul` have set [consul_acl](https://www.consul.io/api/acl.html):
```
export CONSUL_HTTP_TOKEN=e95597e0-4045-11e7-a9ef-b6ba84687927
```

3. start confd:
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
confd -backend consul -node localhost:8500 -watch
```

#### run `checkmysqlslave`

you can run `checkmysqlslave` to check and update the alive mysql slave info:
```
# perl bin/checkmysqlslave --conf etc/db.conf --tag slave3301 --consul localhost:8500 --token e95597e0-4045-11e7-a9ef-b6ba84687927
[2017-10-16T16:03:33] connect to 10.0.21.5, 3301, monitor, xxxxxxxx ...
[2017-10-16T16:03:34] slave3301 in consul is ok
[2017-10-16T16:03:35] connect to 10.0.21.5, 3301, monitor, xxxxxxxx ...
[2017-10-16T16:03:35] slave3301 in consul is ok
....
....
```

#### run `checkmemcached`

run `checkmemcached` to check and update the alive memcached info:
```
perl checkmemcached --server 10.0.21.5:11211,10.0.21.7:11211 --tag mem11211 --consul localhost:8500 --token e95597e0-4045-11e7-a9ef-b6ba84687927
[2017-10-17T17:03:21] set new alive 10.0.21.7:11211 with tag mem11211 ok
[2017-10-17T17:03:22] current 10.0.21.7:11211 is ok
....
....
```


## toolkit

the `bin` dir include some useful script to check the backend servers which behind at the haproxy:
```
checkmysqlslave - check mysql whether is slave or not, set new slave info to consul when current slave is abnormal.
checkmemcached - check memcached and ensure only one alive memcached in consul when current memcached is abnormal.
...
```

## consul key format

read [quick-start](https://github.com/kelseyhightower/confd/blob/master/docs/quick-start-guide.md) for more info.

#### `checkmysqlslave`

the value in json format, and proxyport is `port + 10050` by default, all key name with a prefix name 'haproxy/mysqlslave':
```
curl -X PUT -d '{"name":"slave3301", "host":"10.0.21.5", "port":3301, "proxyport":13351}' http://localhost:8500/v1/kv/haproxy/mysqlslave/slave3301
```

#### `checkmemcached`

the value in json format, and proxyport is `port + 5000` by default, all key name with a prefix name 'haproxy/memcached':
```
curl -X PUT -d '{"name":"mem11211", "host":"10.0.21.7", "port":11211, "proxyport":16211}' http://localhost:8500/v1/kv/haproxy/memcached/mem11211
```

## TODO list:

  * mysql slave proxy (bin/checkmysqlslave)
  * redis master proxy (TODO)
  * memcached proxy (bin/checkmemcached)

