# ArxivSIGN

每日 arXiv + HuggingFace 论文采集 → LangGraph Agent 摘要生成 → SovitsTTS 语音合成 → 静态 HTML 卡片展示 + Railway 反馈服务。

**公网地址：** https://echograil.github.io/arxivsign-web  
**关于本站：**[ArxivSIGN — About](https://echograil.github.io/arxivsign-web/about.html)

---

## 快速上手

### 环境要求

- Python 3.10+
- GPT-SoVITS v4（本地 TTS，可选）
- Git

### 安装依赖

```bash
pip install openai langchain langchain-openai langgraph llama-index-core llama-index-embeddings-huggingface flask flask-cors flask-limiter requests python-dotenv tqdm
```

### 配置

在 `Step2cleaner/` 目录下新建 `.env`：

```
OPENAI_API_KEY=your_key
```

如需切换模型接口（如 aihubmix）：

```
# cleaner_v2.py 顶部
llm = ChatOpenAI(
    model="gpt-4o-mini",
    base_url="https://aihubmix.com/v1"
)
```

### 一键运行

```bash
python run_all.py
```

启动后交互式询问是否采集、是否生成 TTS，回车默认跳过，直接跑 Step2 + Step4。

```bash
python run_all.py --from step4   # 只重新渲染 HTML
python run_all.py --from step2   # 从摘要开始续跑
```

---

## 管线说明

```
Step1  arxiv_fetcher.py + hf_fetcher.py + merger.py
         → Step1sources/out_merge/merged_result.json

Step2  cleaner_v2.py   （LangGraph Best-of-5 摘要生成）
         → Step2cleaner/tts_ready_v2.json

       cleaner_RAG.py  （可选，RAG 对照实验版本）
         → Step2cleaner/tts_ready_RAG.json

Step3  tts_runner.py   （需要本地 sovitsTTS 服务在 9880 端口）
         → Step3tts/tts_result.json
         → Step3tts/audio/*.wav

Step4  step4_extractv2.py   → Step4visible/enriched_v2.json
       step4_visualize_v4.py → Step4visible/papers_wall_v4.html

Step4c feedback_server.py   （Flask，可选，管反馈和点赞）
         → Step4visible/feedback.jsonl
```

---

## 部署

### 本地预览

直接用浏览器打开 `Step4visible/papers_wall_v4.html`。

音频需要 feedback_server 在线才能播放本地文件：

```bash
python Step4visible/feedback_server.py
```

### 公网发布

```bash
python deploy.py          # 推送 HTML + 音频到 GitHub Pages
python deploy.py --dry    # 预览，不实际操作
```

发布前修改 `deploy.py` 顶部：

```python
WEB_REPO     = Path(r"你的 arxivsign-web 本地路径")
FEEDBACK_URL = "https://你的railway域名.railway.app"
```

### Railway 部署 feedback_server

1. 新建 GitHub 仓库 `arxivsign-feedback`，上传：
   
   - `Step4visible/feedback_server.py`
   - `requirements.txt`（flask, flask-cors, flask-limiter）
   - `Procfile`（`web: python feedback_server.py`）

2. Railway → New Project → Deploy from GitHub repo → 选该仓库

3. Variables 添加：`ADMIN_TOKEN`（自定义）、`GITHUB_TOKEN`、`GITHUB_REPO` 等

---

## 文件结构

```
ArxivSIGN/
├── run_all.py              # 一键管线入口
├── deploy.py               # 一键发布到 GitHub Pages
├── Step1sources/
│   ├── arxiv_fetcher.py
│   ├── hf_fetcher.py
│   ├── merger.py
│   └── out_merge/
│       ├── merged_result.json    # 累积归档，不会被覆盖
│       └── archive/              # 每次采集前自动备份
├── Step2cleaner/
│   ├── cleaner_v2.py             # 主力：LangGraph Best-of-5
│   ├── cleaner_RAG.py            # 对照：RAG few-shot 版本
│   ├── tts_ready_v2.json
│   └── 旧版/
├── Step3tts/
│   ├── tts_runner.py
│   ├── tts_result.json
│   └── audio/
├── Step4visible/
│   ├── step4_extractv2.py
│   ├── step4_visualize_v4.py
│   ├── feedback_server.py
│   ├── enriched_v2.json
│   ├── papers_wall_v4.html
│   ├── feedback.jsonl
│   └── 旧版/
```

---

## 关键配置项

| 文件              | 变量                      | 说明              |
| --------------- | ----------------------- | --------------- |
| `cleaner_v2.py` | `START_INDEX` / `LIMIT` | 断点续跑范围          |
| `tts_runner.py` | `TTS_BACKEND`           | `local` 或 `api` |
| `tts_runner.py` | `REF_AUDIO_PATH`        | 参考音频路径          |
| `deploy.py`     | `WEB_REPO`              | 本地 web 仓库路径     |
| `deploy.py`     | `FEEDBACK_URL`          | Railway 服务地址    |
| `run_all.py`    | `SCRIPTS`               | 各步骤脚本路径         |

---

## 数据流说明

所有步骤通过 JSON 文件交换数据，各步骤可独立运行：

- 每步输出立即落盘，断点续跑安全
- `merger.py` 累积归档，新数据追加而不覆盖
- TTS 用文件名哈希缓存，已生成的音频自动跳过
- 旧版本文件保留在 `旧版/` 目录，迭代时不丢失存档
