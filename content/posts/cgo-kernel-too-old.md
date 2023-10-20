+++
title = "cgo 遇到 kernel too old 解决小记"
date = "2023-09-26"
author = "Jeff"
cover = "https://image.0xbb.dev/2023/09/202309261818904.png"
description = "Linux 运行有 cgo sqlite 依赖的应用时出现 FATAL: kernel too old 错误"
+++

## 背景

事情是从前几天 [loggie](https://github.com/loggie-io/loggie) 的一次阿里云 ECS 部署开始的，之前有过部署华为云的 EulerOS 没有遇到任何问题，但阿里云 ECS 是 centos6 直接
```bash
FATAL: kernel too old
Aborted
```
原因是 loggie 依赖了 [go-sqlite3](https://github.com/mattn/go-sqlite3) 需要使用 cgo 编译，然后是编译时二进制静态打包的 c lib 需要更高的内核版本，所以我们可以尝试降级编译时使用的 gcc 版本，理论上可以降低运行时要求的 kernel version

## 尝试 gcc4.8

我们可以在 centos 上测试静态构建看看，一般 centos 6 和 7 都低于 4.8 需要额外安装。
```bash
# 安装编译依赖
$ yum install https://packages.endpointdev.com/rhel/7/os/x86_64/endpoint-repo.x86_64.rpm -y && yum install -y gcc-c++ wget git make glibc-static

# 安装 gcc 4.8
$ wget http://people.centos.org/tru/devtools-2/devtools-2.repo -O /etc/yum.repos.d/devtoolset-2.repo --no-check-certificate
$ yum -y install devtoolset-2-gcc devtoolset-2-gcc-c++ devtoolset-2-binutils

# 查看 gcc 版本
$ scl enable devtoolset-2 bash
$ gcc -v
...
gcc version 4.8.2 20140120 (Red Hat 4.8.2-15) (GCC)

# 开始静态编译
$ export TAG="$(git describe --tags --exact-match 2> /dev/null || git symbolic-ref -q --short HEAD)-$(git rev-parse --short HEAD)"
$ CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -mod=vendor -a -ldflags '-X github.com/loggie-io/loggie/pkg/core/global._VERSION_=${TAG} -s -w -extldflags "-static"' -o loggie cmd/loggie/main.go
# command-line-arguments
/tmp/go-link-2902590011/000028.o: In function `unixDlOpen':
/falcon-loggie/vendor/github.com/mattn/go-sqlite3/sqlite3-binding.c:40175: warning: Using 'dlopen' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/tmp/go-link-2902590011/000033.o: In function `mygetgrouplist':
/_/os/user/getgrouplist_unix.go:15: warning: Using 'getgrouplist' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/tmp/go-link-2902590011/000032.o: In function `mygetgrgid_r':
/_/os/user/cgo_lookup_unix.go:37: warning: Using 'getgrgid_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/tmp/go-link-2902590011/000032.o: In function `mygetgrnam_r':
/_/os/user/cgo_lookup_unix.go:42: warning: Using 'getgrnam_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/tmp/go-link-2902590011/000032.o: In function `mygetpwnam_r':
/_/os/user/cgo_lookup_unix.go:32: warning: Using 'getpwnam_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/tmp/go-link-2902590011/000032.o: In function `mygetpwuid_r':
/_/os/user/cgo_lookup_unix.go:27: warning: Using 'getpwuid_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/tmp/go-link-2902590011/000004.o: In function `_cgo_2ac87069779a_C2func_getaddrinfo':
/tmp/go-build/cgo-gcc-prolog:58: warning: Using 'getaddrinfo' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/usr/lib/../lib64/libpthread.a(libpthread.o): In function `sem_open':
(.text+0x77cd): warning: the use of `mktemp' is dangerous, better use `mkstemp'

# 检查二进制
$ file ./loggie
./loggie: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.18, stripped
```

现在你可以通过 gcc 4.8 编译出来一个二进制文件并且可以在 centos 6 上跑了，算是解决了我们当前所遇到的 kernel too old 的问题。

但你会发现，这种方式存在一些缺陷

- 构建过程中有一些 warning 存在，它并不会导致构建失败，而且构建出来的二进制也是能运行的，你需要根据实际情况检查 warning，因为这些 warning 都表示不可用
- 兼容性问题，如果你想要旧的 glibc 环境运行，你就必须使用旧的 glibc 环境构建
- 不适合 Alpine

为了解决上述缺陷，我们可以尝试使用 musl

## musl 替换 glibc

还是原来的 gcc 环境，只是编译时 glibc 改为使用 musl
```bash
# 安装 musl
$ curl -LSs https://www.musl-libc.org/releases/musl-1.1.21.tar.gz -o musl.tar.gz
$ tar -xvf musl.tar.gz && cd musl-1.1.21 && ./configure
...

$ make && make install
...

# 开始静态编译
$ export TAG="$(git describe --tags --exact-match 2> /dev/null || git symbolic-ref -q --short HEAD)-$(git rev-parse --short HEAD)"
$ CC=/usr/local/musl/bin/musl-gcc CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -mod=vendor -a -ldflags '-X github.com/loggie-io/loggie/pkg/core/global._VERSION_=${TAG} -linkmode external -s -w -extldflags "-static"' -o loggie cmd/loggie/main.go

# 检查二进制
$ file ./loggie
./loggie: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
```

顺利的话 glibc 的 warning 都不会出现，运行也正常。

## 构建属性选项 --enable-kernel=2.6.32

最近看到还有一种方式是使用 extldflags 添加构建参数 --enable-kernel=2.6.32，在 docker image golang:1.17 尝试一下
```bash
# 开始静态编译
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -mod=vendor -a -ldflags '-X github.com/loggie-io/loggie/pkg/core/global._VERSION_=${TAG} -s -w -extldflags "-static --enable-kernel=2.6.32"' -o loggie cmd/loggie/main.go

# 检查二进制
$ file ./loggie
./loggie: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=9b895884eadb2cd3736917c87d8a6fe26e573a01, for GNU/Linux 3.2.0, stripped
```

看起来还想并没有效果，可能是我的姿势不对。

实际上编译出来的二进制信息还是 for GNU/Linux 3.2.0，理论上还是无法在 2.6.32 的内核上跑(本人太懒，不想再搭一套 2.6.32 的内核环境测试了)

## 总结

在 glibc 的静态编译下可能会出现一些 warning 信息，并且这些 warning 表示该功能不可用；在我的场景下我觉得可以使用 musl 替代，但 musl 并不是对任何情况都好使，你需要判断 musl 是否满足你需要的 feature，而且 musl 对比 glibc 性能有所下降，还有不少的限制存在没有仔细了解。
