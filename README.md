# 参考视频成片制作 Skill

给 Codex 使用的一份完整视频制作 Skill：输入参考视频、修改要求和已认可素材，按镜头拆解、锁定改动、生成提示词、制作变化镜头、质检，再导出 MP4。长视频按镜头和时长限制分段处理。默认使用火山方舟 Seedance 2.0、720p。

**[下载完整 Skill ZIP](https://github.com/Aiintalk/reference-video-production-skill/releases/latest/download/reference-video-production.skill.zip)**

仓库中的 [Skill 源码](skills/reference-video-production/SKILL.md) 可以直接查看；ZIP 包含相同的完整目录和脚本。

## 安装

1. 下载 ZIP 并解压，得到 `reference-video-production` 文件夹。
2. 将整个文件夹放到 `~/.codex/skills/`。Windows 放到 `%USERPROFILE%\.codex\skills\`。
3. 重新打开 Codex。在该 Skill 文件夹中运行 `python3 scripts/setup.py`（Windows 用 `python`）。此步骤安装所需 Python 依赖和 FFmpeg 执行器。
4. 运行 `python3 scripts/run.py configure --all`，在本机隐藏输入中填写自己的火山方舟视频生成 API Key、Access Key ID 和 Secret Access Key。运行 `python3 scripts/run.py status` 可检查是否已配置。Windows 将这些命令中的 `python3` 换成 `python`。不要把密钥发到聊天中，也不要写进 Skill 文件。
5. 在 Codex 中说：“使用 `$reference-video-production`，根据这个参考视频和我的修改要求生成成片。”只要分析或提示词，也可以明确指定范围。

安装后的目录应包含 `~/.codex/skills/reference-video-production/SKILL.md`。完整接入说明见[方舟与虚拟素材库指南](skills/reference-video-production/references/ark-setup-and-usage.md)。

## 工作流程

参考视频与修改要求 → ①拆解镜头 → ②锁定修改范围 → ③分配素材 → ④口播分段 → ⑤逐镜头提示词 → ⑥只生成变化镜头 → ⑦实际视频质检与局部修复 → ⑧拼接字幕音乐并导出。

完全不变的镜头直接复用；只改字幕或音乐的镜头在剪辑阶段处理。要成片时执行全部八步。

## 接口与权限

- 内置火山方舟 Seedance 2.0 客户端、私域虚拟人像素材库操作和 FFmpeg 执行器；不需要另装 Hypit Skill。
- 视频生成使用方舟 API Key；虚拟素材库管理使用 Access Key ID 与 Secret Access Key。账号还需有模型、项目、额度和素材库权限。
- 上传本地素材至虚拟素材库时，需使用者已有的北京 TOS 存储桶及相应权限。已有可访问的 HTTPS 素材链接可以直接创建素材资产。
- 密钥保存在使用者本机的系统凭据库，不随 ZIP 或仓库分发。配置完成不等于自动产生付费任务。

生成片段的台词、人物、声音和口型需根据实际视频逐段验收；提示词或 API Key 本身不能保证完全一致。
