Redis 6.x 配置文件说明
====================================================================================================

本文档是对 Redis 6 配置文件 ``redis.conf`` 的注释说明。

**Redis version**：``6.0.5``

注意：为了读取配置文件，Redis 必须以配置文件路径作为第一个参数来启动：

.. code-block :: sh

    ./redis-server /path/to/redis.conf

关于单位的注意事项：当需要设置内存大小时，可以按惯例形式指定，如 1k、5GB、4M，以此类推：

.. code-block :: sh

    1k => 1000 bytes
    1kb => 1024 bytes
    1m => 1000000 bytes
    1mb => 1024*1024 bytes
    1g => 1000000000 bytes
    1gb => 1024*1024*1024 bytes

单位不区分大小写，因此 1GB 1Gb 1gB 都是一样的。

引入外部配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Redis 可以引入一个或多个外部配置文件。

当搭建多个 Redis 实例时，如果所有 Redis 实例都有一部分通用的配置，且每个 Redis 实例又有一些个性化配置时，将通用部分定义为一个外部配置文件，然后使用 ``include`` 选项来引入会很方便。

.. code-block :: sh

    include /path/to/local.conf
    include /path/to/other.conf

注意，include 选项不会被 admin 或 Redis Sentinel 中的 CONFIG REWRITE 命令重写。

对于同一个配置选项，Redis 始终以最后一行的配置为准，因此最好将 include 放在当前配置文件的开头，避免在运行时覆盖当前配置。

当然，如果你就是要使用 include 覆盖当前配置中的同名选项，则应将 include 放在当前配置文件的最后。


加载外部扩展模块
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Redis 4.0 开始加入了自定义外部扩展模块，外部扩展模块可以实现新的 Redis 命令、数据结构等。

通过多个 ``loadmodule`` 选项配置可以让 Redis 在启动时加载多个模块，如果 Redis 无法加载指定模块，则会中止启动。

.. code-block :: sh

    loadmodule /path/to/my_module.so
    loadmodule /path/to/other_module.so


网络配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

bind：绑定网络接口
--------------------------------------------------

默认情况下，如果未指定 bind 配置选项，则 Redis 将侦听服务器上所有可用网络接口的连接。而使用 bind 选项可以使 Redis 仅侦听一个或多个指定的网络接口，依次限制 Redis 只侦听一个或多个指定的 IP 地址。

如——

.. code-block :: sh

    bind 192.168.1.100 10.0.0.1
    bind 127.0.0.1 ::1

.. warning ::

    - 如果运行 Redis 的服务器直接暴露于 Internet，则绑定到任何网络接口都是很危险的，这会将 Redis 实例暴露给 Internet 上的所有人。因此，默认情况下，会使用 ``bind 127.0.0.1`` 配置，强制 Redis 仅侦听 IPv4 回环地址 127.0.0.1。这意味着 Redis 只能接受来自同一台计算机上的客户端连接。
    
    - 如果确定要侦听所有网络接口，则删除（或注释）该配置选项即可。

protected-mode：保护模式
--------------------------------------------------

保护模式是 Redis 安全保护的一层，用以避免暴露在 Internet 的 Redis 实例被访问和利用。

当开启保护模式时，如果 **①服务器没有适用 bind 指令显式绑定到指定网络接口**；且 **②没有配置密码**。那么服务器仅接受来自 IPv4 和 IPv6 的回环地址（127.0.0.1 和 ::1） 以及 Unix Domain Socket 的客户端连接。

.. code-block :: sh

    protected-mode yes

.. warning ::

    - 保护模式默认是开启的（``yes``）。只有确定希望在没有使用 bind 绑定网络接口且未配置身份验证的情况下，其它主机的客户端仍要连接到 Redis，才应该禁用保护模式（``no``）。

port：端口号
--------------------------------------------------

接受指定端口上的连接，默认为 6379。

如果设置为 0，则 Redis 不会侦听 TCP Socket。

.. code-block :: sh

    port 6379

tcp-backlog：TCP 已建立连接队列的大小
--------------------------------------------------

该选项用于设置 TCP 连接中已完成队列的最大长度。在高并发场景下，可以增大该值以提高客户端连接速度。

注意，该选项受到 Linux 内核的限制，需要同时配置内核参数 ``net.core.somaxconn`` 和 ``net.ipv4.tcp_max_syn_backlog`` 以达到希望的效果。

.. code-block :: sh

    tcp-backlog 511

.. note ::

    - tcp-backlog 是已建立好的连接队列大小，即服务端接收到 ACK 完成三次握手后，状态为 ESTABLISHED 的连接队列（accept queue）。
    - Redis 会取该选项和 Linux 内核参数 ``net.core.somaxconn`` 二者中的较小值。
    - TCP 连接建立过程中，服务端接收到 SYN 后会将连接加入未完成队列（syn queue），当服务端最终收到 ACK 后会将连接转换到 accept queue。
    - syn queue 的大小受 Linux 内核参数 ``net.ipv4.tcp_max_syn_backlog`` 控制；accept queue 的大小受 Linux 内核参数 ``net.core.somaxconn`` 控制。
    - 如果 somaxconn 大而 tcp_max_syn_backlog 小，那么可能 syn queue 中没有足够的连接移动到 accept queue，accept queue 永远都不会满；反之如果 tcp_max_syn_backlog 大而 somaxconn 小，那么可能 syn queue 中会堆积等待建立的连接。
    - 因此无论只调 somaxconn 还是 tcp_max_syn_backlog 可能都不起作用，需要将同时调整两个参数。
    - 注：从 Linux 内核版本 4.3 开始，用 ``net.core.somaxconn`` 来同时表示 syn queue 和 accept queue 的大小，因此只需要配置这一个参数即可。

