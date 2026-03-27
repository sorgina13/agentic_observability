# From Zero to Hero: Building Agents with Microsoft Foundry and Agent Framework

## Create agents in Foundry

### Requirements

#### Login to Azure

```bash
az login
```

#### Environment setup

```bash
export RG="rg-minihack-test"
export LOCATION="eastus2" # one that supports hosted agents, e.g., northcentralus
export AGENTS_HOME="/mnt/c/Users/loreaa/VS_CODE/foundry_governance_from_zero_to_hero"
```

Move to `AGENTS_HOME`:
```bash
cd $AGENTS_HOME
```

#### Install resources

Before deploying the infra resources, check the file `infra/basic-setup.parameters.json` to set the location and resource names you want.

```bash
az group create --name $RG --location $LOCATION
# deployment with file parameters
az deployment group create --resource-group $RG --template-file infra/basic-setup.bicep --parameters @infra/basic-setup.parameters.json
```

Update env variables with outputs from deployment

```bash
# get vars from deployment output
export FOUNDRY_RESOURCE_NAME=$(az deployment group show --resource-group $RG --name basic-setup --query properties.outputs.accountName.value -o tsv)
export FOUNDRY_PROJECT_NAME=$(az deployment group show --resource-group $RG --name basic-setup --query properties.outputs.projectName.value -o tsv) 
export AZURE_AI_PROJECT_ENDPOINT="https://$FOUNDRY_RESOURCE_NAME.services.ai.azure.com/api/projects/$FOUNDRY_PROJECT_NAME"
export AZURE_AI_MODEL_DEPLOYMENT_NAME="gpt-4.1"  # or your deployment name
export 
```

Use the Application Insights connection string from the connected Application Insights resource. Keep the real value out of the repository and set it locally in your terminal or a non-committed `.env` file.

From portal:

