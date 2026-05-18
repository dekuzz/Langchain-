---
date: 2026-05-18
aliases:
  - my first note about RAG
tags:
  - 基础流水线
  - langchain
---
# 基础流水线

> [!NOTE] 大致流程
> 文档加载-文本块切分-向量化存储-检索-提示词-大模型-生成回答
## 数据流程

> [!NOTE] 数据格式走向
>RawFile/Str - List[Document]->List[Document]-Embedding Model->List[Vector()]+Metadata字典->Vectorstore-Vectorestore.as_retriever->Top_k List->Prompt Template->Formatted Prompt-LLM->final answer(str)

## 代码实战
### 文档加载
``` python
from langchain_community.document_loaders import PyPDFLoader
loader = PyPDFLoader("data/2024全球经济金融展望报告.pdf")  
documents = loader.load()
```
### 文档切分
``` python
from langchain_text_splitters import RecursiveCharacterTextSplitter
splitter = RecursiveCharacterTextSplitter(  
    chunk_size=chunk_size,  
    chunk_overlap=chunk_overlap,  
    separators=separators  
)  
split_docs = splitter.split_documents(documents)  
for chunk in split_docs:  
    chunk.metadata['uuid'] = str(uuid4())

```
### 向量化
``` python
from langchain_community.embeddings import HuggingFaceBgeEmbeddings  
import torch  
  
device = 'cuda' if torch.cuda.is_available() else 'cpu'  
print(f'device: {device}')  
  
embeddings = HuggingFaceBgeEmbeddings(  
    model_name=EMBEDDING_MODEL_PATH,  
    model_kwargs={'device': device},  
    encode_kwargs={'normalize_embeddings': True}  
)
vector_db = Chroma(  
    persist_directory=store_path,  
    embedding_function=embeddings  
)
```
# 计算检索准确率
使用预先加载的qa问答对数据集检测检索工具的准确率
``` python
def retrieve(vector_db, query: str, k=5):  
    return vector_db.similarity_search(query, k=k)
```
``` python
test_df = qa_df[(qa_df['dataset'] == 'test') & (qa_df['qa_type'] == 'detailed')]
top_k_arr = list(range(1, 9))  
hit_stat_data = []  
  
for idx, row in tqdm(test_df.iterrows(), total=len(test_df)):  
    question = row['question']  
    true_uuid = row['uuid']  
    chunks = retrieve(vector_db, question, k=max(top_k_arr))  
    retrieved_uuids = [doc.metadata['uuid'] for doc in chunks]  
  
    for k in top_k_arr:  
        hit_stat_data.append({  
            'question': question,  
            'top_k': k,  
            'hit': int(true_uuid in retrieved_uuids[:k])  
        })
hit_stat_df = pd.DataFrame(hit_stat_data)
hit_stat_df.groupby('top_k')['hit'].mean().reset_index()
```
``` python
import seaborn as sns
sns.barplot(x='top_k', y='hit', data=hit_stat_df.groupby('top_k')['hit'].mean().reset_index())

```
