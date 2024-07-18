
# 微软 GraphRAG 本地部署

## 环境准备
1、GPU服务器 2G显卡用于运行本地bge-large-zh-v1.5对文本做向量化  
2、搭建ONEAPI环境，用于调用GLM4进行实体抽取和图建立。**有条件可在本地运行LLM模型，但需要GPU服务器。**  
3、python环境，本人测试使用python3.10.8，**熟悉docker的也可以直接在docker中运行，下载python镜像即可。**  

## 搭建bge向量化模型  
下载bge权重文件https://huggingface.co/BAAI/bge-large-en-v1.5/tree/main  
使用app_fastgpt.py启动向量化模型   

## 安装ONEAPI步骤
### 搭建ONEAPI服务并配置环境
https://github.com/songquanpeng/one-api  

使用docker安装，下载镜像后运行：  
- 使用 SQLite 的部署命令：
docker run --name one-api -d --restart always -p 3000:3000 -e TZ=Asia/Shanghai -v /home/ubuntu/data/one-api:/data justsong/one-api
- 使用 MySQL 的部署命令，在上面的基础上添加 `-e SQL_DSN="root:123456@tcp(localhost:3306)/oneapi"`，请自行修改数据库连接参数，不清楚如何修改请参见下面环境变量一节。
- 例如：
docker run --name one-api -d --restart always -p 3000:3000 -e SQL_DSN="root:123456@tcp(localhost:3306)/oneapi" -e TZ=Asia/Shanghai -v /home/ubuntu/data/one-api:/data justsong/one-api

### 配置oneapi连接智普GLM4
在浏览器输入http://localhost:3000/，进入oneapi管理界面。  
默认登录密码 root/1234，登录后进入oneapi管理界面。
在渠道处添加智普的信息，api-key  
![](images/oneapi-glm.png)
同样把搭建好的bge向量化模型配置到oneapi中，在模型页面添加模型。  
![](images/oneapi-bge.png)
在令牌页面添加一个新的令牌，配置好权限后，将token复制出来。  
![](images/oneapi-token.png)
![](images/oneapi-token1.png)

## 运行GraphRAG
可参考网址 https://juejin.cn/post/7390185041374887946启动GraphRAG  
**使用上面配置OneAPI的token，启动GraphRAG，不需要按照OLlama启动模型。**  
配置文件更新为附件settings.yaml，替换文件中的ip地址和token。  

使用清华源安装  
pip install graphrag -i https://pypi.tuna.tsinghua.edu.cn/simple  
创建目录  
mkdir -p ./ragtest/input  
下载测试文件  
https://www.gutenberg.org/cache/epub/24022/pg24022.txt  
使用 wget https://www.gutenberg.org/cache/epub/24022/pg24022.txt  

将文件放入input目录  
mv pg24022.txt ./ragtest/input/book.txt  

初始化工作空间  
python -m graphrag.index --init --root ./ragtest  

将配置文件替换为settings.yaml,覆盖    
./ragtest/settings.yaml   

启动GraphRAG，构建知识图谱  
python -m graphrag.index --root ./ragtest  

最后构建报错，修改源码  
修改源码/usr/local/python3.10/lib/python3.10/site-packages/graphrag/llm/openai/utils.py  
```
代码第90行修改为如下：
def try_parse_json_object(input: str) -> dict:
    """Generate JSON-string output using best-attempt prompting & parsing techniques."""
    try:
        if input.startswith("```json"):
            input = input[7:-3]
        result = json.loads(input)
    except json.JSONDecodeError:
        log.exception("error loading json, json=%s", input)
        raise
    else:
        if not isinstance(result, dict):
            raise TypeError
        return result


```



## 测试结果  
全局测试  
python -m graphrag.query --root ./ragtest --method global "What are the top themes in this story?"  
本地测试  
python -m graphrag.query --root ./ragtest --method local "Who is Scrooge, and what are his main relationships?"  
