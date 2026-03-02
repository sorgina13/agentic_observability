# From Zero to Hero: Building Agents with Microsoft Foundry and Agent Framework

## Create agents in Foundry

### Requirements

#### Login to Azure

```bash
az login
```

#### Environment setup

```bash
export RG="rg-governance_ai"
export LOCATION="eastus2" # one that supports hosted agents, e.g., northcentralus
export AGENTS_HOME="/VS_CODE/agents-observability-tt202/from-zero-to-hero"
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
```

From portal:

- Create a `Grounding with bing` resource and connect to the Microsoft Foundry project (https://learn.microsoft.com/en-us/azure/ai-foundry/agents/how-to/tools/bing-tools?view=foundry&tabs=grounding-with-bing&pivots=python#prerequisites)

![alt text](images/bingconnectofoundry.png)

Export variable:

```bash
export BING_CONNECTION_NAME="GroundingB" 
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export BING_PROJECT_CONNECTION_ID="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.CognitiveServices/accounts/$FOUNDRY_RESOURCE_NAME/projects/$FOUNDRY_PROJECT_NAME/connections/$BING_CONNECTION_NAME"
```


### Create venv and install the Agent Framework packages

As of Feb 4, 2026, I will create two venvs:
- venv260130 for latest MAF packages (260130)
- venv260107 for previous MAF packages (260107) and compatible with azure-ai-agentserver-agentframework 1.0.0b10

```bash
python3 -m venv venv260130
source venv260130/bin/activate
pip install --no-compile -r requirements-260130.txt
pip install --no-compile --no-cache-dir --default-timeout=1000 --retries 5 -r 
pip list
deactivate
python3 -m venv venv260107
source venv260107/bin/activate
pip install --no-compile --no-cache-dir --default-timeout=1000 --retries 5 -r 
pip list
deactivate

pip install azure-ai-agentserver-agentframework --no-compile --no-cache-dir --default-timeout=1000 --retries 5 

pip install --no-compile --pre agent-dev-cli 

```

### Create agents

Activate latest venv:

```bash
source venv260130/bin/activate
```

**Using Foundry SDK**

```bash
python agents-standalone/foundry/create_research_agent.py
python agents-standalone/foundry/create_writer_agent.py
python agents-standalone/foundry/create_reviewer_agent.py
```

**Using Microsoft Agent Framework**

```bash
python agents-standalone/maf/create_research_agent.py
python agents-standalone/maf/create_writer_agent.py
python agents-standalone/maf/create_reviewer_agent.py
```

### Publish the agent

Use publish in Foundry portal. 

You get a set of endpoints for the Researcher agent (responses api and activity protocol):

### Test the agent

Use the responses endpoint to test the agent:

```bash
export AGENT_NAME=ResearcherAgentV2
python agents-client/agent_client.py "What are the latest AI trends?"
```

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

As per today (Feb 4, 2026), we have to use the previous venv (260107) to build the orchestration as an agent.

Activate the previous venv:

```bash
deactivate
source venv260107/bin/activate
```

### Workflow as agent

First, we will adapt the workflow to become an agent. For that, we will use the `azure-ai-agentserver-agentframework` library to expose the workflow as agent. The relevant code is:

```python
      agentwf = workflow.as_agent()
      await from_agent_framework(agentwf).run_async()
```

### Instrument the agent

The agents under `orchestration/tracing/` are already instrumented with OpenTelemetry via the Agent Framework's built-in `configure_otel_providers()`, which sends traces to the AI Toolkit Agent Inspector on port `4317` (gRPC):

```python
from agent_framework.observability import configure_otel_providers

configure_otel_providers(
    vs_code_extension_port=4319,  # AI Toolkit gRPC port
    enable_sensitive_data=True    # Capture prompts and completions
)
```

This is configured in both:
- `orchestration/tracing/group_chat_agent_manager_as_agent.py`
- `orchestration/tracing/sequential_agents_as_agent.py`

### Run and test locally with Agent Inspector

We will use the **AI Toolkit Agent Inspector** integrated in VS Code to run, test and visualize traces interactively.

#### Prerequisites

Make sure `debugpy` and `agent-dev-cli` are installed in your active venv:

```bash
pip install --no-compile debugpy
pip install --no-compile --pre agent-dev-cli
```

#### Start the agent server from bash

Make sure all environment variables are exported in your terminal, then start the server with `venv260107` active:

```bash
# For Group Chat workflow:
python -Xfrozen_modules=off -m debugpy --listen 127.0.0.1:5679 -m agentdev run orchestration/tracing/group_chat_agent_manager_as_agent.py --verbose --port 8087

# OR for Sequential workflow:
python -Xfrozen_modules=off -m debugpy --listen 127.0.0.1:5679 -m agentdev run orchestration/tracing/sequential_agents_as_agent.py --verbose --port 8087
```

> **Note:** The `-Xfrozen_modules=off` flag is required to prevent the debugger from missing breakpoints. Without it, you may see the warning: _"It seems that frozen modules are being used, which may make the debugger miss breakpoints."_

Wait until you see `Application startup complete` in the terminal before proceeding.

#### Open Agent Inspector in VS Code

- Click the **AI Toolkit** icon in the left Activity Bar
- Navigate to **Agent Inspector**
- It will automatically connect to `http://localhost:8087`

#### Send a test prompt

Send a prompt from the Agent Inspector UI input box, or using curl:

```bash
curl -X POST http://localhost:8087/responses \
  -H "Content-Type: application/json" \
  -d '{
    "input": "Report about the latest AI trends."
}'
```

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
