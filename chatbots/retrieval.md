
```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.runnables import RunnableBranch
from langchain_community.document_loaders import PDFPlumberLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain_openai import ChatOpenAI
import dotenv
from langchain.chains.combine_documents import create_stuff_documents_chain

dotenv.load_dotenv()


#### 1. load PDF file 
loader = PDFPlumberLoader("test.pdf")
data = loader.load()


#### 2. splitter 
text_splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=0)
all_splits = text_splitter.split_documents(data)

### 3. stroe vector store
vectorstore = Chroma.from_documents(documents=all_splits, embedding=OpenAIEmbeddings())

### 4. creat a retriever 
retriever = vectorstore.as_retriever(k=4)
docs = retriever.invoke("Can I know about MBTI?")

```

```python
#### 5. declare chatbot
chat = ChatOpenAI(model="gpt-3.5-turbo-1106", temperature=0.2)

#### 6. declare prompt 
question_answering_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "Answer the user's questions based on the below context:\n\n{context}",
        ),
        MessagesPlaceholder(variable_name="messages"),
    ]
)


##### 7. declare prompt for making a question
query_transform_prompt = ChatPromptTemplate.from_messages(
    [
        MessagesPlaceholder(variable_name="messages"),
        (
            "user",
            "Given the above conversation, generate a search query to look up in order to get information relevant to the conversation. Only respond with the query, nothing else.",
        ),
    ]
)
```

```python
from langchain.memory import ChatMessageHistory
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableBranch


### 8. declare a quenstion chain
query_transforming_retriever_chain = RunnableBranch(
    (
        lambda x: len(x.get("messages", [])) == 1,
        # If only one message, then we just pass that message's content to retriever
        (lambda x: x["messages"][-1].content) | retriever,
    ),
    # If messages, then we pass inputs to LLM chain to transform the query, then pass to retriever
    query_transform_prompt | chat | StrOutputParser() | retriever,
).with_config(run_name="chat_retriever_chain")


### 9. declare a answering chain
document_chain = create_stuff_documents_chain(chat, question_answering_prompt)

 
### 10. make a history
demo_ephemeral_chat_history = ChatMessageHistory()
```

```python
from langchain_core.runnables import RunnablePassthrough

demo_ephemeral_chat_history.add_user_message("What is ISTP ?")

#### 11. connection with question and awnswering 
conversational_retrieval_chain = RunnablePassthrough.assign(
    context=query_transforming_retriever_chain,
).assign(
    answer=document_chain,
)

#### 12. process for lunching
stream = conversational_retrieval_chain.stream(
    {"messages": demo_ephemeral_chat_history.messages},
) 

#### 13. streaming results
for chunk in stream: 
     print(chunk)


```