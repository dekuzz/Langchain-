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
### LCEL处理问答系统
```python
from langchain_community.llms import Ollama  
from langchain_core.output_parsers import StrOutputParser  
from langchain_core.runnables import RunnablePassthrough  
from langchain_core.prompts import PromptTemplate  
  
def format_docs(docs):  
    return "\n\n".join(doc.page_content for doc in docs)  
  
llm = Ollama(  
    model='deepseek-r1:8b',  
    base_url="http://localhost:11434"  
)  
  
prompt_tmpl = """  
你是一个金融分析师，擅长根据所获取的信息片段，对问题进行分析和推理。  
你的任务是根据所获取的信息片段（<<<<context>>><<<</context>>>之间的内容）回答问题。  
回答保持简洁，不必重复问题，不要要添加描述性解释和与答案无关的任何内容。  
已知信息：  
<<<<context>>>  
{context}  
<<<</context>>>  
  
问题：{question}  
请回答：  
"""  
prompt = PromptTemplate.from_template(prompt_tmpl)  
retriever = vector_db.as_retriever(search_kwargs={'k': 4})  
  
rag_chain = (  
    {"context": retriever | format_docs, "question": RunnablePassthrough()}  
    | prompt  
    | llm  
    | StrOutputParser()  
)  
#流式输出
for chunk in rag_chain.stream("2023年10月美国ISM制造业PMI指数较上月有何变化？"):  
    print(chunk, end="", flush=True)
#非流式输出
print(rag_chain.invoke('2023年10月美国ISM制造业PMI指数较上月有何变化？'))

```