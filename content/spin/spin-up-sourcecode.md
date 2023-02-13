---
title: "Spin Up Sourcecode"
linkTitle: "Spin up 源码分析"
type: docs
date: 2023-02-11T16:27:40+08:00
draft: false
---

### 简介

通过之前文章，大概对fermyon平台架构有了了解。我们知道通过`spin up [-f path/to/spin.toml]`命令可以“启动”一个wasm程序，但是在这个过程中实际执行的代码逻辑是什么？我们的网络请求发给了谁，如何传递到wasm程序的？接下来从实际的代码执行角度来看下这个过程中的实际代码逻辑。实际上这里大概分为了两个大步骤，spin up命令对参数进行解析，根据启动时指定的是file（默认）还是bindle来进行不同的数据解析准备，最终通过spin trigger命令来进行实际动作。（从这个角度来看spin up就是为了方便命令行使用的用户友好的封装）

### 代码

对于命令的封装spin使用了clap库，这里不对这个库的使用做介绍，也就是后面代码分析的起点是命令所执行的业务函数开始，也就是每个命令对应的run函数。命令解析和参数绑定部分会被忽略。

#### spin up

spin up命令首先对参数进行解析，--file对应app有值，--bindle对应bindle有值，走不同的分支预处理：其实就是把配置读取进来，如果都定义了就报错只能定义一个。这里注意up command会记录下来所有不认识的flag，当启动trigger的时候把这些参数透传下去给trigger用。然后准备环境变量和启动参数数据，通过spin trigger命令来启动一个子进程来进行处理，并监听退出信息退出子进程。相对来说逻辑比较简单。

```rust
async fn run_inner(self) -> Result<()> {
    let working_dir_holder = match &self.tmp {
        None => WorkingDirectory::Temporary(tempfile::tempdir()?),
        Some(d) => WorkingDirectory::Given(d.to_owned()),
    };
    let working_dir = working_dir_holder.path().canonicalize()?;
    // 判断是file（app）还是bindle，对配置进行解析
    let mut app = match (&self.app, &self.bindle) {
        (app, None) => {
            let manifest_file = app
                .as_deref()
                .unwrap_or_else(|| DEFAULT_MANIFEST_FILE.as_ref());
            let bindle_connection = self.bindle_connection();
            let asset_dst = if self.direct_mounts {
                None
            } else {
                Some(&working_dir)
            };
            spin_loader::from_file(manifest_file, asset_dst, &bindle_connection).await?
        }
        (None, Some(bindle)) => match &self.server {
            //...
        },
        (Some(_), Some(_)) => bail!("Specify only one of app file or bindle ID"),
    };

    // 判断trigger是http、redis还是自定义
    let trigger_type = match &app.info.trigger {
        ApplicationTrigger::Http(_) => vec!["trigger".to_owned(), "http".to_owned()],
        ApplicationTrigger::Redis(_) => vec!["trigger".to_owned(), "redis".to_owned()],
        ApplicationTrigger::External(cfg) => vec![resolve_trigger_plugin(cfg.trigger_type())?],
    };

    // 这里就是准备启动另一个spin进行来实际处理，准备环境变量和命令参数，对于httptrigger就是 spin trigger http。
    // The docs for `current_exe` warn that this may be insecure because it could be executed
    // via hard-link. I think it should be fine as long as we aren't `setuid`ing this binary.
    let mut cmd = std::process::Command::new(std::env::current_exe().unwrap());
    cmd.args(&trigger_type).env(SPIN_WORKING_DIR, &working_dir);

    if self.help {
        cmd.arg("--help-args-only");
    } else {
        // 将配置写进SPIN_LOCKED_URL对应的文件，并设置这个环境变量
        let locked_url = self.write_locked_app(app, &working_dir)?;
        cmd.env(SPIN_LOCKED_URL, locked_url)
            .args(&self.trigger_args);
    };

    // 启动子进程
    let mut child = cmd.spawn().context("Failed to execute trigger")?;

    // 接下来就是等待退出命令好退出子进程，并等待子进程结束
}
```

先来看下一个在启动子进程的时候，除了http和redis还有一个`External`的方式，通过查看代码，这里的子命令会是 spin "trigger-{trigger_type}"，在组装命令时会检查"trigger-{trigger_type}"对应的plugin是否已经安装，如果没有安装会从built-in（内置）列表中找，如果还没有就报错。关键查找函数