Unix socket：侦听 Unix socket
--------------------------------------------------

指定用于侦听传入连接的 Unix socket 路径。

没有默认值，因此若未指定该选项，则 Redis 不会侦听 Unux Socket。

.. code-block :: sh

    unixsocket /tmp/redis.sock
    unixsocketperm 700

timeout：连接空闲超时
--------------------------------------------------

客户端闲置 N 秒后关闭连接，设为 0 则禁用该机制。

.. code-block :: sh

    timeout 0

tcp-keepalive：TCP 连接保活
--------------------------------------------------

如果该选项配置不为 0，则 Redis 将周期性使用 SO_KEEPALIVE 向 客户端发送 TCP ACK 包，有两个用途：

    1. 对端状态探测。
    2. 保持网络连接的活动状态。

在 Linux 上，指定的值（以秒为单位）是用于发送 ACK 的时间间隔。注意，关闭连接需要两倍的时间。在其它内核上，期限取决于内核配置。

该选项的合理值为 300 秒，这是从 Redis 3.2.1 开始的新默认值。

.. code-block :: sh

    tcp-keepalive 300


TLS/SSL 配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

tls-port：TLS 侦听端口
--------------------------------------------------

默认情况下，TLS/SSL 是禁用的。要启用的话，可以使用 ``tls-port`` 选项来定义 TLS 侦听端口。

要在默认端口上启用 TLS，请使用如下配置：

.. code-block :: sh

    port 0
    tls-port 6379

tls-cert-file/tls-key-file：证书和私钥
--------------------------------------------------

配置 X.509 证书和私钥，用于对连接的客户端、主服务器、集群等进行身份验证。

证书和私钥文件应该为 PEM 格式。

.. code-block :: sh

    tls-cert-file redis.crt 
    tls-key-file redis.key

tls-dh-params-file：DH 密钥交换
--------------------------------------------------

Redis 的 TLS/SSL 通信支持  Diffie-Hellman (DH) 密钥交换算法，可以通过该选项指定 DH 参数文件来开启。

.. code-block :: sh

    tls-dh-params-file redis.dh

tls-ca-cert-file/tls-ca-cert-dir：CA 证书
--------------------------------------------------

要使 Redis 支持 TLS/SSL，除了配置 X.509 证书和私钥之外，还需要指定用作可信根的 CA 证书捆绑文件或者路径。

Redis 不会隐式使用系统全局的证书配置，因此下面的文件和路径两个选项必须至少配置一项：

.. code-block :: sh

    tls-ca-cert-file ca.crt
    tls-ca-cert-dir /etc/ssl/certs

tls-auth-clients：客户端证书认证
--------------------------------------------------

默认情况下，Redis 使用双向 TLS 认证，要求客户端（包括副本服务器）使用有效的证书进行身份验证（客户端使用由 ``tls-ca-cert-file`` 或 ``tls-ca-cert-dir`` 指定的可信根 CA 颁发的证书进行身份认证）。

可以使用此配置选项禁用身份验证。

默认为 ``yes`` 启用，设为 ``no`` 则禁用。

.. code-block :: sh

    tls-auth-clients no

tls-replication：复制连接 TLS
--------------------------------------------------

默认情况下，Redis 副本服务器不尝试与其主服务器建立 TLS 连接。

如果要让副本服务器到其主服务器的复制连接使用 TLS ，必须在副本服务器上将 ``tls-replication`` 设置为 ``yes``。

.. code-block :: sh

    tls-replication yes

tls-cluster：集群 TLS
--------------------------------------------------

默认情况下，Redis Cluster 总线使用普通 TCP 连接，要为集群总线协议和跨节点连接启用 TLS，需将 ``tls-cluster`` 设为 ``yes``。

.. code-block :: sh

    tls-cluster yes

tls-protocols：TLS 版本
--------------------------------------------------

明确指定要支持的 TLS 版本，包括 ``TLSv1``、``TLSv1.1``、``TLSv1.2``、``TLSv1.3`` （需要 OpenSSL >= 1.1.1），不区分大小写。

可以同时指定多个版本，使用空格分隔，如下仅支持 TLSv1.2 和 TLSv1.3：

.. code-block :: sh

    tls-protocols "TLSv1.2 TLSv1.3"

tls-ciphers：TLS 密码选择
--------------------------------------------------

配置允许的 TLS 加密算法。关于该字符串的语法详情，请参见 `ciphers(1ssl) 联机帮助页 <https://www.mankier.com/1/ciphers.1ssl>`_。

注意：该配置仅适用于 <= TLSv1.2。

.. code-block :: sh

    tls-ciphers DEFAULT:!MEDIUM

tls-ciphersuites：TLSv1.3 密码学套件
--------------------------------------------------

配置允许的 TLSv1.3 密码学套件。关于该字符串的语法详情（尤其是 TLSv1.3），请参见 `ciphers(1ssl) 联机帮助页 TLS v1.3 cipher suites <https://www.mankier.com/1/ciphers.1ssl#Cipher_Suite_Names-TLS_v1.3_cipher_suites>`_。

.. code-block :: sh

    tls-ciphersuites TLS_CHACHA20_POLY1305_SHA256

tls-prefer-server-ciphers：密码学套件首选项
--------------------------------------------------

选择密码学套件时，请使用服务器而不是客户端的首选项。服务器默认遵循客户端的首选项。

.. code-block :: sh

    tls-prefer-server-ciphers yes

常规配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


快照配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


副本配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~