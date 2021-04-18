# Dockerfile 最佳实践

Dockerfile 是一个文本文件，其中包含了用户可以在命令行上调用的已组装images的所有 「指令」。一条指令就是一层。
Union FS 最大层数限制的，比如 AUFS，曾经是最大不得超过 42 层，现在是不得超过 127 层。

## FROM

- 指定基础镜像且有效的 Dockerfile 文件必须以 `FROM` 开始。后续所有的指令操作都会指定的基础镜像上进行定制操作。

**特殊镜像：** scratch。这个镜像是虚拟的概念，实际不存在，一般用它来表示空白的镜像。如果以 scratch 为基础镜像的话，意味着不以任何镜像为基础，所写的指令将作为镜像第一层开始存在。

```shell
ARG  VERSION=latest
FROM [--platform] <image>[:tag] [AS <name>]
# 或者
FROM [--platform] <image>[@<digest>] [AS <name>]
```

- \-\-platform：在 FROM 引用多平台图像的情况下，可选标志可用于指定图像的平台。例如，`linux/amd64`， `linux/arm64`，或 `windows/amd64`。默认情况下，使用构建请求的目标平台。
- `tag` 和 `digest` 是可选的，如果忽略其中任何一个，那么构建器 latest 默认情况下都将使用标签。
- FROM 指令支持由 `ARG` 第一指令之前的任何指令声明的变量 FROM

## RUN

- 执行命令行的命令，

RUN 有 2 中表达形式：

    - RUN <command> 直接在命令行运行一样。

    ```shell
    # 如果执行长语句或者复杂语句，可以使用 \ (反斜杠) 分割，使其更具可读性，可理解性和可维护性
    RUN "/bin/bash -c 'source $HOME/.bashrc; \
    echo $HOME'"
    ```

    - RUN ["executable <可执行文件>", "param1", "param2"]

   ```shell
   # Exec 表单被解析为 JSON 数组，这意味着必须在单引号(‘)前后使用双引号(“)。
   RUN ["/bin/bash", "-c", "echo hello"]
   ```

## WORKDIR

WORKDIR 类似于 `cd`，如果 WORKDIR 不存在，那么即使后面的 Dockerfile 任何指令中都没有使用它，也将创建它。如多次使用时，只提供了相对路径，那么它将相对于上一条WORKDIR指令的路径。

使用 WORKDIR，而不要使用 `cd`，尽量使用 `绝对路径`，不要使用相对路径（易出错）。

```shell
WORKDIR /aa
WORKDIR bb

RUN pwd   
# 那么实际的输出应该是 /aa/bb
```

## COPY

2 种表达形式：