- Create a `Grounding with bing` resource and connect to the Microsoft Foundry project (https://learn.microsoft.com/en-us/azure/ai-foundry/agents/how-to/tools/bing-tools?view=foundry&tabs=grounding-with-bing&pivots=python#prerequisites)

![alt text](images/bingconnectofoundry.png)

Export variable:

```bash
export BING_CONNECTION_NAME="GroundingBy2603u" 
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export BING_PROJECT_CONNECTION_ID="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.CognitiveServices/accounts/$FOUNDRY_RESOURCE_NAME/projects/$FOUNDRY_PROJECT_NAME/connections/$BING_CONNECTION_NAME"
```


### Create venv and install the Agent Framework packages

Use a single environment for the full repo workflow. The newest Microsoft Agent Framework packages (`260130`) conflict with the current `agent-dev-cli` and `azure-ai-agentserver-agentframework` dependencies used by the tracing and hosted-agent flows, so the root `requirements.txt` pins the newest compatible set.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --no-compile --no-cache-dir --default-timeout=1000 --retries 5 -r requirements.txt
pip list
```

### Create agents

Activate the environment:

```bash
source .venv/bin/activate
```

**Using Foundry SDK**

```bash
python agents-standalone/foundry/create_research_agent.py
python agents-standalone/foundry/create_writer_agent.py
python agents-standalone/foundry/create_reviewer_agent.py
```

These scripts create the agents `ResearcherAgent`, `WriterAgent`, and `ReviewerAgent`.

If you want to call the research agent through `agents-client/agent_client.py`, you must publish `ResearcherAgent` after creating it.

**Using Microsoft Agent Framework**

```bash
python agents-standalone/maf/create_research_agent.py
python agents-standalone/maf/create_writer_agent.py
python agents-standalone/maf/create_reviewer_agent.py
```

These scripts create the agents `ResearcherAgentV2`, `WriterAgentV2`, and `ReviewerAgentV2`.

If you want to call the research agent through `agents-client/agent_client.py`, you must publish `ResearcherAgentV2` after creating it.

### Publish the agent

Use publish in Foundry portal after the create script completes successfully.

This step is required before `agents-client/agent_client.py` will work. The client calls the Agent Application endpoint under `/applications/<agent-name>/protocols/openai`, and that endpoint does not exist until the agent is published as an Agent Application.

For the research agent, this means publishing `ResearcherAgent` if you created it with the Foundry SDK scripts, or publishing `ResearcherAgentV2` if you created it with the Microsoft Agent Framework scripts.

You get a set of endpoints for the Researcher agent (responses api and activity protocol):

### Test the agent

Use the responses endpoint to test the published agent. Set `AGENT_NAME` to match the creation script you used:

```bash
export AGENT_NAME=ResearcherAgent   # For agents-standalone/foundry/create_research_agent.py
# export AGENT_NAME=ResearcherAgentV2  # For agents-standalone/maf/create_research_agent.py
python agents-client/agent_client.py "What are the latest AI trends?"
```

If you get `Application '<agent-name>' not found`, one of these is still true:
- The create script never completed successfully.
- The agent was created but not published yet.
- `AGENT_NAME` does not match the published application name.

## Create workflow

Test the sequential agents workflow



```bash
python orchestration/demo/sequential_agents.py
```

Test the group chat agent workflow

```bash
python orchestration/demo/group_chat_agent_manager.py
```

## Build as Agent and trace the workflow locally with AI Toolkit Agent Inspector

Use the same environment created above:

```bash
source .venv/bin/activate
```

### Workflow as agent

First, we will adapt the workflow to become an agent. For that, we will use the `azure-ai-agentserver-agentframework` library to expose the workflow as agent. The relevant code is:

```python
      agentwf = workflow.as_agent()
      await from_agent_framework(agentwf).run_async()
```

### Instrument the agent

To trigger the workflow directly from the AI Toolkit Agent Inspector UI, run the sample through `agentdev`. In this mode, `agentdev` injects the conversation backend and tracing hooks that Agent Inspector expects.

Before starting `agentdev`, clear any tracing environment variables that would make the underlying agent server register its own exporters.

### Run and test locally with Agent Inspector

We will use the **AI Toolkit Agent Inspector** integrated in VS Code to run, test and visualize traces interactively.

#### Prerequisites

`debugpy` and `agent-dev-cli` are already installed from the root `requirements.txt` in the active environment.

```bash
pip show debugpy agent-dev-cli
```

#### Start the agent server from bash

Make sure all environment variables are exported in your terminal, then start the server with `.venv` active:

```bash
export PYDEVD_DISABLE_FILE_VALIDATION=1
python -Xfrozen_modules=off -m debugpy --listen 127.0.0.1:5679 -m agentdev run orchestration/tracing/sequential_agents_as_agent.py --verbose --port 8088
```

> **Note:** The `-Xfrozen_modules=off` flag is required to prevent the debugger from missing breakpoints. Without it, you may see the warning: _"It seems that frozen modules are being used, which may make the debugger miss breakpoints."_

Wait until you see `Application startup complete` in the terminal before proceeding.


### Group Chat Manager as agent

```bash
export PYDEVD_DISABLE_FILE_VALIDATION=1
python -Xfrozen_modules=off -m debugpy --listen 127.0.0.1:5679 -m agentdev run orchestration/tracing/group_chat_agent_manager_as_agent.py --verbose --port 8088
```

#### Open Agent Inspector in VS Code

- Click the **AI Toolkit** icon in the left Activity Bar
- Navigate to **Agent Inspector**
- It will automatically connect to `http://localhost:8088`

#### Send a test prompt

Send a prompt from the Agent Inspector UI input box.

#### View traces in Agent Inspector

The AI Toolkit Agent Inspector will display:
- Full multi-agent message flows between Researcher, Writer, and Reviewer
- LLM prompts and completions (with `enable_sensitive_data=True`)
- Timing and span hierarchy for each agent step

![Microsoft Foundry Local Traces](images/localtraces.png)

#### Stop the server

When done, press `Ctrl+C` in the bash terminal to stop the server.


## Deploy as hosted agent

### Observability configuration for Azure Monitor

To get traces in `Microsoft Foundry`, we need to connect our Foundry project to an Application Insights resource. The application insights resource has been already created in the infrastructure deployment step, so you just need to connect it to the Foundry project. To do that, go to the Foundry portal and navigate to `Operate/Admin/<choose project>/Connected Resources/Application Insights` and connect the Application Insights resource that was created in the infrastructure step:

![Application Insights Connected Resource](images/appinsightsconfig.png)

### Understand folder structure

In order to deploy the workflow as a hosted agent in Foundry, we will need to create several files under the agent's folder:

- the agent code: `orchestration/hosted/groupchat/group_chat_agent_manager_as_agent.py`
- a python file with the OpenTelemetry configuration for Azure Monitor: `orchestration/hosted/groupchat/observability.py`. This file will be used to configure the OpenTelemetry providers to send traces to Azure Monitor. We need this file because the configuration for Azure Monitor is different than the one for local tracing with AI Toolkit, so we need to separate the configuration and import the correct one depending on where we are running (locally with AI Toolkit or as hosted agent in Foundry).
- a `requirements.txt` file with the dependencies
- a `Dockerfile` to build the container image
- a .env file with environment variables that are then injected into the container. For this demo, the required variables are:
    ```
    AZURE_AI_PROJECT_ENDPOINT=
    AZURE_AI_MODEL_DEPLOYMENT_NAME=
    ```

Also, we need to create a `.foundry/.deployment.json` file to define the hosted agent deployment options. The Microsoft Foundry extension will look for this file to know how to build and deploy the hosted agent. The content of the file would be generated for you if you use the extension to deploy, but there is a limitation that it doesn't generate the correct dockerContextPath if your Dockerfile is not in the root of the project, so make sure to update those paths to point to the `orchestration/hosted/groupchat` folder:

```json
{
  "hostedAgentDeployOptions": {
    "dockerContextPath": "/workspaces/agents-observability-tt202/from-zero-to-hero/orchestration/hosted/groupchat",
    "dockerfilePath": "/workspaces/agents-observability-tt202/from-zero-to-hero/orchestration/hosted/groupchat/Dockerfile",
    "agentName": "groupchatwriter",
    "cpu": "1.0",
    "memory": "2.0Gi"
  }
}
```

You can try without this file and you will be asked to fill in the deployment options in the Microsoft Foundry extension UI when you click on Deploy, but the final deployment will fail as the context is just the root of the project and not the folder where the Dockerfile is (that is defult behavior of the extension).

To avoid this, you can copy the content from `orchestration/hosted/groupchat/.foundry/.deployment.json` to the root `.foundry/.deployment.json` before deploying, or just update the paths in the existing root `.foundry/.deployment.json` to point to the correct Dockerfile and context.

### Deploy

In the Local Agent Playground from the Microsoft Foundry extension, click on `Deploy` and select the folder `orchestration/hosted/groupchat`. This will build the container image and deploy it as a hosted agent in Foundry:

![Deploy hosted agent](images/deployhosted.png)


It takes a few minutes to build and deploy the agent. Once it's deployed, you can see it in the Foundry portal under Agents.

**Important:** Before testing, we need to give permission to the Foundry Project Managed Identity. Use the portal to give "Azure AI User" role over the Foundry project.

You can now test the hosted agent from the portal or even better, from the Hosted Agent Playground in the Microsoft Foundry extension, select the `groupchatwriter` or `sequentialwriter` agent and version to finally test it with a prompt:

![Hosted Agent Playground](images/hostedextensionplayground.png)


Optionally, you can also test it using the responses endpoint as before, just changing the AGENT_NAME to the name of the hosted agent. Remember that you must publish the hosted agent in Foundry portal first.


```bash
export AGENT_NAME=groupchatwriter
python agents-client/agent_client.py "Write a short article about the latest AI trends."
```

### Explore traces in Microsoft Foundry

Under the `Traces` tab, click on the `Trace ID` and you would see a similar output to this:

![Traces in Foundry](images/foundrytraces.png)

You can also explore the metrics directly from Application Insights:

![Metrics in App Insights](images/appinsights-metrics.png)