```rust
fn resolve_trigger_plugin(trigger_type: &str) -> Result<String> {
    use crate::commands::plugins::PluginCompatibility;
    use spin_plugins::manager::PluginManager;

    let subcommand = format!("trigger-{trigger_type}");
    let plugin_manager = PluginManager::try_default()
        .with_context(|| format!("Failed to access plugins looking for '{subcommand}'"))?;
    let plugin_store = plugin_manager.store();
    let is_installed = plugin_store
        .installed_manifests()
        .unwrap_or_default()
        .iter()
        .any(|m| m.name() == subcommand);

    if is_installed {
        return Ok(subcommand);
    }

    if let Some(known) = plugin_store
        .catalogue_manifests()
        .unwrap_or_default()
        .iter()
        .find(|m| m.name() == subcommand)
    {
        match PluginCompatibility::for_current(known) {
            PluginCompatibility::Compatible => Err(anyhow!("No built-in trigger named '{trigger_type}', but plugin '{subcommand}' is available to install")),
            _ => Err(anyhow!("No built-in trigger named '{trigger_type}', and plugin '{subcommand}' is not compatible"))
        }
    } else {
        Err(anyhow!("No built-in trigger named '{trigger_type}', and no plugin named '{subcommand}' was found"))
    }
}
```

#### spin trigger

通过上面的分析可以看出，在设置好环境变量，创建了对应的lock配置文件情况下，完全可以通过spin trigger命令达到同样的效果。对于http和redis两种trigger来说，spin会执行不同的逻辑代码，但是整体都是一样的，启动trigger（http server或者连接redis），在有对应的信号的时候，调用wasm程序，返回处理结果。

通过查看spin的命令，可以发现，trigger命令下目前只有http和redis。接下来就以http trigger为例来介绍具体的处理逻辑。

```rust
/// The Spin CLI
#[derive(Parser)]
#[clap(
    name = "spin",
    version = version(),
)]
enum SpinApp {
    //...
    #[clap(subcommand, hide = true)]
    Trigger(TriggerCommands),
    #[clap(external_subcommand)]
    External(Vec<String>),
}
// 可以看到trigger下只有http和redis两个子命令
#[derive(Subcommand)]
enum TriggerCommands {
    Http(TriggerExecutorCommand<HttpTrigger>),
    Redis(TriggerExecutorCommand<RedisTrigger>),
}
```

接下来查看下http trigger源码，先解析传递来的两个环境变量SPIN_WORKING_DIR和SPIN_LOCKED_URL，其中SPIN_LOCKED_URL指向包含配置的文件或地址，具体文件内容实例附在了最后。先将app信息从lock文件中读取解析完毕，后面的逻辑就比较简单，解析监听地址及tls信息，启动server。

```rust
/// Create a new TriggerExecutorBuilder from this TriggerExecutorCommand.
pub async fn run(self) -> Result<()> {
    if self.help_args_only {
        Self::command()
            .disable_help_flag(true)
            .help_template("{all-args}")
            .print_long_help()?;
        return Ok(());
    }

    // Required env vars
    let working_dir = std::env::var(SPIN_WORKING_DIR).context(SPIN_WORKING_DIR)?;
    let locked_url = std::env::var(SPIN_LOCKED_URL).context(SPIN_LOCKED_URL)?;

    let loader = TriggerLoader::new(working_dir, self.allow_transient_write);

    let trigger_config =
        TriggerExecutorBuilderConfig::load_from_file(self.runtime_config_file.clone())?;

    let executor: Executor = {
        let _sloth_warning = warn_if_wasm_build_slothful();

        let mut builder = TriggerExecutorBuilder::new(loader);
        self.update_wasmtime_config(builder.wasmtime_config_mut())?;

        let logging_hooks = StdioLoggingTriggerHooks::new(self.follow_components(), self.log);
        builder.hooks(logging_hooks);
				// 在这一步会解析配置，并将components下的source下的定义的wasm从文件中读入内存，同时对wasm有很多预处理也是在这个函数中完成的，解析wasm程序到Module、校验、linker以及创建一个`InstancePre`，简单来说就是实例化一个wasm之前的步骤这里都做完了，这个工作只需要做一次，后面来请求的时候就实例化wasm就行。这里的处理涉及到一些wasmtime对wasm处理，就不进一步贴详细代码了，通过代码来看目前spin只支持wasmtime引擎。
        builder.build(locked_url, trigger_config).await?
    };

    let run_fut = executor.run(self.run_config);

    let (abortable, abort_handle) = futures::future::abortable(run_fut);
    ctrlc::set_handler(move || abort_handle.abort())?;
    match abortable.await {
        Ok(Ok(())) => {
            tracing::info!("Trigger executor shut down: exiting");
            Ok(())
        }
        Ok(Err(err)) => {
            tracing::error!("Trigger executor failed");
            Err(err)
        }
        Err(_aborted) => {
            tracing::info!("User requested shutdown: exiting");
            Ok(())
        }
    }
}
```

