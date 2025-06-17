sk:
# Copyright (c) Microsoft. All rights reserved.

# --- Imports ---
import asyncio
import json
import os
from azure.identity import AzureCliCredential  # For Azure AD authentication
from semantic_kernel.agents import ChatCompletionAgent  # SK agent for chat completions
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion  # Azure OpenAI connector
from performance_evaluator import PerformanceLogger, is_valid_html
import time

# --- Config ---
DICTIONARY_FILE = os.path.join(os.path.dirname(os.path.dirname(os.path.abspath(__file__))), "dictionaries/05dictionary.json")
INSTRUCTIONS_FILE = os.path.join(os.path.dirname(os.path.dirname(os.path.abspath(__file__))), "sk+sk/main_instructions.txt")

# --- Azure OpenAI Auth Setup ---
ENDPOINT = "https://pf-t217-openai-use1.openai.azure.com/"
API_VERSION = "2024-02-15-preview"
DEPLOYMENT_NAME = "gpt-4-turbo-2024-04-09"

# Get Azure AD token using AzureCliCredential
credential = AzureCliCredential()
access_token = credential.get_token("https://cognitiveservices.azure.com/.default").token

# --- Load Parameter Dictionary ---
with open(DICTIONARY_FILE, "r") as f:
    PARAMETER_DICTIONARY = json.load(f)

# --- Load Codegen Instructions ---
with open(INSTRUCTIONS_FILE, "r") as f:
    CODEGEN_INSTRUCTIONS = f.read()

# --- Code Generation Function ---
async def generate_code_with_sk(agent, prompt) -> str:
    response = await agent.get_response(messages=prompt)
    return str(response)

async def agentic_code_workflow():
    max_attempts = 3
    attempt = 0
    css = 0.0
    final_code = None
    metrics = {}
    while attempt < max_attempts and css < 1.0:
        # Agent 1: Code Generator
        codegen_agent = ChatCompletionAgent(
            service=AzureChatCompletion(
                deployment_name=DEPLOYMENT_NAME,
                endpoint=ENDPOINT,
                api_key=access_token,
                api_version=API_VERSION
            ),
            name="CodeGenerator",
            instructions=CODEGEN_INSTRUCTIONS + "\n\nHere is the parameter dictionary as JSON:\n" + json.dumps(PARAMETER_DICTIONARY, indent=2),
        )

        # Agent 2: Reviewer
        review_instructions = (
            "You are a code reviewer. Carefully review the following code draft for correctness, completeness, and best practices. "
            "Suggest improvements or corrections. If the code is correct, say so, but still suggest any improvements."
        )
        review_agent = ChatCompletionAgent(
            service=AzureChatCompletion(
                deployment_name=DEPLOYMENT_NAME,
                endpoint=ENDPOINT,
                api_key=access_token,
                api_version=API_VERSION
            ),
            name="CodeReviewer",
            instructions=review_instructions,
        )

        perf_logger = PerformanceLogger()

        # Step 1: Agent 1 generates draft
        t1 = time.time()
        prompt = "Generate VueBootstrapNext code for this dictionary."
        draft = await generate_code_with_sk(codegen_agent, prompt)
        t2 = time.time()
        agent1_time = t2 - t1

        # Step 2: Agent 2 revises the draft and outputs improved code
        t3 = time.time()
        review_prompt = (
            "Revise and improve the following VueBootstrapNext code draft. "
            "Output the improved code only, with corrections and enhancements:\n\n"
            f"{draft}"
        )
        revised_draft = await generate_code_with_sk(review_agent, review_prompt)
        t4 = time.time()
        agent2_time = t4 - t3

        # Step 3: Agent 1 reviews Agent 2's revision and outputs the final code
        t5 = time.time()
        final_prompt = (
            "Here is the revised code draft from the reviewer:\n"
            f"{revised_draft}\n\n"
            "Review and further improve this code. Output the final improved code only."
        )
        final_code = await generate_code_with_sk(codegen_agent, final_prompt)
        t6 = time.time()
        agent1_review_time = t6 - t5

        # Example error detection logic (customize as needed)
        agent1_detected_agent2_errors = not is_valid_html(revised_draft)
        agent2_detected_errors = not is_valid_html(draft)

        perf_logger.log_case(
            case_id=f"test{attempt+1}",
            agent1_html=draft,
            agent2_html=revised_draft,
            final_html=final_code,
            agent1_time=agent1_time,
            agent2_time=agent2_time,
            agent1_review_time=agent1_review_time,
            agent1_errors=agent1_detected_agent2_errors,
            agent2_errors=agent2_detected_errors
        )

        metrics = perf_logger.compute_metrics()
        print(metrics)
        css = metrics.get("CSS", 0.0)
        attempt += 1
        if css >= 1.0:
            break
        print(f"Attempt {attempt}: CSS < 1.0, repeating workflow...")

    return final_code

# --- Main Entrypoint ---
async def main():
    final_code = await agentic_code_workflow()
    print(final_code)

# --- Script Entrypoint ---
if __name__ == "__main__":
    asyncio.run(main())






sk2:
# Copyright (c) Microsoft. All rights reserved.

# --- Imports ---
import asyncio
import json
import os
from azure.identity import AzureCliCredential  # For Azure AD authentication
from semantic_kernel.agents import ChatCompletionAgent  # SK agent for chat completions
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion  # Azure OpenAI connector

# --- Config ---
DICTIONARY_FILE = os.path.join(os.path.dirname(os.path.dirname(os.path.abspath(__file__))), "dictionaries/05dictionary.json")
INSTRUCTIONS_FILE = os.path.join(os.path.dirname(os.path.dirname(os.path.abspath(__file__))), "sk+sk/reviewer_instructions.txt")

# --- Azure OpenAI Auth Setup ---
ENDPOINT = "https://pf-t217-openai-use1.openai.azure.com/"
API_VERSION = "2024-02-15-preview"
DEPLOYMENT_NAME = "gpt-4-turbo-2024-04-09"

# Get Azure AD token using AzureCliCredential
credential = AzureCliCredential()
access_token = credential.get_token("https://cognitiveservices.azure.com/.default").token

# --- Load Parameter Dictionary ---
with open(DICTIONARY_FILE, "r") as f:
    PARAMETER_DICTIONARY = json.load(f)

# --- Load Codegen Instructions ---
with open(INSTRUCTIONS_FILE, "r") as f:
    CODEGEN_INSTRUCTIONS = f.read()

# --- Code Generation Function ---
async def generate_code_with_sk() -> str:
    agent = ChatCompletionAgent(
        service=AzureChatCompletion(
            deployment_name=DEPLOYMENT_NAME,
            endpoint=ENDPOINT,
            api_key=access_token,  # Use the Azure AD token as the API key
            api_version=API_VERSION
        ),
        name="CodeGenerator",
        instructions=CODEGEN_INSTRUCTIONS + "\n\nHere is the parameter dictionary as JSON:\n" + json.dumps(PARAMETER_DICTIONARY, indent=2),
    )
    # The prompt is always: generate VueBootstrapNext code for this dictionary
    prompt = "Generate VueBootstrapNext code for this dictionary."
    response = await agent.get_response(messages=prompt)
    return str(response)

# --- Main Entrypoint ---
async def main():
    # Always generate code for the loaded dictionary
    generated_code = await generate_code_with_sk()
    print(generated_code)

# --- Script Entrypoint ---
if __name__ == "__main__":
    asyncio.run(main())









sK
