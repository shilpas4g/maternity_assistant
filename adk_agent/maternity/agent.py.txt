import os
import logging
import google.cloud.logging
import dotenv
from maternity import tools
from google.adk.agents import Agent,LlmAgent
from google.adk.agents import SequentialAgent
from google.adk.tools.tool_context import ToolContext
from google.adk.tools.langchain_tool import LangchainTool

from langchain_community.tools import WikipediaQueryRun
from langchain_community.utilities import WikipediaAPIWrapper

import google.auth
import google.auth.transport.requests
import google.oauth2.id_token

cloud_logging_client = google.cloud.logging.Client()
cloud_logging_client.setup_logging()


#model_name = os.getenv("MODEL")

dotenv.load_dotenv()

PROJECT_ID = os.getenv("GOOGLE_CLOUD_PROJECT", "default-project")


maps_toolset = tools.get_maps_mcp_toolset()
bigquery_toolset = tools.get_bigquery_mcp_toolset()

# Greet user and save their prompt

def add_prompt_to_state(
    tool_context: ToolContext, prompt: str
) -> dict[str, str]:
    """Saves the user's initial prompt to the state."""
    tool_context.state["PROMPT"] = prompt
    tool_context.state["PROJECT_ID"] = PROJECT_ID
    #project_ref=tool_context.state.get("project_id", "default-project")
    logging.info(f"[State updated] Added to PROMPT: {prompt}")
    return {"status": "success"}

# Configuring the Wikipedia Tool
wikipedia_tool = LangchainTool(
    tool=WikipediaQueryRun(api_wrapper=WikipediaAPIWrapper())
)




comprehensive_agent = LlmAgent(
    model='gemini-2.5-flash',
    name='comprehensive_agent',
    description="The primary researcher that can access both internal maternity data and external knowledge from Wikipedia",
    instruction="""
                You are a helpful research assistant. Your goal is to fully answer the user's PROMPT.
    You have access to three tools:
    1.  **BigQuery toolset:** Access zip_code,city,neighborhood,nurse_name,pay_per_month,avaliability,age from demographics table and nurse_name,services from services tables from maternity_info big query dataset to extract info about individual nurses .Do  not use any other dataset.Run all query jobs from the project  mini-single-agent.
    2.  **Maps Toolset:** Use this for real-world location analysis.Include a hyperlink to an interactive map in your response where appropriate.
	3. A tool for searching Wikipedia for general knowledge (health monitoring , scheduling , diet, exercises etc )

    First, analyze the user's PROMPT.
    - If the prompt can be answered by only one tool, use that tool.
    - If the prompt is complex and requires information from all tools Bigquery , Maps AND Wikipedia,
        you MUST use all tools to gather all necessary information.
    - Synthesize the results from the tool(s) you use into preliminary data outputs.

    PROMPT:
    { PROMPT }      
            """,
    tools=[maps_toolset,bigquery_toolset,wikipedia_tool]
    #output_key="research_data"
)

# response_formatter = Agent(
#     name="response_formatter",
#     model="gemini-2.5-flash",
#     description="Synthesizes all information into a friendly, readable response.",
#     instruction="""	You are the friendly voice assitant. Your task is to take the
#     RESEARCH_DATA and present it to the user in a complete and helpful answer.

#     - First, present the specific information from the Bigquery and maps (like  zip_code,city,neighborhood,nurse_name,pay_per_month,avaliability,age from demographics table and nurse_name,services )
#     - Then, add the interesting general facts from the research.
#     - If some information is missing, just present the information you have.
#     - Be conversational and engaging.

#     RESEARCH_DATA:
#     { research_data }"""
    
#     )

execution_workflow = SequentialAgent(
    name="execution_workflow",
    description="The main workflow for handling a user's request",
    sub_agents=[
        comprehensive_agent
        #response_formatter      
    ]
    )


root_agent = LlmAgent(
    name="maternity_assistant",
    model="gemini-2.5-flash",
    description="Entry point to get the assistant",
    instruction="""	Greet the user .
	-Let the user know you will helping them learn about the   maternity care and baby care services .
    When the user responds, use the 'add_prompt_to_state' tool to save their response.
    After using the tool, transfer control to the 'execution_workflow' subagent.
    """,
    tools=[add_prompt_to_state],
    sub_agents=[execution_workflow]
     )