然后经过系列模板代码之后会转到处理函数，这个函数做的事情也很简单，根据路径去router中查找component类型，转入对应的处理：组装数据，实例化wasm，执行wasm对应的函数。

```rust
pub async fn handle(
    &self,
    mut req: Request<Body>,
    scheme: Scheme,
    addr: SocketAddr,
) -> Result<Response<Body>> {
    set_req_uri(&mut req, scheme)?;

    log::info!(
        "Processing request for application {} on URI {}",
        &self.engine.app_name,
        req.uri()
    );

    let path = req.uri().path();

    // Handle well-known spin paths
    if let Some(well_known) = path.strip_prefix(WELL_KNOWN_PREFIX) {
        return match well_known {
            "health" => Ok(Response::new(Body::from("OK"))),
            "info" => self.app_info(),
            _ => Self::not_found(),
        };
    }

    // Route to app component
    match self.router.route(path) {
        Ok(component_id) => {
            let trigger = self.component_trigger_configs.get(component_id).unwrap();

            let executor = trigger.executor.as_ref().unwrap_or(&HttpExecutorType::Spin);

            let res = match executor {
                HttpExecutorType::Spin => {
                    let executor = SpinHttpExecutor;
                    executor
                        .execute(
                            &self.engine,
                            component_id,
                            &self.base,
                            &trigger.route,
                            req,
                            addr,
                        )
                        .await
                }
                HttpExecutorType::Wagi(wagi_config) => {
                    let executor = WagiHttpExecutor {
                        wagi_config: wagi_config.clone(),
                    };
                    executor
                        .execute(
                            &self.engine,
                            component_id,
                            &self.base,
                            &trigger.route,
                            req,
                            addr,
                        )
                        .await
                }
            };
            match res {
                Ok(res) => Ok(res),
                Err(e) => {
                    log::error!("Error processing request: {:?}", e);
                    Self::internal_error(None)
                }
            }
        }
        Err(_) => Self::not_found(),
    }
}
```

`executor.execute`经过一系列封装会调用wasmtime的`instantiate_async`来实例化wasm程序，然后调用wasm的函数，这里通过execute_impl函数包装了一下。

```rust
impl HttpExecutor for SpinHttpExecutor {
    async fn execute(
        &self,
        engine: &TriggerAppEngine<HttpTrigger>,
        component_id: &str,
        base: &str,
        raw_route: &str,
        req: Request<Body>,
        _client_addr: SocketAddr,
    ) -> Result<Response<Body>> {
        tracing::trace!(
            "Executing request using the Spin executor for component {}",
            component_id
        );

        let (instance, store) = engine.prepare_instance(component_id).await?;

        let resp = Self::execute_impl(store, instance, base, raw_route, req)
            .await
            .map_err(contextualise_err)?;

        tracing::info!(
            "Request finished, sending response with status code {}",
            resp.status()
        );
        Ok(resp)
    }
}
```

`execute_impl`函数就是数据格式转化+调用wasm的handle_http_request的函数。

