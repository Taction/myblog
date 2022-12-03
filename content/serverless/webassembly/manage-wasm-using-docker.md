---
type: docs
title: "使用Docker工具在WasmEdge中管理WebAssembly应用程序"
linkTitle: "using docker manage wasm【译】"
date: 2021-10-26T16:19:30+08:00
weight: 200
draft: false
---

开发人员可以利用DockerHub和CRI-O等Docker工具部署、管理和运行轻量级WebAssembly应用，这些应用使用[WasmEdge](https://github.com/WasmEdge/WasmEdge)运行时。虽然 WebAssembly 应用可以由多种编程语言编写，但 Rust 是迄今为止最安全、最快的 选择。

> WasmEdge 是[由 CNCF](https://www.cncf.io/sandbox-projects/)（云原生计算基金会）托管的高级 WebAssembly 运行时，是边缘计算应用程序的执行沙箱。

虽然 WebAssembly 最初是作为浏览器应用程序的运行时而发明的，但其轻量级和高性能的沙箱设计使其成为通用应用程序容器的一个有吸引力的选择。

> If WASM+WASI existed in 2008, we wouldn't have needed to create Docker. — Solomon Hykes, co-founder of Docker

与 Docker 相比，[WebAssembly 的启动速度可以提高 100 倍](https://www.infoq.com/articles/arm-vs-x86-cloud-performance/)，内存和磁盘占用空间要小得多，并且具有更好定义的安全沙箱。然而，权衡是 WebAssembly 需要自己的语言 SDK 和编译器工具链，使其成为比 Docker 更受限制的开发人员环境。WebAssembly 越来越多地用于难以部署 Docker 或应用程序性能至关重要的边缘计算场景。

Docker 的一大优势是其丰富的工具生态系统。在 WasmEdge，我们希望为我们的开发人员带来类似 Docker 的工具。为了实现这一点，我们为 CRI-O 创建了一个名为[crunw](https://github.com/second-state/crunw)的替代运行[器](https://github.com/second-state/crunw)，以将 WebAssembly 字节码程序作为 Docker 映像文件加载和运行。

{{< youtube lr4LsOnqaLY >}}


#### 安装WebAssembly runner crunw

首先，需要安装[CRI-O ](https://cri-o.io/)，[crictl](https://github.com/kubernetes-sigs/cri-tools)，[containernetworking-插件](https://github.com/containernetworking/plugins)，和[buildah](https://github.com/containers/buildah)或[docker](https://github.com/docker/cli)两者其一。在 Ubuntu 20.04 上，您只需运行以下脚本即可。这里面涉及到一些系统库，所以您可能需要以`sudo`来运行指令。

```bash
# Install CRI-O
export OS="xUbuntu_20.04"
export VERSION="1.21"
apt update
apt install -y libseccomp2 || sudo apt update -y libseccomp2
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list

curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -

apt-get update
apt-get install criu libyajl2
apt-get install cri-o cri-o-runc cri-tools containernetworking-plugins
systemctl start crio
```

您还需要安装[WasmEdge Runtime](https://github.com/WasmEdge/WasmEdge/blob/master/docs/install.md)来运行 WebAssembly 程序。

```bash
# Instal WasmEdge

wget -q https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install.sh
bash install.sh --path="/usr/local"
```

接下来，安装`crunw`，以下命令为 Ubuntu 20.04系统的命令，以使 CRI-O 使用 WasmEdge 来运行容器内wasm程序。

```bash
# Install CRUNW

wget https://github.com/second-state/crunw/releases/download/1.0-wasmedge/crunw_1.0-wasmedge+dfsg-1_amd64.deb
dpkg -i crunw_1.0-wasmedge+dfsg-1_amd64.deb
```

现在，您需要修改两个配置文件，以便 CRI-O 找到并`crunw`用作默认运行时。在进行`/etc/crio/crio.conf`以下更改。

```bash
[crio.runtime]
default_runtime = "crunw"

# 如果pause镜像拉取有问题，你可以手动拉取然后上传到你自己的dockerhub上，然后配置为自己的源进行拉取。否则不需要添加以下内容自定义pause镜像。
[crio.image]
insecure_registries = ["docker.io"]
pause_image = "docker.io/yourownname/pause:3.5"
```

然后，在`/etc/crio/crio.conf.d/01-crio-runc.conf`文件中，添加以下内容。

```bash
[crio.runtime.runtimes.runc]
runtime_path = "/usr/lib/cri-o-runc/sbin/runc"
runtime_type = "oci"
runtime_root = "/run/runc"
# The above is the original content

# Add our crunw runtime here
[crio.runtime.runtimes.crunw]
runtime_path = "/usr/bin/crun"
runtime_type = "oci"
runtime_root = "/run/crunw"
```

最后，重新启动`cri-o`以使新的 WebAssembly 运行器生效。

```bash
sudo systemctl restart crio
```

#### 从rust项目编译wasm程序

> 如果对源码没有修改，而且只想看运行效果，那么可以跳过这一步，直接使用其提供的镜像。

你可以下载[git仓](https://github.com/second-state/wasm-learning/tree/master/cli/wasi)，友情提示如果你在国内不建议，因为仓库大概有400多M。或者可以按照以下步骤创建一个rust项目并自行编译二进制文件（但是你需要已经安装了rust）

```bash
mkdir rustwasmfiledemo && cd rustwasmfiledemo
cargo init
vim src/main.rs
```

用以下内容替换原有内容

```bash
use rand::prelude::*;
use std::fs;
use std::fs::File;
use std::io::{Write, Read};
use std::env;

fn main() {
  println!("Random number: {}", get_random_i32());
  println!("Random bytes: {:?}", get_random_bytes());
  println!("{}", echo("This is from a main function"));
  print_env();
  create_file("tmp.txt", "This is in a file");
  println!("File content is {}", read_file("tmp.txt"));
  del_file("tmp.txt");
}

pub fn get_random_i32() -> i32 {
  let x: i32 = random();
  return x;
}

pub fn get_random_bytes() -> Vec<u8> {
  let mut rng = thread_rng();
  let mut arr = [0u8; 128];
  rng.fill(&mut arr[..]);
  return arr.to_vec();
}

pub fn echo(content: &str) -> String {
  println!("Printed from wasi: {}", content);
  return content.to_string();
}

pub fn print_env() {
  println!("The env vars are as follows.");
  for (key, value) in env::vars() {
    println!("{}: {}", key, value);
  }

  println!("The args are as follows.");
  for argument in env::args() {
    println!("{}", argument);
  }
}

pub fn create_file(path: &str, content: &str) {
  let mut output = File::create(path).unwrap();
  output.write_all(content.as_bytes()).unwrap();
}

pub fn read_file(path: &str) -> String {
  let mut f = File::open(path).unwrap();
  let mut s = String::new();
  match f.read_to_string(&mut s) {
    Ok(_) => s,
    Err(e) => e.to_string(),
  }
}

pub fn del_file(path: &str) {
  fs::remove_file(path).expect("Unable to delete");
}
```

```bash
vim Cargo.toml
```

```bash
[[bin]]
name = "wasi_example_main"
path = "src/main.rs"

[dependencies]
rand = "0.7.3"
wasm-bindgen = "=0.2.61"
```

执行编译命令：

```bash
rustwasmc build --enable-aot
```

wasm文件会被编译在pkg目录下。

```bash
cd pkg/
vim Dockerfile

# 添加以下内容
FROM scratch
ADD wasi_example_main.wasm .
CMD ["wasi_example_main.wasm"]
```

创建image并推送到dockerhub仓库

```bash
# 你需要修改hydai为自己的dockerhub用户名
chmod 777 wasi_example_main.wasm
sudo docker build -f Dockerfile -t hydai/wasm-wasi-example:latest .
sudo docker push hydai/wasm-wasi-example:latest
```

接下来你可以 Docker 工具（例如 ）`crictl`将发布的 wasm 镜像拉下来。下面是我们发布的 wasm 文件图像的示例。

```bash
sudo crictl pull docker.io/hydai/wasm-wasi-example
```

#### 使用 CRI-O 启动 Wasm 应用程序

要启动并运行 wasm 文件，您需要为 CRI-O 创建两个配置文件。创建`container_wasi.json`文件并添加以下内容。配置 CRI-O 运行时从 Docker 存储库中提取 wasm 文件映像的位置。

```json
{
  "metadata": {
    "name": "podsandbox1-wasm-wasi"
  },
  "image": {
    "image": "hydai/wasm-wasi-example:latest"
  },
  "args": [
    "/wasi_example_main.wasm", "50000000"
  ],
  "working_dir": "/",
  "envs": [],
  "labels": {
    "tier": "backend"
  },
  "annotations": {
    "pod": "podsandbox1"
  },
  "log_path": "",
  "stdin": false,
  "stdin_once": false,
  "tty": false,
  "linux": {
    "resources": {
      "memory_limit_in_bytes": 209715200,
      "cpu_period": 10000,
      "cpu_quota": 20000,
      "cpu_shares": 512,
      "oom_score_adj": 30,
      "cpuset_cpus": "0",
      "cpuset_mems": "0"
    },
    "security_context": {
      "namespace_options": {
        "pid": 1
      },
      "readonly_rootfs": false,
      "capabilities": {
        "add_capabilities": [
          "sys_admin"
        ]
      }
    }
  }
}
```

然后创建`sandbox_config.json`文件并添加以下内容。它定义了运行 wasm 应用程序的沙箱环境。

```json
{
  "metadata": {
    "name": "podsandbox12",
    "uid": "redhat-test-crio",
    "namespace": "redhat.test.crio",
    "attempt": 1
  },
  "hostname": "crictl_host",
  "log_directory": "",
  "dns_config": {
    "searches": [
      "8.8.8.8"
    ]
  },
  "port_mappings": [],
  "resources": {
    "cpu": {
      "limits": 3,
      "requests": 2
    },
    "memory": {
      "limits": 50000000,
      "requests": 2000000
    }
  },
  "labels": {
    "group": "test"
  },
  "annotations": {
    "owner": "hmeng",
    "security.alpha.kubernetes.io/seccomp/pod": "unconfined"
  },
  "linux": {
    "cgroup_parent": "pod_123-456.slice",
    "security_context": {
      "namespace_options": {
        "network": 0,
        "pid": 1,
        "ipc": 0
      },
      "selinux_options": {
        "user": "system_u",
        "role": "system_r",
        "type": "svirt_lxc_net_t",
        "level": "s0:c4,c5"
      }
    }
  }
}
```

通过以下例子创建一个CRI-O pod

```bash
# Create the POD. Output will be different from example.
sudo crictl runp sandbox_config.json
7992e75df00cc1cf4bff8bff660718139e3ad973c7180baceb9c84d074b516a4

# Set a helper variable for later use.
export POD_ID=7992e75df00cc1cf4bff8bff660718139e3ad973c7180baceb9c84d074b516a4
```

由于我通过minikube启动的k8s，没有启动dockershim，所以在执行上述命令时报以下错误，所以我将其替换成了containerd。

```bash
FATA[0030] run pod sandbox: rpc error: code = Unknown desc = failed to get sandbox image "k8s.gcr.io/pause:3.2": failed to pull image "k8s.gcr.io/pause:3.2": failed to pull and unpack image "k8s.gcr.io/pause:3.2": failed to resolve reference "k8s.gcr.io/pause:3.2": failed to do request: Head "https://k8s.gcr.io/v2/pause/manifests/3.2": dial tcp 74.125.23.82:443: i/o timeout
# 或者
FATA[0002] connect: connect endpoint 'unix:///var/run/dockershim.sock', make sure you are running as root and the endpoint has been started: context deadline exceeded
```

那么我们查一下[crictl文档](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md)可以看出来我们可以采用以下方式：

```bash
$ vim /etc/crictl.yaml
runtime-endpoint: unix:///var/run/dockershim.sock
image-endpoint: unix:///var/run/dockershim.sock
```

由于国内拉取k8s pause镜像有问题，因此将`unix:///var/run/dockershim.sock`改为`unix:///run/crio/crio.sock`。然后。重新执行上述命令。

切换成containerd我们要重新拉一下镜像

```bash
sudo crictl pull docker.io/hydai/wasm-wasi-example
```

从pod中创建一个容器来运行wasm字节码程序。

```bash
# Create the container instance. Output will be different from example.
sudo crictl create $POD_ID container_wasi.json sandbox_config.json
1d056e4a8a168f0c76af122d42c98510670255b16242e81f8e8bce8bd3a4476f
```

最后，启动容器并查看wasm应用程序输出

```bash
# List the container, the state should be `Created`
sudo crictl ps -a

CONTAINER           IMAGE                           CREATED              STATE               NAME                     ATTEMPT             POD ID
1d056e4a8a168       hydai/wasm-wasi-example:latest   About a minute ago   Created             podsandbox1-wasm-wasi   0                   7992e75df00cc

# Start the container
sudo crictl start 1d056e4a8a168f0c76af122d42c98510670255b16242e81f8e8bce8bd3a4476f
1d056e4a8a168f0c76af122d42c98510670255b16242e81f8e8bce8bd3a4476f

# Check the container status again.# If the container is not finishing its job, you will see the Running state# Because this example is very tiny. You may see Exited at this moment.
sudo crictl ps -a
CONTAINER           IMAGE                           CREATED              STATE               NAME                     ATTEMPT             POD ID
1d056e4a8a168       hydai/wasm-wasi-example:latest   About a minute ago   Running             podsandbox1-wasm-wasi   0                   7992e75df00cc

# When the container is finished. You can see the state becomes Exited.
sudo crictl ps -a
CONTAINER           IMAGE                           CREATED              STATE               NAME                     ATTEMPT             POD ID
1d056e4a8a168       hydai/wasm-wasi-example:latest   About a minute ago   Exited              podsandbox1-wasm-wasi   0                   7992e75df00cc

# Check the container's logs
sudo crictl logs 1d056e4a8a168f0c76af122d42c98510670255b16242e81f8e8bce8bd3a4476f

Test 1: Print Random Number
Random number: 960251471

Test 2: Print Random Bytes
Random bytes: [50, 222, 62, 128, 120, 26, 64, 42, 210, 137, 176, 90, 60, 24, 183, 56, 150, 35, 209, 211, 141, 146, 2, 61, 215, 167, 194, 1, 15, 44, 156, 27, 179, 23, 241, 138, 71, 32, 173, 159, 180, 21, 198, 197, 247, 80, 35, 75, 245, 31, 6, 246, 23, 54, 9, 192, 3, 103, 72, 186, 39, 182, 248, 80, 146, 70, 244, 28, 166, 197, 17, 42, 109, 245, 83, 35, 106, 130, 233, 143, 90, 78, 155, 29, 230, 34, 58, 49, 234, 230, 145, 119, 83, 44, 111, 57, 164, 82, 120, 183, 194, 201, 133, 106, 3, 73, 164, 155, 224, 218, 73, 31, 54, 28, 124, 2, 38, 253, 114, 222, 217, 202, 59, 138, 155, 71, 178, 113]

Test 3: Call an echo function
Printed from wasi: This is from a main function
This is from a main function

Test 4: Print Environment Variables
The env vars are as follows.
PATH: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TERM: xterm
HOSTNAME: crictl_host
PATH: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
The args are as follows.
/var/lib/containers/storage/overlay/006e7cf16e82dc7052994232c436991f429109edea14a8437e74f601b5ee1e83/merged/wasi_example_main.wasm
50000000

Test 5: Create a file `/tmp.txt` with content `This is in a file`

Test 6: Read the content from the previous file
File content is This is in a file

Test 7: Delete the previous file
```

实际运行效果:

![image-20211028152632029](/images/runwasmedgewhitdocker.png)

#### 展望下一步

在本文中，我们看到了如何使用类似 Docker 的 CRI-O 工具启动、运行和管理 WasmEdge 应用程序。

我们的下一步是使用 Kubernetes 来管理 WasmEdge 容器。为此，我们需要在 Kubernetes 中安装一个 runner 二进制文件，以便它可以同时支持常规 Docker 镜像和 wasm 字节码镜像。

#### 排错指南

在按照文档进行测试的时候遇到了一些错误，在这里记录下遇到的问题及解决方式：

##### 创建container报错权限不足

创建container的时候遇到权限问题，大致猜测可能是wasm文件的权限问题，因此在docker build image之前，`chmod 777 wasi_example_main.wasm`然后再执行docker build，重新推到仓库，拉取到本地。然后再执行创建container即可。以下是执行命令的报错：

```bash
$ crictl create $POD_ID container_wasi.json sandbox_config.json
FATA[0000] creating container: rpc error: code = Unknown desc = container create failed: open executable: Permission denied
```



[英文原文地址](https://www.secondstate.io/articles/manage-webassembly-apps-in-wasmedge-using-docker-tools/)