COPY 指令从以下位置复制新文件或目录 <src> ，并将它们添加到容器的文件系统中 <dest>。
<src>可以指定多个资源，但是文件和目录的路径将被解释为相对于构建上下文的源。
每个都<src>可能包含通配符，并且匹配将使用 Go 的 [filepath.Match](http://golang.org/pkg/path/filepath#Match) 规则完成

- COPY <src> <dest>

```shell
# 如果 mydir 是一个 相对路径，那么是相对于 WORKDOR ，到其中的源将在容器内进行复制。
COPY hom* /mydir/
```

## ADD

ADD是更高级的复制指令。只是在 COPY 的基础上新增了一些功能。添加远程文件/目录，建议使用 curl 或者 wget。

例如：

- 如果源文件是 URL，那么则会尝试下载该链接文件放到目标路径下，下载后的权限默认为 `600`，如果权限不满足，则需要自行设置；
如果 URL 下载的是压缩包，那么是不会自动解压的，还是需要通过 RUN 命令来进行操作。
- 如果源文件是压缩包，那么则会解压复制到目标路径下。

```shell
ADD test.tar.gz /opt
```

大部分情况下， COPY 要优于 ADD， ADD 除了有解压的功能外，

## ENV 

表达形式：

```shell
# 允许一次设置多个变量
ENV <key>=<value> ...
```

该 `ENV` 指令将环境变量 <key> 设置为 value <value>，此值将在构建阶段中所有后续指令的环境中使用。并且在许多情况下也可以内联替换。
该值将为其他环境变量解释，因此如果不对引号字符进行转义，则将其删除。像命令行解析一样，引号和反斜杠可用于在值中包含空格。

```shell
ENV MY_NAME="ascmcs Zh"
ENV MY_cat=Rex\ The\ cat
```

## CMD

> CMD 指令就是用于指定默认的容器主进程的启动命令的。
> CMD 指令在整个 dockerfile 中只能出现一条，如果出现了多次，那么只有最后一个才会生效。


CMD 有 3 种表达方式：

- CMD ["executable <可执行文件>","param1","param2"] `推荐方式`


```shell
CMD [ "sh", "-c", "echo $HOME" ]
```

- CMD ["param1","param2"] `作为 ENTRYPOINT` 的默认参数

如果要用这种方式表达，那么需要配合 [ENTRYPOINT](./Dockerfile.md#ENTRYPOINT) 一起使用。

- CMD command param1 param2  `shell 表达形式`

```shell
CMD echo $HOME

# 如果用下面的进行表达，则将不会替换变量 $HOME，如果希望能被替换，那么请使用推荐的方式进行表达。
CMD [ "echo", "$HOME" ]
```

> 需要注意的是：Docker 不是虚拟机，容器中应用没有前台或者后台执行的概念。对于容器而言，启动的程序即是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。

假设希望 nginx 这个应用程序可以以守护进程的方式运行，那么我们应该这样表达：


```shell

# 正确表达，直接执行 nginx 可执行文件，并且要求以前台形式运行
CMD ["nginx", "-g", "daemon off;"]

# 错误表达

CMD systemctl start nginx
```

### RUN 与 CMD 的区别

RUN：实际运行命令并提交结果；
CMD：生成时不执行任何操作，但指定镜像的预期命令。

## ENTRYPOINT

ENTRYPOINT 有 2 中表达方式：

> ENTRYPOINT 的目的和 CMD 一样，都是在指定容器启动程序及参数。ENTRYPOINT 在运行时也可以替代，不过比 CMD 要略显繁琐，需要通过 docker run 的参数 `--entrypoint` 来指定。
> 当指定了 ENTRYPOINT 后，CMD 的含义就发生了改变，不再是直接的运行其命令，而是将 CMD 的内容作为参数传给 ENTRYPOINT 指令，换句话说实际执行时，将变为  `<ENTRYPOINT> "<CMD>"`。

- ENTRYPOINT ["executable <可执行文件>", "param1", "param2"] `推荐方式`

使用该方式，可以设置稳定的默认命令和参数，然后通过使用两种形式的 `CMD` 来设置有可能被修改的其他默认值。

```shell
CMD ["curl", "-s", "http://myip.ipip.net"]
```

这种方式显然是没有问题的，但是在执行时，希望获取头部信息，那么则会报可执行文件找不到 `executable file not found` 的错误信息，那么使用 ENTRYPOINT 解决

```shell
ENTRYPOINT ["curl", "-s", "http://myip.ipip.net"]
```

此时，执行时传入 `-i` 是能成功获取头部信息的，这是因为存在了 ENTRYPOINT，那么 CMD 的内容将会作为参数传给 ENTRYPOINT，而 -i 就是新的 CMD，从而传给 curl，以达到预期的执行效果。

- ENTRYPOINT command param1 param2  `shell 表达方式`

```shell
ENTRYPOINT exec top -b
```


### CMD 和 ENTRYPOINT 是怎么相互作用的？

- Dockerfile 中，至少要指定 CMD 或者 ENTRYPOINT 其中之一。
- ENTRYPOINT 使用容器可执行文件时应定义。
- CMD 应该用作 ENTRYPOINT 在容器中定义命令或执行临时命令的默认参数的方式。
- CMD 当使用其他参数运行容器时，将被覆盖。

 


## 参考资料
- [dockerfile最佳实践](https://docs.docker.com/engine/reference/builder/)