```rust
impl SpinHttpExecutor {
    pub async fn execute_impl(
        mut store: Store,
        instance: Instance,
        base: &str,
        raw_route: &str,
        req: Request<Body>,
    ) -> Result<Response<Body>> {
        //...
        let http_instance = SpinHttp::new(&mut store, &instance, |data| data.as_mut())?;
        //...
        let req = crate::spin_http::Request {
            method,
            uri: &uri,
            headers: &headers,
            params: &params,
            body,
        };

        let resp = http_instance.handle_http_request(&mut store, req).await?;

        if resp.status < 100 || resp.status > 600 {
            tracing::error!("malformed HTTP status code");
            return Ok(Response::builder()
                .status(http::StatusCode::INTERNAL_SERVER_ERROR)
                .body(Body::empty())?);
        };

        let mut response = http::Response::builder().status(resp.status);
        if let Some(headers) = response.headers_mut() {
            Self::append_headers(headers, resp.headers)?;
        }

        let body = match resp.body {
            Some(b) => Body::from(b),
            None => Body::empty(),
        };

        Ok(response.body(body)?)
    }
  //...
}
```

`handle_http_request`从哪里来的呢，这个就涉及到rust与wasm交互的一个库`wit_bindgen`在程序中只是一行代码`wit_bindgen_wasmtime::import!({paths: ["../../wit/ephemeral/spin-http.wit"], async: *});`这里`spin-http.wit`文件中定义的就是wasm交互的interface，以下就是这个文件的内容（类型信息被抽取到`http-types`文件中了，只有类型定义信息这里就不附了，有兴趣的同学可以在源码里看到）。

```wit
use * from http-types

// The entrypoint for an HTTP handler.
handle-http-request: func(req: request) -> response
```



### 小结

使用内置http trigger的方式运行一个wasm程序的源码大致如上所述，这里可以看到spin目前支持http和redis两个内置的wasm程序触发方式，而且仅有这两种方式，用户来自己定义更多的触发方式看起来只能通过[扩展源码](https://developer.fermyon.com/spin/extending-and-embedding)。自定义扩展经过后面的js2wasm案例来看的话参与的是编译程序到wasm过程，还有另一个自定义扩展的案例是直接运行wasm程序。spin与wasm程序交互通过wasm component model进行定义，这是一个wasm本身在推进的协议，对于其他语言的支持可以借助社区工具，这就是一个比较大的优势（不过看了下go的sdk，仍然还是要编写一些胶水代码的）。但是同时也由于这是一个WebAssembly的proposal并且处于早期，所以标准落地以及更多语言生态支持应该也会慢一些。所以fermyon还支持了[wagi](https://github.com/deislabs/wagi)，一个WebAssembly的[Common Gateway Interface](https://datatracker.ietf.org/doc/html/rfc3875)定义（主要用于初期过渡，看文档后期component model生态成熟了会废弃）。

### 附

spin.lock文件内容

```json
{
  "spin_lock_version": 0,
  "metadata": {
    "description": "a spin http handler in go for wasm",
    "name": "spinhttpgo",
    "origin": "file:///Users/root/Desktop/Workspace/github/spin/spinhttpgo/spin.toml",
    "trigger": {
      "base": "/",
      "type": "http"
    },
    "version": "0.1.0"
  },
  "triggers": [
    {
      "id": "trigger--spinhttpgo",
      "trigger_type": "http",
      "trigger_config": {
        "component": "spinhttpgo",
        "executor": null,
        "route": "/hello"
      }
    }
  ],
  "components": [
    {
      "id": "spinhttpgo",
      "metadata": {
        "allowed_http_hosts": []
      },
      "source": {
        "content_type": "application/wasm",
        "source": "file:///Users/root/Desktop/Workspace/github/spin/spinhttpgo/main.wasm"
      }
    }
  ]
}
```

spin.toml文件

```toml
spin_version = "1"
authors = ["zhangchao <zchao9100@gmail.com>"]
description = "a spin http handler in go for wasm"
name = "spinhttpgo"
trigger = { type = "http", base = "/" }
version = "0.1.0"

[[component]]
id = "spinhttpgo"
source = "main.wasm"
[component.trigger]
route = "/hello"
[component.build]
command = "tinygo build -wasm-abi=generic -target=wasi -gc=leaking -no-debug -o main.wasm main.go"
```

参考

https://developer.fermyon.com/spin/http-trigger

https://developer.fermyon.com/spin/distributing-apps

https://developer.fermyon.com/spin/plugin-authoring

https://developer.fermyon.com/spin/extending-and-embedding
