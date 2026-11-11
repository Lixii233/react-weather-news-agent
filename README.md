# ReAct 智能天气与新闻助手

用 ReAct（Reasoning + Acting）模式让模型自己决定"调哪个工具"，把实时天气与新闻查询接进对话。

- 技术栈：Python · LangChain · FastAPI · OpenWeatherMap API · Tavily Search API
- 状态：本地可运行，未部署线上

## 1. 解决什么问题

大模型本身拿不到实时天气和新闻，回答时只能给训练数据里那一刻的信息，容易过时或答非所问。

这个项目把两个外部 API 封装成工具，让模型在多轮对话里自己判断——这个问题要不要查天气、要不要查新闻、还是两个都要——然后调用、读结果、再组织回答。

## 2. 怎么做的

执行流程：

```
用户提问 → Thought（要不要调工具、调哪个）
        → Action（带 JSON 参数调用）
        → Observation（把返回结果喂回模型）
        → 继续思考，或直接生成回答
```

三个关键取舍：

1. **用 ReAct 而不是写死 if-else 分支。**
   用户可能只问天气、只问新闻、也可能两个都问。规则分支穷举不完，把"调不调、调哪个"交给模型判断更省事；代价是偶尔判断错，所以后面加了兜底。

2. **工具用 JSON Schema 描述参数。**
   不写 Schema 时模型经常把 city 写成 location、把日期写成中文"明天"。显式约束参数名和格式后，参数错误明显减少。

3. **错误当 Observation 回灌，而不是直接抛异常。**
   API 超时、城市名拼错、参数格式不对，都把这些错误信息作为观察结果返回给模型，让它自己改参数重试；另加最大执行步数，防止死循环。

上下文管理：ConversationBufferWindowMemory 保留最近若干轮，再叠一层摘要压缩历史，避免多轮之后 token 线性增长。

## 3. 怎么跑起来

```bash
git clone <repo-url>
cd react-weather-news-agent
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts ctivate
pip install -r requirements.txt
```

运行前需要配置三个环境变量（Windows 用 set，macOS/Linux 用 export）：

| 变量 | 说明 |
|---|---|
| OPENAI_API_KEY | 模型调用密钥 |
| OPENWEATHER_API_KEY | 天气查询 |
| TAVILY_API_KEY | 新闻搜索 |

```bash
uvicorn app:app --reload
curl -X POST http://127.0.0.1:8000/chat -H "Content-Type: application/json"   -d '{"session_id":"demo","message":"广州明天天气怎么样？顺便看看有没有相关科技新闻"}'
```

## 4. 已知局限

- 只有两个工具，工具选择准确率和失败率都没做过统计，属于"感觉能用"级别；
- 没有评测集，回答质量靠人工看几条判断；
- 摘要压缩是有损的，特别长的对话里早期约束可能被压掉；
- 没做并发和限流，单实例只适合本地演示。

## 5. 下一步

- 造 20 条测试问题，统计工具选择准确率与参数错误率；
- 用 pydantic 在调用前做参数校验，把"参数错"挡在 API 之外。
