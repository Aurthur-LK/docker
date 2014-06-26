page_title: Linking Containers Together
page_description: Learn how to connect Docker containers together.
page_keywords: Examples, Usage, user guide, links, linking, docker, documentation, examples, names, name, container naming, port, map, network port, network

# 把container连接起来协同工作

在[如何使用docker的章节里](/userguide/usingdocker)我们通过网路连接到了一个跑在docker里的服务.
这是一种和docker里的servie交互的方法。(译者：docker的理念是一个service一个container， 所以link
是非常重要的) 在这章里我们会介绍一些吧docker的各个服务连接起来的进阶方法。


## Network port mapping refresher

网络接口映射

在[如何使用docker的章节里](/userguide/usingdocker)， 我们创建了一个运行Flask应用的container.

    $ sudo docker run -d -P training/webapp python app.py


> **注**
> 每个container都有自己的网络和IP地址. (回忆一下在如何使用docker的章节里
> 我们如何使用 `docker inspect` 来显示container的IP地址)
> Docker 可以有各种网络配置， 详情参阅 [Docker的网络](/articles/networking/).


当我们创建一个container的时候使用 `-P` 的时候， docker内部暴露的Port自动分配到
宿主机的一个随机的，范围在49000到49900的Port上去。在创建之后使用 `docker ps` 命令
我们可以看到container内部的5000端口被绑到了宿主机的49155上。


    $ sudo docker ps nostalgic_morse
    CONTAINER ID  IMAGE                   COMMAND       CREATED        STATUS        PORTS                    NAMES
    bc533791f3f5  training/webapp:latest  python app.py 5 seconds ago  Up 2 seconds  0.0.0.0:49155->5000/tcp  nostalgic_morse

我们还可以使用 `-p` (译者: 注意大小写)吧container内部的IP绑定到一个你定义的宿主机的端口上去。

    $ sudo docker run -d -p 5000:5000 training/webapp python app.py

这个方法不怎么好， 因为一个宿主端口只能被一个container的端口绑定。
(译者：似乎没看到下面有啥比这个更好的， 似乎想说明的是port是属于IP的， 加上IP就可以多搞几个了？见下面一段)


还有一些其他的使用 `-p` 的方法。 默认的情况下 `-p` 会把container的Port绑到宿主机上的所有ip的该端口号上。
(译者：如果你的宿主机有多个IP， 每个IP的那个端口都会被绑到container的那个port上)
我们可以通过指定接口, 例如下面的例子， 端口就只绑定到了 `localhost` 上。

    $ sudo docker run -d -p 127.0.0.1:5000:5000 training/webapp python app.py

这会吧container内部的5000端口绑定到宿主机的 `127.0.0.1` 的5000端口上去.


如果只指定了网络接口而没有指定端口号， 则container的端口会绑到这个网络接口上的一个随机端口上.


    $ sudo docker run -d -p 127.0.0.1::5000 training/webapp python app.py

我们还可以使用UDP

    $ sudo docker run -d -p 127.0.0.1:5000:5000/udp training/webapp python app.py


使用 `docker port` 可以查看当前的container的端口的绑定情况， 和特殊的端口的配置。
例如， 如果我们吧container的端口绑到了宿主机的 `localhost`， 我们输入 `docker port`
的时候输出会是:


    $ docker port nostalgic_morse
    127.0.0.1:49155


> **注**
> `-p` 可以被多次使用用来吧多个container里的port绑定到宿主机上

## Docker Container Linking

映射网络端口不是吧container彼此连接起来的唯一方法。Docker的linking系统允许你吧多个
container连接起来， 让他们彼此交互信息。Docker的linking会创建一种父子级别的关系。
父container可以看到他的子container提供的信息。

Network port mappings are not the only way Docker containers can connect
to one another. Docker also has a linking system that allows you to link
multiple containers together and share connection information between
them. Docker linking will create a parent child relationship where the
parent container can see selected information about its child.

## Container naming

Docker的linking系统依赖于container的名字。我们已经注意到了每个container都会被自动的
分配一个名字， 在本教程里大家可能已经熟悉了 `nostalgic_morse`  这个名字（译者：docker
自动分配的名字都是确实存在的词， 有些名字比较有意思 例如我的一个container就叫做tender einstein
温柔的爱因斯坦， 对于大多数情况， 这些名字绝对是考验你英语单词量的机会, 比如 backstabbing nobel, stoic carson）. 我们可以自己对container命名. 命名有两个用处:

1. 好记 比如命名一个承载web服务的container为`web`
2. container 之间可以互相引用， 例如一个 `web` container 使用一个 `db` container

通过 `--name` 可以给container命名, 例如:


    $ sudo docker run -d -P --name web training/webapp python app.py

你可以看到我们启动了一个新的container, 启动的时候使用了 `--name` 命名这个container为 `web`。
我们可以通过命令 `docker ps` 查看container的名字

    $ sudo docker ps -l
    CONTAINER ID  IMAGE                  COMMAND        CREATED       STATUS       PORTS                    NAMES
    aed84ee21bde  training/webapp:latest python app.py  12 hours ago  Up 2 seconds 0.0.0.0:49154->5000/tcp  web


我们也可以用 `docker inspect` 查看container的名字.

    $ sudo docker inspect -f "{{ .Name }}" aed84ee21bde
    /web

(译者： 上面那个乖乖的语法源于golang自带的模板语言)

