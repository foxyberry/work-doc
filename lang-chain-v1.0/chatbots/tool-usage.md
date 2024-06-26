1. Define a tool, chat, prompt, agent history 

```python
import dotenv
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.memory import ChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

dotenv.load_dotenv()


### Define a tool
tools = [TavilySearchResults(max_results=1)]

# Choose the LLM that will drive the agent
# Only certain models support this

### Define a chat
chat = ChatOpenAI(model="gpt-3.5-turbo-1106", temperature=0)

### Define a prompt
prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "You are a helpful assistant. You may not need to use tools for every query - the user may just want to chat!",
        ),
        MessagesPlaceholder(variable_name="chat_history"),
        ("human", "{input}"),
        MessagesPlaceholder(variable_name="agent_scratchpad"),
    ]
)


### Define a agent
agent = create_openai_tools_agent(chat, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

#### Define history
demo_ephemeral_chat_history_for_chain = ChatMessageHistory()

conversational_agent_executor = RunnableWithMessageHistory(
    agent_executor,
    lambda session_id: demo_ephemeral_chat_history_for_chain,
    input_messages_key="input",
    output_messages_key="output",
    history_messages_key="chat_history",
)

```

2. run the executor

```
conversational_agent_executor.invoke(
    {
        "input": "I'm Nemo!",
    },
    {"configurable": {"session_id": "unused"}},
)

conversational_agent_executor.invoke(
    {
        "input": "What is the current conservation status of the Great Barrier Reef?",
    },
    {"configurable": {"session_id": "nature"}},
)
```

```
conversational_agent_executor.invoke(
    {
        "input": "What is my name?",
    },
    {"configurable": {"session_id": "unused"}},
)


conversational_agent_executor.invoke(
    {
        "input": "Could you summarize my quenstion and your answer?",
    },
    {"configurable": {"session_id": "nature"}},
)


conversational_agent_executor.invoke(
    {
        "input": "you know what my name is?",
    },
    {"configurable": {"session_id": "nature"}},
)
```