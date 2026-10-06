# 小爱音箱 OH2P + Open-XiaoAI 折腾全记录（2026-09 ~ 2026-10）

> 作者：zhufanhaosha
> 机型：Xiaomi 智能音箱 Pro（OH2P）
> 环境：威联通 TS-551 NAS（Docker）+ 自制补丁固件
> 结论：最终恢复原厂系统，不再使用 Open-XiaoAI（详见文末"项目当前存在的问题"）

---

## 一、背景

- 音箱固件版本为 **1.62.2**，高于官方 Open-XiaoAI 支持的 **1.58.6**
- 官方补丁固件仅支持 1.58.6 / 1.94.13，跨版本刷机有变砖风险
- 因此决定**自制 1.62.2 匹配的补丁固件**（官方教程提供自制流程）

## 二、自制 1.62.2 补丁固件（阶段 0.5）

原理：项目脚本通过小米账号登录 OTA 接口，拉取设备当前版本（1.62.2）的原厂固件 → 解包 → 打补丁（开启 SSH、禁用系统更新、加入开机自启脚本）→ 重新打包。

在 NAS 上用 Docker 构建：

```bash
# 获取项目代码
docker run --rm -v /share/CACHEDEV1_DATA/open-xiaoai-build:/repo \
  alpine/git clone --depth 1 https://github.com/idootop/open-xiaoai.git /repo/open-xiaoai

# 配置 .env（小米账号 + 音箱名称 + SSH 密码）
cp .env.example .env
# 填入 MI_USER / MI_PASS / MI_DID / SSH_PASSWORD

# 执行构建
docker run -it --rm \
  --platform linux/amd64 \
  --env-file "$(pwd)/.env" \
  -v "$(pwd)/assets:/app/assets" \
  -v "$(pwd)/patches:/app/patches" \
  idootop/open-xiaoai:latest
```

产物：`assets/mico_all_*/root-patched.squashfs` → 重命名 `root_patched.squashfs`

## 三、刷机（阶段 1）

用 Amlogic Flash Tool v6.0.0（Windows + Type-C 数据线）：

```bash
./update.exe identify
./update.exe bulkcmd "setenv bootdelay 15"
./update.exe bulkcmd "setenv boot_part boot0"
./update.exe bulkcmd "saveenv"
./update.exe partition system0 root_patched.squashfs
```

刷后 SSH 登录（密码 open-xiaoai）。双系统：boot0 = 补丁系统，boot1 = 原厂系统，可随时切换。

## 四、部署（阶段 2 ~ 3）

音箱端：

```bash
mkdir -p /data/open-xiaoai
echo 'ws://NAS_IP:4399' > /data/open-xiaoai/server.txt
curl -sSfL https://gitee.com/idootop/artifacts/releases/download/open-xiaoai-client/init.sh | sh
curl -L -o /data/init.sh https://gitee.com/idootop/artifacts/releases/download/open-xiaoai-client/boot.sh
reboot
```

NAS 端（Docker）：

```bash
docker run -d --name open-xiaoai-xiaozhi --restart unless-stopped -p 4399:4399 \
  -v /share/.../xiaozhi-config/config.py:/app/config.py \
  idootop/open-xiaoai-xiaozhi:latest
```

## 五、问题与排查全记录

### 5.1 唤醒后说话没反应（ASR 不执行）

现象：唤醒词触发成功（提示音正常），但唤醒后说话 → VAD 检测到语音 → 没有 `💬 我说` → 20 秒超时"再见"。

排查：
- 单独测试 arecord / aplay 均正常，网络 0% 丢包
- VAD 日志显示能检测到语音开始/结束，但 ASR 环节不输出
- 对比新旧 config.py 发现：**新版本删除了 `vad.boost`（录音音量增强倍数）**
- 旧版 config（2025-06-16）有 `"boost": 10`，新版（2026-03-22）没有

解决：换回旧版 config.py（含 boost=10），录音被放大后识别恢复正常，短句长句都能识别。

### 5.2 Client 崩溃（Connection reset）

现象：`❌ 消息处理异常: WebSocket protocol error: Connection reset without closing handshake`，client 在收到 start_play（AI 播放）指令后崩溃。

排查：
- noop 设备（`type plug → Capture`）只能录音不能播放，`aplay -D noop` 报 dsnoop 错误
- 对比官方 1.58.6 固件 asound.conf，发现 noop 定义完全一致 —— 说明不是固件差异
- 真正影响：播放路径与系统音频栈（dmixer/hw:0,2）交互问题

尝试：bind mount 修改 asound.conf 的 noop 为 asym（playback+capture 双通道），client 播放打开不再失败。

### 5.3 Task 泄漏（Task was destroyed but pending）

现象：日志反复出现 `Task was destroyed but it is pending!`，多次唤醒/对话后 KWS 停摆、无法再次唤醒。