> **注**
> Container的名字必须是唯一的。也就是说你只能命名一个container为 `web`。
> 要重新使用一个container的名字的时候必须把之前叫这个名字的container删除， 才能
> 再使用。删除container可以使用 `docker rm`. 还有个便利的方法就是在启动container的时候
> 使用 `--rm` 标记. 这样container停止的时候就会自动被删除。
> （译者： 如果你使用过一段时间docker， 用 `docker ps -a` 你可能会发现一大堆的已经停止了的container,
> 在很多的docker教程里， 在启动container的时候都会带上 `--rm`.)


## Container Linking
Links运行container之间发现彼此并且彼此间安全的通讯。使用 `--link`可以创建一个link.
让我们创建一个运行数据库的container。

    $ sudo docker run -d --name db training/postgres

这里我们基于 `training/postgres` image创建了一个叫做 `db` 的container, 上面提供PostgreSQL
数据库服务.

现在让我们创建一个叫做 `web`的container， 并且把他和 `db` container连接到一起。

    $ sudo docker run -d -P --name web --link db:db training/webapp python app.py

这个命令会吧 `db` 和 `web` 连接到一起 `--link` 的用法：

    --link name:alias

这里 `name` 是我们要连接的container的名字， `alias` 是一link的别名. 下面我们会看到如何使用这个
别名.

下面让我们用 `docker ps` 看看被连接到一起的container们。

    $ docker ps
    CONTAINER ID  IMAGE                     COMMAND               CREATED             STATUS             PORTS                    NAMES
    349169744e49  training/postgres:latest  su postgres -c '/usr  About a minute ago  Up About a minute  5432/tcp                 db
    aed84ee21bde  training/webapp:latest    python app.py         16 hours ago        Up 2 minutes       0.0.0.0:49154->5000/tcp  db/web,web


这里我们可以看到我们创建的两个分别叫  `db` 和 `web` 的container, 注意 `web` container
在name列里还显示了另外一个名字 `db/web`。 这个名字告诉我们 `web` container 被连接到了
`db` container， 并且建立了一种父子关系。

linking到底有什么用呢？我们已经看到了link在两个container间创建了一个 父子 关系.
父container 这个例子里的 `db` 可以得到他的子container `web`上的信息. Docker是通过在
两个container建立了一个安全通道来实现的, 这样container就不用对外暴露端口了. 你可能已经注意到了
我们在启动 `db` container的时候没有使用 `-p` 或者 `-P`。 因为我们已经吧两个container通过
link连接起来了， 所以没必要通过端口暴露数据库的服务了.


Docker通过下述两种方法吧子container里的信息暴露给父container:

* 环境变量
* 更新 `/etc/host` 文件

我们先看下docker设定的环境变量。 在 `web` containre里， 让我们运行 `env` 命令
列出所有的环境变量。


    root@aed84ee21bde:/opt/webapp# env
    HOSTNAME=aed84ee21bde
    . . .
    DB_NAME=/web/db
    DB_PORT=tcp://172.17.0.5:5432
    DB_PORT_5000_TCP=tcp://172.17.0.5:5432
    DB_PORT_5000_TCP_PROTO=tcp
    DB_PORT_5000_TCP_PORT=5432
    DB_PORT_5000_TCP_ADDR=172.17.0.5
    . . .

> **注**:
> 这些环境变量只是为第一个在container里的进程设置的。 类似的守护进程(例如 `sshd`)
> 会在新建子shell的时候抹除这些变量。
>

我们可以看到Docker创建了一些列的对于我们使用db很有用的环境变量。 每一个变量都以
`DB` 开头， 这个 `DB` 就是上面的那个别名。（译者： 为啥用别名这里就清楚了, image
不用知道其他container的名字， 只用在run的时候把名字映射为自己了解的就可以了)
如果我们起的别名是 `db1` 那么环境变量的名字的前缀就是 `DB1_`. 你可以使用这些环境
变量来配置你的应用程序来连接 `db` container 里的数据库。 连接是安全的、私有的
仅限于 `web` 和 `db`  之间。


除了环境变量之外Docker会把父container的IP添加到子container的/etc/hosts里。我们
看下一下 `web` container里的hosts文件：

    root@aed84ee21bde:/opt/webapp# cat /etc/hosts
    172.17.0.7  aed84ee21bde
    . . .
    172.17.0.5  db

我们可以看到， 有两个相关的host配置项。 第一个是给 `web` container 的， 名字
就是container的ID。 第二条吧父container的别名和父container的IP绑到了一起。我们
试着通过host的名字ping一下.

    root@aed84ee21bde:/opt/webapp# apt-get install -yqq inetutils-ping
    root@aed84ee21bde:/opt/webapp# ping db
    PING db (172.17.0.5): 48 data bytes
    56 bytes from 172.17.0.5: icmp_seq=0 ttl=64 time=0.267 ms
    56 bytes from 172.17.0.5: icmp_seq=1 ttl=64 time=0.250 ms
    56 bytes from 172.17.0.5: icmp_seq=2 ttl=64 time=0.256 ms


> **注**
> 上面第一个命令是用来安装ping的， 因为我们使用的container里没有安装。

我们可以使用这个host吧web服务和数据库服务连接到一起。

**注**
> 一个父container可以连接多个子container。 例如， 我们可以把多个web服务的container连接
> 到一个数据库container上
>


# 下一步

我们已经知道了在Docker里如何把多个container联起来协同工作，下面我们会讲解一下如何
在docker中管理数据、volumne。

跳转到 [如何管理container的数据](/userguide/dockervolumes).


