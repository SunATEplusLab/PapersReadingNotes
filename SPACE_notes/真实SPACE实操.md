# 从真实例子学习 SPACE

本机已跑通真实 Firecracker：官方示例输出 `Hello from SPACE`、收到 `process_exited=0`，随后成功删除沙箱。部署使用固定源码 `4791af31eaa2`，没有修改官方示例。

**先做一次官方例子，再按 create → hello → workspace → failure 学。每次只看当前例子涉及的函数。**

## 1. 启动或恢复平台，并进入 root

**每次重启 Windows/WSL 后，先恢复数据卷和服务。** 如果目前看到 `acer@...$`，先输入 `exit` 回到 Windows PowerShell；然后运行：

```powershell
cd 'E:\好好读论文\agent安全\项目\Sandboxed Platform for Agentic Code Execution'
powershell.exe -NoProfile -ExecutionPolicy Bypass -File '.\学习材料\启动SPACE.ps1'
```

启动脚本会挂载并核对原 E 数据卷、恢复原有服务，再明确使用 `root --exec /bin/bash` 进入项目目录。需要挂载时 Windows 会弹出管理员授权窗口。等到 `SPACE_RESUME_READY` 后进入 Linux 终端，`whoami` 应输出 `root`。

这是日常学习用的恢复入口：复用已部署镜像，不启动 Tilt 的构建监听。需要修改 SPACE 平台源码并重新部署时，才使用官方开发启动流程。

需要网页控制台时，在另一个 root 终端运行 `kubectl --context kind-space-local -n space port-forward --address=127.0.0.1 svc/space-ui 3000:3000`，保持该终端开启，再访问 `http://127.0.0.1:3000`。

平台已经恢复时，也可以直接从 Windows PowerShell 进入 root；`--exec` 不要省略：

```powershell
& 'C:\Program Files\WSL\wsl.exe' -d SPACE-Ubuntu-24.04 -u root --cd '/mnt/e/好好读论文/agent安全/项目/Sandboxed Platform for Agentic Code Execution' --exec /bin/bash
```

接下来在这个 root Linux 终端运行：

```bash
source 学习材料/登录SPACE.sh && python3 /root/SPACE/examples/first_sandbox.py
```

登录脚本设置本机 API、数据平面、`seed:latest` 和 E 盘记录目录；token 只保留在终端环境中。`&&` 保证登录失败时不会继续运行示例。已经进入 root 并登录后，同一终端可以直接运行后面的课程。

`/mnt/e/…` 的项目文件夹可见，并不代表 `/mnt/wsl/space-build-e` 的独立 Linux 数据卷已经挂载；出现卷未挂载时重新运行启动入口。`/root/SPACE` 由 root 访问，普通用户的 `Permission denied` 也不代表源码丢失。

看到 `Hello from SPACE` 和 `Command completed successfully` 后，在删除提示处输入 `y`。这会等待删除操作成功，并清理该次命令的客户端事件文件。

本次成功示例的创建操作约 5 秒完成；这是本机这一次观测，不是性能承诺。完整配置与修复过程见[部署记录](../部署记录/2026-10-23-部署记录.md)，验收见[验证记录](验证记录.md)。

## 2. 四个递进练习

配套脚本：[progressive_examples.py](progressive_examples.py)。它直接导入实际部署的 `/root/SPACE/examples/first_sandbox.py`，调用官方函数。下面四课都请求真实平台、创建真实 VM；不需要大模型 API。

### 第 1 课：只创建和删除

```bash
python3 学习材料/progressive_examples.py create
```

观察 `sandbox_id`、`operation_id`，以及查询结果里的 `state: running`。本课不提交任务命令，最后输入 `y` 删除。

只读官方文件中的：`create_sandbox()` → `mutation()` / `api()` → `wait_for_state()` → `inspect_run()` → `delete_sandbox()`。

要理解：创建请求返回 ID 之后，后台仍要完成工作。**请求被接受、创建操作成功、guest 进程接口就绪，是不同阶段。** 官方 `create_sandbox()` 已依次等待。