根因（源码分析）：
- `start_session()` 每次创建新 task，从不取消旧 task
- `next_step_future` 是共享单 Future，多会话并发时旧会话等一个没人 set 的 Future → 永久挂起
- `asyncio.wait` 返回后不取消 timeout task

修复（event.py 补丁）：
- `start_session()`：进入新会话前 `cancel()` 旧 task
- `wait_next_step()`：改用 `asyncio.wait_for`（标准库正确处理取消，不泄漏 waiter task）
- `update_step()`：set_result 前检查 `future.done()`
- `wakeup()`：`before_wakeup` 用 try/finally 包裹，确保 `kws.resume()` 一定执行

效果：`Task was destroyed` 归零，唤醒/对话稳定。

### 5.4 唤醒词（你好小智）时灵时不灵

原因：KWS（sherpa-onnx 小模型）对**自定义词**识别率低，官方 FAQ 也建议"换更易识别的词（如天猫精灵）"。

尝试：
- 调低 vad.threshold
- KWS sherpa.py 补丁：`keywords_score` 2.0→4.0、`keywords_threshold` 0.2→0.1、音频放大 3 倍
- KWS 去掉 LISTENING/SPEAKING 暂停条件（永不暂停，可随时唤醒/打断）

效果有限：唤醒词仍是主要痛点；"小爱同学 → 召唤小智"（原厂唤醒入口）反而稳定。

### 5.5 无法连续对话（回答一次后收不到指令）

原因：**Open-XiaoAI 官方已知 bug**（GitHub Issue #118，作者标记 not planned 不修）。TTS 回复结束后 on_tts_end 链路缺陷，无法自动进入下一轮监听。

绕过：每轮对话重新喊唤醒词。

### 5.6 其他踩坑

- 挂载文件名错误：`init.py` ≠ `__init__.py`，Python 实际 import `__init__.py`，挂载 init.py 不生效
- 新版镜像只有 `latest`/`1.0.0` 标签（同一镜像，2026-03-22），无法回退旧镜像；但 config.py 可以回退（GitHub commit 历史）
- 自制固件 /etc 只读，asound.conf 修改需 bind mount 或 HOME 覆盖
- 环境变量/容器重建后，docker cp 的补丁文件会丢失（需备份或挂载）

## 六、项目当前存在的问题（结论）

**1. 项目已停更（2026-04-04 归档，read-only）** —— 所有 bug 无人修复，104 个 issue 全部自动关闭。

**2. 唤醒词识别率低（自定义词）** —— KWS 小模型只对品牌词（天猫精灵等）识别稳定，自定义词时灵时不灵，官方建议换词，无解。

**3. 新版删除了 boost 参数** —— 导致录音音量弱、识别差；需回退旧版 config.py（含 boost=10）才恢复正常。

**4. Task 泄漏** —— 多次会话后 KWS 停摆；可用 event.py 补丁（wait_for 方案）修复，但需自行维护。

**5. 无法连续对话** —— 官方已知 bug（#118 not planned），只能每轮重新唤醒。

**6. 自制固件兼容性风险** —— 官方仅支持 1.58.6；自制 1.62.2 补丁虽可用，但音频栈交互存在未知问题（播放/录音冲突、偶发崩溃）。

**7. 安全性** —— 项目无加密/认证，client 具备执行脚本能力，仅限局域网使用；公网部署风险极高。

**8. 依赖云服务（小智 AI）** —— ASR/LLM 走 `api.tenclass.net`，云服务不稳定时全部失效；黑盒无法排查。

## 七、最终决定与恢复

- 折腾约一周后，决定**放弃 Open-XiaoAI，恢复音箱原厂**：
  - 音箱端：删除 /data/open-xiaoai、/data/init.sh，`fw_env -s boot_part boot1` + reboot 切回原厂系统（固件更新恢复正常）
  - NAS 端：`docker rm -f open-xiaoai-xiaozhi`，删除配置文件目录
  - 补丁固件仍保留在 system0（boot0），以后想再玩可随时切回或重刷

## 八、以后想再进 SSH 的方法

1. 若补丁系统未被 OTA 覆盖：刷机线 + Amlogic 工具切 boot0：
   ```bash
   ./update.exe bulkcmd "setenv boot_part boot0"
   ./update.exe bulkcmd "saveenv"
   ```
2. 若已被覆盖：用备份的 `root-patched-1.62.2.squashfs` 重新刷 system0。

## 九、给后来者的建议

- **能用官方 1.58.6 就不要自制新版本固件**（兼容性坑太多）
- **config.py 记得保留 boost=10**（旧版配置）
- **别用自定义唤醒词**，直接用官方推荐词
- **不要期待连续对话**（框架 bug）
- 想深度玩：研究 xiaozhi-esp32-server（协议活跃维护），或走"逆向固件 + 原厂唤醒 + 本地 ASR/LLM/TTS 管线"方案（需逆向 usock 协议）

---

*记录时间：2026-10-05*
