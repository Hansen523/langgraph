# API参考

LangGraph Cloud的API参考可在每个部署的`/docs` URL路径下找到（例如`http://localhost:8124/docs`）。

点击<a href="/langgraph/cloud/reference/api/api_ref.html" target="_blank">此处</a>查看API参考。

## 认证

对于部署到LangGraph Cloud的情况，需要进行认证。每次向LangGraph Cloud API发出请求时，需传递`X-Api-Key`头信息。该头的值应设置为部署API的组织中有效的LangSmith API密钥。

示例`curl`命令：
```shell
curl --request POST \
  --url http://localhost:8124/assistants/search \
  --header 'Content-Type: application/json' \
  --header 'X-Api-Key: LANGSMITH_API_KEY' \
  --data '{
  "metadata": {},
  "limit": 10,
  "offset": 0
}'  
```