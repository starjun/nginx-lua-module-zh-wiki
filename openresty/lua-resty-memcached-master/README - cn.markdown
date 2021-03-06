Name
====

lua-resty-memcached - 为ngx\_lua增加的 Memcached 客户端驱动库，使用lua编写，基于cosocket API 实现

Table of Contents
=================

* [Name](#name)
* [Status](#status)
* [Description](#description)
* [Synopsis](#synopsis)
* [Methods](#methods)
    * [new](#new)
    * [connect](#connect)
    * [set](#set)
    * [set_timeout](#set_timeout)
    * [set_keepalive](#set_keepalive)
    * [get_reused_times](#get_reused_times)
    * [close](#close)
    * [add](#add)
    * [replace](#replace)
    * [append](#append)
    * [prepend](#prepend)
    * [get](#get)
    * [gets](#gets)
    * [cas](#cas)
    * [touch](#touch)
    * [flush_all](#flush_all)
    * [delete](#delete)
    * [incr](#incr)
    * [decr](#decr)
    * [stats](#stats)
    * [version](#version)
    * [quit](#quit)
    * [verbosity](#verbosity)
* [Automatic Error Logging](#automatic-error-logging)
* [Limitations](#limitations)
* [TODO](#todo)
* [Author](#author)
* [Copyright and License](#copyright-and-license)
* [See Also](#see-also)

Status
======

该库已验证，可用于生产环境。

Description
===========

该lua库是为ngx_lua模块提供的memcached客户端驱动
nginx module:

http://wiki.nginx.org/HttpLuaModule

该lua库使用ngx_lua中的cosockey API实现，确定是非阻塞操作。


注意版本最低要求 [ngx_lua 0.5.0rc29](https://github.com/chaoslawful/lua-nginx-module/tags) or [OpenResty 1.0.15.7](http://openresty.org/#Download) is required.

Synopsis
========

```lua
    #引用lua-resty-memcached库，即文件memcached.lua
    lua_package_path "/path/to/lua-resty-memcached/lib/?.lua;;";

    server {
        location /test {
            content_by_lua '
                # 引用memcached操作库
                local memcached = require "resty.memcached"
                # new一个实体对象
                local memc, err = memcached:new()
                # new出错
                if not memc then
                    ngx.say("failed to instantiate memc: ", err)
                    return
                end
                # 设置超时时间（单位毫秒）
                memc:set_timeout(1000) -- 1 sec

                # 或者使用unix 套接字连接方式
                -- or connect to a unix domain socket file listened
                -- by a memcached server:
                --     local ok, err = memc:connect("unix:/path/to/memc.sock")

                # 配置IP 端口
                local ok, err = memc:connect("127.0.0.1", 11211)
                if not ok then
                    ngx.say("failed to connect: ", err)
                    return
                end

                # 清空所有内容
                local ok, err = memc:flush_all()
                if not ok then
                    ngx.say("failed to flush all: ", err)
                    return
                end
    
                # 增加一个key是dog,值是32的数据
                local ok, err = memc:set("dog", 32)
                if not ok then
                    ngx.say("failed to set dog: ", err)
                    return
                end

                # 获取key是dog的值
                local res, flags, err = memc:get("dog")
                if err then
                    ngx.say("failed to get dog: ", err)
                    return
                end

                if not res then
                    ngx.say("dog not found")
                    return
                end

                ngx.say("dog: ", res)

                -- put it into the connection pool of size 100,
                -- with 10 seconds max idle timeout
                local ok, err = memc:set_keepalive(10000, 100)
                if not ok then
                    ngx.say("cannot set keepalive: ", err)
                    return
                end

                -- or just close the connection right away:
                -- local ok, err = memc:close()
                -- if not ok then
                --     ngx.say("failed to close: ", err)
                --     return
                -- end
            ';
        }
    }
```

[Back to TOC](#table-of-contents)

Methods
=======

`key` 在使用时请先进行uri转义下，否则key将有可能操作失败，因为默认是对key做的转义的。如`%20 转义成一个空格` 

[Back to TOC](#table-of-contents)

new
---
`syntax: memc, err = memcached:new(opts?)`

创建一个memcached对象，如果创建失败则返回`nil`和一个string类型的错误信息。

它接受`opts`这个table类型的可选参数，下面是它支持的选项：

* `key_transform`

    包含两个功能逃逸和非转义memcached键数组表，默认时，memcached的key将会被URI转码。接下来的例子：
    
```lua
    memached:new{
        key_transform = { ngx.escape_uri, ngx.unescape_uri }
    }
```

[Back to TOC](#table-of-contents)

connect
-------
`syntax: ok, err = memc:connect(host, port)`

`syntax: ok, err = memc:connect("unix:/path/to/unix.sock")`

配置连接到远程主机IP和端口或者是本地的套接字文件路径作为memcached服务器。

在连接之前，该方法会先调用空闲的连接池（若已经连接过）

[Back to TOC](#table-of-contents)

set
---
`syntax: ok, err = memc:set(key, value, exptime, flags)`

无条件插入一个向memcached的key时。如果键已存在，覆盖它

The `value` argument could also be a Lua table holding multiple Lua
这个`value`的值可以是一个或多个lua的table。它是被连接为一个整体的字符串(中间没有任何连接符)，例如：

```lua
    memc:set("dog", {"a ", {"kind of"}, " animal"})
```

功能上相当于如下:
```lua
    memc:set("dog", "a kind of animal")
```
可选参数`exptime`，默认值是`0`
可选参数`flags`，默认值也是`0`

[Back to TOC](#table-of-contents)

set_timeout
----------
`syntax: ok, err = memc:set_timeout(time)`

设置连接超时时间（单位：毫秒 1000为1秒）
Sets the timeout (in ms) protection for subsequent operations, including the `connect` method.

ok=1表示成功，err则是nil,错误时，ok就不等于1，err则是详细的错误信息。


[Back to TOC](#table-of-contents)

set_keepalive
------------
`syntax: ok, err = memc:set_keepalive(max_idle_timeout, pool_size)`

对当前memcached连接立即进入ngx_lua cosocket的连接池。用于长连接操作使用。

You can specify the max idle timeout (in ms) when the connection is in the pool and the maximal size of the pool every nginx worker process.
您可以设定它的最大空闲超时时间（毫秒）

成功时返回ok = `1`, 失败时ok=`nil`，err中是详细的错误信息。

仅当调用`close`方法时，调用此方法将立即关闭当前缓存的已连接的对象。任何在`connect()`方法后的操作，将会返回`closed`错误信息。


[Back to TOC](#table-of-contents)

get_reused_times
----------------
`syntax: times, err = memc:get_reused_times()`
该方法返回当前连接（连接成功的）的复用时间，错误情况下，它会返回time=`nil`，和一个字符串类型的错误信息。

如果当前连接不来自内置连接。那么就返回`0`，就表示当前的连接从来没有被复用过。如果连接来自当前连接池，然后返回就是一个非零的值，所以该方法可以用于判断当前连接是否来自当前连接池。


[Back to TOC](#table-of-contents)

close
-----
`syntax: ok, err = memc:close()`

关闭当前会话连接并返回状态。正常ok=`1`,错误时ok=`nil`和一个字符串类型的错误信息。


[Back to TOC](#table-of-contents)

add
---
`syntax: ok, err = memc:add(key, value, exptime, flags)`

Inserts an entry into memcached if and only if the key does not exist.
向memcached服务器非重复插入一个key,`value`值可以是一个或多个lua table。多个table会被当成一个字符串（没有任何连接符），使用如下：

```lua
    memc:add("dog", {"a ", {"kind of"}, " animal"})
```

它实际的也是这样：

```lua
    memc:add("dog", "a kind of animal")
```
可选参数`exptime`，默认值为`0`.
可选参数`flags`,默认值为`0`.
正常是ok=`1`,错误时，返回ok=`nil`，err=详细错误信息。

[Back to TOC](#table-of-contents)

replace
-------
`syntax: ok, err = memc:replace(key, value, exptime, flags)`

向memcached服务器替换一个已存在的key的值。

`value`值可以是一个或多个lua table。多个table会被当成一个字符串（没有任何连接符），使用如下：

```lua
    memc:replace("dog", {"a ", {"kind of"}, " animal"})
```

它实际的也是这样：

```lua
    memc:replace("dog", "a kind of animal")
```


可选参数`exptime`，默认值为`0`.
可选参数`flags`,默认值为`0`.
正常是ok=`1`,错误时，返回ok=`nil`，err=详细错误信息。

[Back to TOC](#table-of-contents)

append
------
`syntax: ok, err = memc:append(key, value, exptime, flags)`

向已存在的key的值进行内容的追加。

`value`值可以是一个或多个lua table。多个table会被当成一个字符串（没有任何连接符），使用如下：

```lua
    memc:append("dog", {"a ", {"kind of"}, " animal"})
```

同样也可以这样写：

```lua
    memc:append("dog", "a kind of animal")
```

可选参数`exptime`，默认值为`0`.
可选参数`flags`,默认值为`0`.
正常是ok=`1`,错误时，返回ok=`nil`，err=详细错误信息。

[Back to TOC](#table-of-contents)

prepend
-------
`syntax: ok, err = memc:prepend(key, value, exptime, flags)`

用于向已存在 key(键) 的 value(数据值) 前面追加数据 。

`value`值可以是一个或多个lua table。多个table会被当成一个字符串（没有任何连接符），使用如下：

```lua
    memc:prepend("dog", {"a ", {"kind of"}, " animal"})
```

同样也可以这样写：

```lua
    memc:prepend("dog", "a kind of animal")
```

可选参数`exptime`，默认值为`0`.
可选参数`flags`,默认值为`0`.
正常是ok=`1`,错误时，返回ok=`nil`，err=详细错误信息。

[Back to TOC](#table-of-contents)

get
---
`syntax: value, flags, err = memc:get(key)`
`syntax: results, err = memc:get(keys)`

Get a single entry or multiple entries in the memcached server via a single key or a table of keys.

Let us first discuss the case When the key is a single string.

The key's value and associated flags value will be returned if the entry is found and no error happens.

In case of errors, `nil` values will be turned for `value` and `flags` and a 3rd (string) value will also be returned for describing the error.

If the entry is not found, then three `nil` values will be returned.

Then let us discuss the case when the a Lua table of multiple keys are provided.

In this case, a Lua table holding the key-result pairs will be always returned in case of success. Each value corresponding each key in the table is also a table holding two values, the key's value and the key's flags. If a key does not exist, then there is no responding entries in the `results` table.

In case of errors, `nil` will be returned, and the second return value will be a string describing the error.

[Back to TOC](#table-of-contents)

gets
----
`syntax: value, flags, cas_unique, err = memc:gets(key)`

`syntax: results, err = memc:gets(keys)`

Just like the `get` method, but will also return the CAS unique value associated with the entry in addition to the key's value and flags.

This method is usually used together with the `cas` method.

[Back to TOC](#table-of-contents)

cas
---
`syntax: ok, err = memc:cas(key, value, cas_unique, exptime?, flags?)`

Just like the `set` method but does a check and set operation, which means "store this data but
  only if no one else has updated since I last fetched it."

The `cas_unique` argument can be obtained from the `gets` method.

[Back to TOC](#table-of-contents)

touch
---
`syntax: ok, err = memc:touch(key, exptime)`

Update the expiration time of an existing key.

Returns `1` for success or `nil` with a string describing the error otherwise.

This method was first introduced in the `v0.11` release.

[Back to TOC](#table-of-contents)

flush_all
---------
`syntax: ok, err = memc:flush_all(time?)`

Flushes (or invalidates) all the existing entries in the memcached server immediately (by default) or after the expiration
specified by the `time` argument (in seconds).

In case of success, returns `1`. In case of errors, returns `nil` with a string describing the error.

[Back to TOC](#table-of-contents)

delete
------
`syntax: ok, err = memc:delete(key)`

Deletes the key from memcached immediately.

The key to be deleted must already exist in memcached.

In case of success, returns `1`. In case of errors, returns `nil` with a string describing the error.

[Back to TOC](#table-of-contents)

incr
----
`syntax: new_value, err = memc:incr(key, delta)`

Increments the value of the specified key by the integer value specified in the `delta` argument.

Returns the new value after incrementation in success, and `nil` with a string describing the error in case of failures.

[Back to TOC](#table-of-contents)

decr
----
`syntax: new_value, err = memc:decr(key, value)`

Decrements the value of the specified key by the integer value specified in the `delta` argument.

Returns the new value after decrementation in success, and `nil` with a string describing the error in case of failures.

[Back to TOC](#table-of-contents)

stats
-----
`syntax: lines, err = memc:stats(args?)`

Returns memcached server statistics information with an optional `args` argument.

In case of success, this method returns a lua table holding all of the lines of the output; in case of failures, it returns `nil` with a string describing the error.

If the `args` argument is omitted, general server statistics is returned. Possible `args` argument values are `items`, `sizes`, `slabs`, among others.

[Back to TOC](#table-of-contents)

version
-------
`syntax: version, err = memc:version(args?)`

Returns the server version number, like `1.2.8`.

In case of error, it returns `nil` with a string describing the error.

[Back to TOC](#table-of-contents)

quit
----
`syntax: ok, err = memc:quit()`

Tells the server to close the current memcached connection.

Returns `1` in case of success and `nil` other wise. In case of failures, another string value will also be returned to describe the error.

Generally you can just directly call the `close` method to achieve the same effect.

[Back to TOC](#table-of-contents)

verbosity
---------
`syntax: ok, err = memc:verbosity(level)`

Sets the verbosity level used by the memcached server. The `level` argument should be given integers only.

Returns `1` in case of success and `nil` other wise. In case of failures, another string value will also be returned to describe the error.

[Back to TOC](#table-of-contents)

Automatic Error Logging
=======================

By default the underlying [ngx_lua](http://wiki.nginx.org/HttpLuaModule) module
does error logging when socket errors happen. If you are already doing proper error
handling in your own Lua code, then you are recommended to disable this automatic error logging by turning off [ngx_lua](http://wiki.nginx.org/HttpLuaModule)'s [lua_socket_log_errors](http://wiki.nginx.org/HttpLuaModule#lua_socket_log_errors) directive, that is,

```nginx
    lua_socket_log_errors off;
```

[Back to TOC](#table-of-contents)

Limitations
===========

* This library cannot be used in code contexts like set_by_lua*, log_by_lua*, and
header_filter_by_lua* where the ngx_lua cosocket API is not available.
* The `resty.memcached` object instance cannot be stored in a Lua variable at the Lua module level,
because it will then be shared by all the concurrent requests handled by the same nginx
 worker process (see
http://wiki.nginx.org/HttpLuaModule#Data_Sharing_within_an_Nginx_Worker ) and
result in bad race conditions when concurrent requests are trying to use the same `resty.memcached` instance.
You should always initiate `resty.memcached` objects in function local
variables or in the `ngx.ctx` table. These places all have their own data copies for
each request.

[Back to TOC](#table-of-contents)

TODO
====

* implement the memcached pipelining API.
* implement the UDP part of the memcached ascii protocol.

[Back to TOC](#table-of-contents)

Author
======

Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>, CloudFlare Inc.

[Back to TOC](#table-of-contents)

Copyright and License
=====================

This module is licensed under the BSD license.

Copyright (C) 2012-2016, by Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>, CloudFlare Inc.

All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

[Back to TOC](#table-of-contents)

See Also
========
* the ngx_lua module: http://wiki.nginx.org/HttpLuaModule
* the memcached wired protocol specification: http://code.sixapart.com/svn/memcached/trunk/server/doc/protocol.txt
* the [lua-resty-redis](https://github.com/agentzh/lua-resty-redis) library.
* the [lua-resty-mysql](https://github.com/agentzh/lua-resty-mysql) library.

[Back to TOC](#table-of-contents)

