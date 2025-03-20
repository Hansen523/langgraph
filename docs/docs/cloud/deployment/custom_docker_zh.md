# 如何自定义Dockerfile

用户可以在从父LangGraph镜像导入后，添加一系列额外的行到Dockerfile中。为此，您只需修改`langgraph.json`文件，将您想要运行的命令传递给`dockerfile_lines`键。例如，如果我们想在图中使用`Pillow`，您需要添加以下依赖项：

```
{
    "dependencies": ["."],
    "graphs": {
        "openai_agent": "./openai_agent.py:agent",
    },
    "env": "./.env",
    "dockerfile_lines": [
        "RUN apt-get update && apt-get install -y libjpeg-dev zlib1g-dev libpng-dev",
        "RUN pip install Pillow"
    ]
}
```

这将安装使用`jpeq`或`png`图像格式所需的系统包。