1. file load -> splitter -> store to vector DB 

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableBranch
import dotenv
from langchain_community.document_loaders import PDFPlumberLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings


#### 1. load PDF file 
loader = PDFPlumberLoader("test.pdf")
data = loader.load()


#### 2. splitter 
text_splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=0)
all_splits = text_splitter.split_documents(data)

### 3. store vector store
vectorstore = Chroma.from_documents(documents=all_splits, embedding=OpenAIEmbeddings())

### 4. create a retriever
retriever = vectorstore.as_retriever(k=4)
docs = retriever.invoke("how can langsmith help with testing?")
```

2. load env for API KEY
```python
dotenv.load_dotenv()
```

3. declare a chatbot and prompts 
```python
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

#### 5. declare a ChatBot
chat = ChatOpenAI(model="gpt-3.5-turbo-1106", temperature=0.2)

#### 6. declare a prompt for question and answer
question_answering_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "Answer the user's questions based on the below context:\n\n{context}",
        ),
        MessagesPlaceholder(variable_name="messages"),
    ]
)


##### 7. declare a prompt for generating a search query
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


3. declare chains
```python
from langchain.memory import ChatMessageHistory
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableBranch


### 8. define a chain for question
query_transforming_retriever_chain = RunnableBranch(
    (
        lambda x: len(x.get("messages", [])) == 1,
        # If only one message, then we just pass that message's content to retriever
        (lambda x: x["messages"][-1].content) | retriever,
    ),
    # If messages, then we pass inputs to LLM chain to transform the query, then pass to the retriever
    query_transform_prompt | chat | StrOutputParser() | retriever,
).with_config(run_name="chat_retriever_chain")


### 9. define a chain for answering
document_chain = create_stuff_documents_chain(chat, question_answering_prompt)

 
### 10. create a history
demo_ephemeral_chat_history = ChatMessageHistory()
```

4. chatHistory
```python
demo_ephemeral_chat_history.add_user_message("what is MBTI")

class ModifiedJSONAgentOutputParser(JSONAgentOutputParser):
    def __call__(self, text: str) -> str:
        parsed_output = super().__call__(text)
        return parsed_output['action_input']
        

#### 11. connect with chains for question and answer
conversational_retrieval_chain = RunnablePassthrough.assign(
    context=query_transforming_retriever_chain,
).assign(
    answer=document_chain,
)

#### 12. run 
response = conversational_retrieval_chain.invoke(
    {"messages": demo_ephemeral_chat_history.messages},
) 

demo_ephemeral_chat_history.add_ai_message(response["answer"])
```



5. Output 
```python
from langchain.schema import HumanMessage, SystemMessage, AIMessage

print("<< Message Contents >> \n")
for message in response['messages']:
    if isinstance(message, HumanMessage):
        print("HumanMessage:", message.content)
    elif isinstance(message, AIMessage):
        print("AIMessage:", message.content)
        print("\n\n")


print("\n\n<< MetaData >> \n")
for document in response['context']:
    file_path = document.metadata['file_path']
    page = document.metadata['page']
    page_content = document.page_content
    print(f" - file name : {file_path}, page : {page}")
    print(f" - contents : {page_content}")
    print(f"\n\n")


print("\n\n<< Final Answer >> \n")
print(f" - Answer : {response['answer']}")
```
