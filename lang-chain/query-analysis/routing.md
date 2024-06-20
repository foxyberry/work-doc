### Routing
LangChain's routing functionality is a technique that utilizes various indexes across multiple domains to select the most appropriate index for a given query. 


```python
import dotenv
from typing import Literal
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.pydantic_v1 import BaseModel, Field
from langchain_openai import ChatOpenAI

class RouteQuery(BaseModel):
    """Route a user query to the most relevant datasource."""

    datasource: Literal["python_docs", "js_docs", "golang_docs"] = Field(
        ...,
        description="Given a user question choose which datasource would be most relevant for answering their question",
    )


dotenv.load_dotenv()

llm = ChatOpenAI(model="gpt-3.5-turbo-0125", temperature=0)
structured_llm = llm.with_structured_output(RouteQuery)

system = """You are an expert at routing a user question to the appropriate data source.

Based on the programming language the question is referring to, route it to the relevant data source."""
prompt = ChatPromptTemplate.from_messages(
    [
        ("system", system),
        ("human", "{question}"),
    ]
)

router = prompt | structured_llm
```


```python
question = """Why doesn't the following code work:

from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages(["human", "speak in {language}"])
prompt.invoke("french")
"""
router.invoke({"question": question})
```


```python

question = """Why doesn't the following code work:


import { ChatPromptTemplate } from "@langchain/core/prompts";


const chatPrompt = ChatPromptTemplate.fromMessages([
  ["human", "speak in {language}"],
]);

const formattedChatPrompt = await chatPrompt.invoke({
  input_language: "french"
});
"""
router.invoke({"question": question})
```

```python
from typing import List


class RouteQuery(BaseModel):
    """Route a user query to the most relevant datasource."""

    datasources: List[Literal["python_docs", "js_docs", "golang_docs"]] = Field(
        ...,
        description="Given a user question choose which datasources would be most relevant for answering their question",
    )


llm = ChatOpenAI(model="gpt-3.5-turbo-0125", temperature=0)
structured_llm = llm.with_structured_output(RouteQuery)
router = prompt | structured_llm
router.invoke(
    {
        "question": "is there feature parity between the Python and JS implementations of OpenAI chat models"
    }
)
```