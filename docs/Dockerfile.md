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
    # 如果指令过长，可以使用 \ (反斜杠) 分割
    RUN "RUN /bin/bash -c 'source $HOME/.bashrc; \
    echo $HOME'"
    ```
    - RUN ["executable", "param1", "param2"]


## 参考资料
- [dockerfile最佳实践](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [FROM 指令最佳参考](https://docs.docker.com/engine/reference/builder/#from)
- [RUN 指令的最佳参考](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)