小练习：在 `state.json` 找出创建请求的 body 和 `Idempotency-Key`。它们在发送请求前已保存；遇到超时先检查原记录，不要立即创建第二台 VM。

### 第 2 课：命令、环境变量、工作目录和输出

```bash
python3 学习材料/progressive_examples.py hello
```

实际命令设置了 `GREETING` 和 `cwd=/tmp`，并分别写 stdout、stderr。应看到：

```text
Hello from SPACE
/tmp
demo warning
Command completed successfully: proc_...
```

只增加阅读：`run_command()` → `event_frames()` → `consume_events()`。

`run_command()` 先向管理 API 创建 command session，取得进程 ID 和 attachment token，再连接数据平面的事件地址。**attach 才触发执行。** 管理 token 和这次会话的 attachment token 用途不同。

`consume_events()` 将 `data_base64` 解码为输出，并检查 `process_exited.exit_code`。本课虽然写了 stderr，但退出码为 0，因此成功。

小练习：把教学脚本本课的 `GREETING` 改成自己的文字，再运行一次；只改教学文件，无需重建 SPACE。

### 第 3 课：同一沙箱连续执行四条命令

```bash
python3 学习材料/progressive_examples.py workspace
```

四步分别是：写入 `12 8` → 求和 → 改成 `12 18` → 再求和。实际输出包括：

```text
Step 2: sum; same sandbox: sbx_...
total=20
Step 4: sum-again; same sandbox: sbx_...
total=30
```

观察四步使用**同一个 sandbox ID**，但各有独立的 process ID 和命令幂等键。读教学文件的 `command_record()`，再对照官方 `mutation()`、`save()`、`run_command()`。

要理解：一次命令结束后，VM 内文件仍可被下一次命令使用。客户端的 `state.json` / `.sse` 在记录目录里；`/tmp/space-study/numbers.txt` 在 guest 内。它们是两个位置。删除沙箱会结束这份 guest 工作环境。

小练习：先预测改成 `7 11` 后的结果，再改教学脚本、实跑确认。

### 第 4 课：故意失败，检查后清理

```bash
python3 学习材料/progressive_examples.py failure
```

guest 命令先打印 `starting check`，再向 stderr 写 `demo check failed`，最后 `exit 7`。官方事件处理函数报告：

```text
Stopped: Command did not exit successfully: 7
```

本课教学脚本退出码应为 **1**，原始事件中的 guest 退出码为 **7**。这是故意设计的失败；沙箱和记录会保留，便于检查。

复制本次打印的 `Run directory`，然后运行（把路径替换成自己的）：

```bash
RUN_DIR='/mnt/wsl/space-build-e/official-example-runs/space-lesson-failure-实际后缀'
python3 /root/SPACE/examples/first_sandbox.py inspect "$RUN_DIR"
python3 /root/SPACE/examples/first_sandbox.py delete "$RUN_DIR"
```

删除提示输入 `y`。读 `consume_events()` 的非零退出分支，再读 `delete_sandbox()`；区分“任务命令失败”和“沙箱是否仍存在”。

## 3. 怎么看记录，怎么讲清项目

每课的 `Run directory` 是客户端记录目录。想先看原始 `.sse`，可以在删除提示先答 `n`，读完再用上面的 `delete` 命令删除。每课清理后再开始下一课。

本次实际运行留下的证据在[实测证据目录](../部署记录/实测证据/)；其中 `workspace/step-*` 保留四步独立记录，`hello/` 和 `failure/` 保留删除前复制的事件。

汇报时沿着这条真实调用链讲即可：

```text
应用/Agent → 管理 API 创建 sandbox → 等待 operation 与 guest 就绪
         → 创建 command session → 带会话 token attach 数据平面
         → VM 内执行 → 解码 stdout/stderr → 检查退出码 → 删除 sandbox
```

先讲清这条链，再延伸网络策略、凭证隔离、btrfs 和快照。当前实测覆盖创建、执行、文件保留、失败处理和删除；安全边界与尚未验证的内容见验证记录。
