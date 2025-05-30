~~~bash
# 启动ES的容器
docker run -d \
  --name elasticsearch \
  -p 9200:9200 \
  -p 9300:9300 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -v /Users/chenhaiqing/Documents/GitHub/ExerciseRxAI/es_data:/usr/share/elasticsearch/data \
  docker.elastic.co/elasticsearch/elasticsearch:8.11.1
  
# 查看初始密码
docker exec -it elasticsearch bin/elasticsearch-reset-password -u elastic
~~~



~~~python
"""
项目入口
"""

from elasticsearch import Elasticsearch
import json

class ESSearch:
    def __init__(self, index_name: str):
        self.es = Elasticsearch("http://localhost:9200", basic_auth=("elastic", "NOkPqYLang9rn7vyS2mq"))
        self.index_name = index_name

    def index_files(self, directory_path: str):
        """
        批量索引目录下的所有文件
        Args:
            directory_path: 目录路径
        """
        import os
        import glob
        
        # 获取目录下所有文件路径
        file_paths = glob.glob(os.path.join(directory_path, "**"), recursive=True)
        file_paths = [f for f in file_paths if os.path.isfile(f)]  # 过滤掉目录
        
        actions = []
        for i, file_path in enumerate(file_paths):
            try:
                with open(file_path, 'r', encoding='utf-8') as f:
                    content = f.read()
                action = {
                    "_index": self.index_name,
                    "_id": str(i + 1),
                    "_source": {
                        "filename": file_path,
                        "content": content
                    }
                }
                actions.append(action)
            except UnicodeDecodeError:
                print(f"跳过非文本文件: {file_path}")
                continue
            except Exception as e:
                print(f"处理文件 {file_path} 时出错: {e}")
                continue
        
        # 批量索引
        from elasticsearch.helpers import bulk
        bulk(self.es, actions)

    def create_index(self):
        if not self.es.indices.exists(index=self.index_name):
            self.es.indices.create(index=self.index_name)

    def index_document(self, doc_id: str, document: dict):
        self.es.index(index=self.index_name, id=doc_id, document=document)

    def search(self, query: str, field: str, size: int = 10, from_: int = 0, fields: list = None):
        """
        搜索文档
        Args:
            query: 搜索关键词
            field: 搜索字段
            size: 返回结果数量
            from_: 分页起始位置
            fields: 指定返回的字段列表
        """
        body = {
            "query": {
                "match": {
                    field: query
                }
            },
            "size": size,
            "from": from_
        }
        
        if fields:
            body["_source"] = fields
            
        return self.es.search(index=self.index_name, body=body)

if __name__ == "__main__":
    # 示例用法
    es = ESSearch("chenhaiqing")
    es.create_index()
    
    # 索引文档
    es.index_files("/Users/chenhaiqing/Documents/GitHub/haiqingchq.github.io/_posts")
    
    # 搜索
    results = es.search(query="Minio", field="content", size=1)
    print(json.dumps(results.body, indent=2, ensure_ascii=False)) 
~~~



