# 火山方舟接入与使用

本 Skill 自带 `scripts/ark.py`，调用官方火山方舟生成接口与私域虚拟人像库接口；无需安装 Hypit Skill。以下命令在本 Skill 目录运行。Python 3、网络和实际账号权限仍是运行条件。

## 首次配置

1. `python3 scripts/setup.py`：在用户目录建立独立 Python 环境并安装 `keyring`、官方 TOS Python SDK `tos` 与 FFmpeg 执行器 `imageio-ffmpeg`。Windows 可用 `python`。
2. `python3 scripts/run.py configure --all`：交互输入方舟视频生成 API Key、Access Key ID、Secret Access Key。输入不显示，保存在本机系统凭据库；也可用环境变量 `ARK_API_KEY`、`VOLC_ACCESS_KEY_ID`、`VOLC_SECRET_ACCESS_KEY`，但不要写入项目或提交版本控制。
3. `python3 scripts/run.py status`：仅检查三个凭据是否存在，不读取或输出密钥值，不访问网络。

视频裁切、转码、拼接与混音可用 `python3 scripts/run.py ffmpeg <FFmpeg参数>`。执行器随首次配置安装，无需另外安装 Hypit 或系统 FFmpeg；带时间码转写仍需另有可用能力或由 Codex 对音频进行核对。

视频生成接口使用 API Key；素材库管理使用 AK/SK。Key 不会自动开通模型、素材库、IAM 项目或额度。首次创建素材组前，账号可能需在控制台完成素材库开通与授权函。API Key 所属项目、素材组与素材资产的 `ProjectName` 要一致。

## 上传与调用虚拟素材库

先查看已有素材：

- `python3 scripts/run.py groups --project default`
- `python3 scripts/run.py assets --project default --group-id group-...`

新建虚拟人像素材组：

- `python3 scripts/run.py group-create --name 我的虚拟人物 --project default`

上传现有 HTTPS 链接：

- `python3 scripts/run.py asset-create --group-id group-... --type Image --url https://example.com/person.png --project default`

上传本地文件：

- `python3 scripts/run.py asset-create --group-id group-... --type Video --file /absolute/path/clip.mp4 --bucket YOUR_CN_BEIJING_TOS_BUCKET --project default`

本地文件先上传到使用者已有的北京 TOS 桶，再用有效的临时签名链接调用 `CreateAsset`。AK/SK 同时要有该桶的上传与签名权限。素材源 URL 不输出到终端，因为签名链接包含临时访问凭据。工具不会创建或公开存储桶。

`CreateAsset` 是异步处理。获得 `asset-...` 后运行：

- `python3 scripts/run.py asset-wait asset-... --project default`

仅当 `Status=Active` 才可在生成请求中使用 `asset://asset-...`。`Failed` 时查看返回的 `Error`。在提示词里使用“图片1、视频1、音频1”等顺序指代，而不是直接写资产 ID。

## 提交 Seedance 2.0

为每个需要改变的短镜头写一个 JSON 文件，例如：

```json
{
  "prompt": "图片1中的虚拟人物保持身份、服装和动作，背景换为轻医美咨询间。准确说出：示例台词。声音参考音频1，语速与参考一致，口型对应中文发音。",
  "duration": 5,
  "resolution": "720p",
  "project": "default",
  "ratio": "9:16",
  "generate_audio": true,
  "references": [
    {"type": "image", "url": "asset://asset-EXAMPLE"},
    {"type": "audio", "url": "https://example.com/voice.mp3"}
  ]
}
```

把示例 URL 与 ID 换成真实素材。`resolution` 不写时默认 720p；模型固定为官方 Seedance 2.0 `doubao-seedance-2-0-260128`。所有 `asset://` 引用须为 `project` 所指项目下已激活素材；提交前工具会逐个核对状态、项目和素材类型。`project` 用于素材核对，不是视频生成接口的独立参数；API Key 本身须归属同一项目。

### 换人物、动作与时机不变

把待修改的原镜头作为视频1，把新人物图片作为图片1；若新人物只提供视频，可将其作为视频2或提取清晰身份帧作为图片1，且在参考素材时长上限内提交。新人物素材只定义身份和用户指定的外观，动作、位置、表情变化、镜头节奏和时间轴以视频1为准。先观察原镜头并记录动作锚点，再在该镜头的 `prompt` 中写入实际时间，不得照抄下面的示意秒数：

> 严格编辑视频1，只将其中原人物替换为图片1中的新人物。图片1只决定新人物的脸、发型、体型和指定服装；视频1是唯一动作和时间轴来源。保持视频1的总时长、人物站位、身体朝向、手臂与手指轨迹、视线变化、动作起止、停顿、背景、道具、机位、构图和切点。0.0–1.2秒：[原镜头动作A]；1.2–2.8秒：[原镜头动作B]；2.8–5.0秒：[原镜头动作C]。新人物在相同时间点完成每个动作，不增加、删除、重排动作，不改变速度和停顿。只替换人物身份，不继承原人物的脸和外观。

