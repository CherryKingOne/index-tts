# IndexTTS API 接口文档

IndexTTS 使用 Gradio 构建 WebUI，同时提供了 API 接口供程序调用。

## 访问地址

| 端点 | URL | 说明 |
|------|-----|------|
| WebUI | `http://<服务器IP>:7860` | 可视化界面 |
| API 文档 | `http://<服务器IP>:7860/api` | Gradio API 文档 |
| OpenAPI | `http://<服务器IP>:7860/openapi.json` | OpenAPI 规范 |

## 启动 API 模式

默认 WebUI 已包含 API 支持。如需独立 API 服务，使用以下命令启动：

```bash
docker run -d \
  --name indextts-api \
  --gpus all \
  -p 7860:7860 \
  -v $(pwd)/checkpoints:/app/checkpoints \
  -v $(pwd)/outputs:/app/outputs \
  indextts:latest \
  python -c "
import uvicorn
from indextts.gradio_api import app
uvicorn.run(app, host='0.0.0.0', port=7860)
  "
```

## 接口列表

### 1. 生成 POST /api/generate

生成语音接口，WebUI 的核心功能。

#### 请求参数

```json
{
  "data": [
    "<情感控制方式索引: int>",
    "<音色参考音频路径或URL>",
    "<要合成的文本: str>",
    "<情感参考音频路径或URL>",
    "<情感权重: float, 0.0-1.0>",
    "<情感描述文本: str>",
    "<情感向量1-8: float[]>"
  ]
}
```

#### 使用示例

```python
import requests

url = "http://<服务器IP>:7860/api/generate"

payload = {
    "data": [
        0,                    # 情感控制方式: 0=与音色相同
        "prompt.wav",         # 音色参考音频路径
        "你好，欢迎来到Bilibili",  # 要合成的文本
        None,                 # 情感参考音频
        0.65,                 # 情感权重
        "",                   # 情感描述文本
        0, 0, 0, 0, 0, 0, 0, 0  # 情感向量 (8维)
    ]
}

response = requests.post(url, json=payload)
print(response.json())
```

### 2. 使用 Gradio Client (推荐)

Gradio 提供了官方客户端，使用更简单：

```python
from gradio_client import Client

client = Client("http://<服务器IP>:7860")

result = client.predict(
    0,                           # emo_control_method
    "prompt.wav",               # prompt_audio (音色参考音频)
    "你好，欢迎来到Bilibili",      # text
    None,                        # emo_audio_prompt
    0.65,                        # emo_weight
    "",                          # emo_text
    0, 0, 0, 0, 0, 0, 0, 0,    # emo_vector (8维)
    api_name="/gen_single"
)

print(result)  # 返回生成的音频文件路径
```

安装 Gradio Client：
```bash
pip install gradio_client
```

## 完整调用示例

### Python

```python
import requests
import json
import base64

API_URL = "http://<服务器IP>:7860/api/generate"

def generate_tts(text: str, prompt_audio: str, output_path: str):
    """调用 IndexTTS 生成语音"""
    payload = {
        "data": [
            0,                    # 情感控制方式
            prompt_audio,         # 音色参考音频
            text,                 # 文本
            None,                 # 情感参考音频
            0.65,                 # 情感权重
            "",                   # 情感描述文本
            0, 0, 0, 0, 0, 0, 0, 0
        ]
    }
    
    response = requests.post(API_URL, json=payload)
    result = response.json()
    
    # 下载生成的音频
    audio_url = result["data"][0]["url"]
    audio_response = requests.get(f"http://<服务器IP>:7860{audio_url}")
    
    with open(output_path, "wb") as f:
        f.write(audio_response.content)
    
    return output_path

# 使用示例
generate_tts(
    text="你好，欢迎来到Bilibili",
    prompt_audio="path/to/prompt.wav",
    output_path="output.wav"
)
```

### cURL

```bash
# 生成语音
curl -X POST http://<服务器IP>:7860/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      0,
      "prompt.wav",
      "你好，欢迎来到Bilibili",
      null,
      0.65,
      "",
      0, 0, 0, 0, 0, 0, 0, 0
    ]
  }'
```

### JavaScript

```javascript
async function generateTTS(text, promptAudio) {
  const response = await fetch('http://<服务器IP>:7860/api/generate', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      data: [
        0,                    // 情感控制方式
        promptAudio,          // 音色参考音频
        text,                 // 文本
        null,                 // 情感参考音频
        0.65,                 // 情感权重
        '',                   // 情感描述文本
        0, 0, 0, 0, 0, 0, 0, 0
      ]
    })
  });
  
  const result = await response.json();
  return result.data[0];  // 返回音频信息
}
```

## 参数说明

| 参数 | 类型 | 说明 |
|------|------|------|
| emo_control_method | int | 情感控制方式: 0=与音色相同, 1=情感参考音频, 2=情感向量, 3=情感文本描述 |
| prompt_audio | str | 音色参考音频文件路径 |
| text | str | 要合成的文本 |
| emo_audio_prompt | str/null | 情感参考音频文件路径 |
| emo_weight | float | 情感权重, 0.0-1.0 |
| emo_text | str | 情感描述文本 (当 mode=3) |
| emo_vector | float[8] | 情感向量 (当 mode=2): 喜、怒、哀、惧、厌恶、低落、惊喜、平静 |

## 错误处理

```python
response = requests.post(API_URL, json=payload)

if response.status_code == 200:
    result = response.json()
    print("生成成功:", result["data"][0])
else:
    print(f"请求失败: {response.status_code}")
    print(response.text)
```

## 注意事项

1. **音频文件**：需先通过 WebUI 上传或放置在挂载目录中
2. **GPU 加速**：确保已正确配置 NVIDIA Container Toolkit
3. **模型加载**：首次需要下载模型，请耐心等待
4. **端口开放**：确保服务器防火墙开放 7860 端口
