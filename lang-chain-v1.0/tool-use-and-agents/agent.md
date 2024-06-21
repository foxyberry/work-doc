
# Tool use and agents

- When using LLMs, you can build natural language interfaces with various tools such as APIs, functions, and databases. 
Tool  : https://python.langchain.com/v0.1/docs/integrations/tools/
These tools can perform API calls, function executions, or database queries.  


- agent :  in situations where it is unclear how many and which tools need to be called to reach a conclusion, agents can autonomously perform these tasks.



```python
import getpass
import os
from langchain_core.tools import tool
import dotenv
from langchain_openai import ChatOpenAI
from langchain import hub
from langchain.agents import AgentExecutor, create_tool_calling_agent

@tool
def multiply(first_int: int, second_int: int) -> int: 
    """ Multiply two integers together."""
    return first_int * second_int

@tool
def add(first_int: int, second_int: int) -> int:
    "Add two integers."
    return first_int + second_int

@tool
def exponentiate(base: int, exponent: int) -> int:
    "Exponentiate the base to the exponent power."
    return base**exponent

dotenv.load_dotenv()
llm = ChatOpenAI(model="gpt-3.5-turbo-0125")
llm_with_tools = llm.bind_tools([multiply])

prompt = hub.pull("hwchase17/openai-tools-agent")
prompt.pretty_print()

tools = [multiply, add, exponentiate]

agent = create_tool_calling_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

agent_executor.invoke(
    {
        "input" : "Take 3 to the fifth power and multiply that by the sum of twelve and three, then square the while result"
    }    
)

```