若身份素材是视频2，把“图片1”改成“视频2”，并增加“视频2只提供新人物身份，不继承其动作或节奏”。若还要修改台词，另写准确台词，允许嘴唇按新台词变化，身体动作时间轴仍以视频1为准。这里是**视频编辑**任务，提示词使用“严格编辑视频1”；仅借用动作生成全新场景时才写“参考视频1”。素材序号须与请求体中同类素材的实际顺序一致，视频和图片本身必须通过 `references` 传入。逐帧完全一致不能靠提示词保证，须对照原镜头逐项验收。

### 编辑原视频与参考动作生成的选择

**编辑原视频**把视频1当作待修改的底稿：尽量沿用原时长、人物位置、动作时序、背景与运镜，只替换用户指定的元素，写“严格编辑视频1，只将……改为……”。**参考动作生成**把视频1当作动作灵感：生成的是新画面，允许用户指定的新场景或新镜头安排，写“参考视频1中的[具体动作]，生成……”。当用户要求“换人且每个动作的时机、节奏保持原片一致”，选编辑原视频。两种方式都不保证逐帧一致，需看实际结果。

### 把提示词写细，同时严格守住修改范围

1. 先拆原片：逐镜头记录起止时间、动作与手势的起止、轨迹、停顿、视线、道具接触、镜头运动和切点；只记录看见或听见的事实，未知处标“未知”。
2. 逐镜头列四栏：“必须修改、必须保留、禁止继承、可自由补充”。用户未提出的变化不要自行加入；用户提供的准确台词、产品名和数字保持原文。
3. 只为需要生成的镜头写提示词。先说明视频1、图片1、视频2等素材各负责什么，再按真实相对时间码写动作段，最后写保留项和禁止项。详细程度来自可核对的事实，不靠堆叠风格形容词。新人物视频只用于身份时明确排除它的动作。
4. 提交前逐句对照四栏，删掉没有来源或超出修改范围的描述；核对素材编号与请求体顺序、模型、720p、时长及声音参数。生成后按同一时间码对照原片验收。

给 AI 的任务指令可直接写：“先观察参考视频，输出逐镜头四栏修改表和带时间码的动作表，再按表写 Seedance 2.0 提示词。每一句细节都要来自我的要求或实际素材；不清楚的标未知。只替换我指定的内容，其他内容保留。换人物且动作不变时，使用视频编辑提示词，原视频是唯一动作时间轴，新人物素材只提供身份；逐句检查提示词是否越界。”

### 在提示词里描述用户提供的素材

先为实际提供的素材写描述卡，再把**本镜头会用到**的内容压缩成一段放入提示词开头。描述卡可按以下字段填写；方括号都要替换为实际观察结果，未提供或看不清的字段不写：

> 本镜头素材描述：图片1是新人物，能确认的外观为[脸型/发型/体态/服装/辨识特征]，只负责新人物身份与指定外观；图片2是目标背景，能确认的空间特征为[布局/材质/颜色/光线/物件位置]，只负责背景；图片3是目标产品，能确认的外形为[包装结构/主色/瓶盖或盒型/可辨标识]，只负责产品外观；音频1的可听特征为[音色/语速/停顿]，只负责用户指定的声音属性；视频1是待编辑的原镜头，只负责原动作、人物位置和时间轴。其余属性按修改清单保留。

若新人物来自视频2，描述其可见的稳定外观，并明确“视频2不提供动作、表情变化和镜头节奏”。只有用户提供的人物、背景、产品素材才出现对应编号；纯文字设定标为“用户文字要求”，不能伪装成图片。产品上的文字若看不清，标“未知”，需准确呈现的字样由用户提供并在成片剪辑阶段校对。描述必须与实际上传的素材编号一致，不能凭文字描述替代接口素材引用。

执行：

1. `python3 scripts/run.py generate --spec /absolute/path/shot01.json --dry-run`：本地检查格式与参数，不产生费用。
2. `python3 scripts/run.py generate --spec /absolute/path/shot01.json`：提交一次生成，并立即写入同名 `.receipt.json`，避免中断后重复提交。
3. `python3 scripts/run.py task-wait cgt-... --output /absolute/path/shot01.mp4`：等待并下载实际结果。中断后用同一任务 ID 继续查询，不重新提交。

接口接受的是短片段。长视频应保留原片时间码、镜头表、统一角色/声音资产 ID，逐段提交，再用视频工具拼接与质检完整 MP4。精确声音、动作与口型不是 API Key 或提示词能保证的；必须验收实际片段。

## 官方接口依据

- [Seedance 2.0 教程](https://docs.volcengine.com/docs/ark/seedance-2-0?lang=zh)
- [创建视频生成任务](https://docs.volcengine.com/docs/ark/create-video-generation-task-api?lang=zh)
- [私域虚拟人像库使用指南](https://docs.volcengine.com/docs/ark/private-virtual-avatar-library-guide-preview?lang=zh)
- [查询素材状态](https://docs.volcengine.com/docs/ark/get-asset-api?lang=zh)
- [火山 TOS Python SDK](https://github.com/volcengine/ve-tos-python-sdk)
