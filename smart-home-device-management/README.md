# Smart Home Device Management — 3-Agent Architecture

A completed implementation of the Module 1 multi-agent architecture exercise. The system uses three specialized Strands agents, backed by Amazon Bedrock, to monitor smart-home device readings, diagnose predefined issues, and send simulated corrective commands.

## Architecture

![Smart Home Agent Architecture](resources/architecture.svg)

The coordinator function `run_device_pipeline()` orchestrates the workflow:

1. **Device Monitor Agent** calls `read_sensor_data()` and returns sensor data as JSON.
2. **Diagnostics Agent** passes the sensor JSON to `diagnose_issue()` and returns detected issues as JSON.
3. **Command Agent** calls `send_device_command()` for each detected issue and returns a command confirmation as JSON.

Each agent has its own `Agent` instance, system prompt, Bedrock model configuration, and tool. The coordinator passes the output from one stage to the next.

## Project structure

```text
smart-home-device-management/
├── README.md
├── .env.example
├── smart_home_device_mgmt.py
└── resources/
    ├── architecture.svg
    ├── smart_home_agent_output1.png
    └── smart_home_agent_output2.png
```

The two terminal screenshots are referenced below. Add your captured terminal images to `resources/` using those exact filenames.

## Requirements

- Python 3.11 or newer
- An AWS account or course lab account with access to Amazon Bedrock
- A Bedrock model/inference profile enabled for the selected AWS Region
- Strands Agents SDK
- AWS credentials available to the Python process

## Setup

Open a terminal in the project folder.

### 1. Create and activate a virtual environment (optional)

Linux/macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Windows PowerShell:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

### 2. Install dependencies

```bash
python -m pip install --upgrade pip
python -m pip install strands-agents boto3 python-dotenv
```

If your Udacity classroom already provides the dependencies, you can use that environment without reinstalling them.

### 3. Create your environment file

Linux/macOS:

```bash
cp .env.example .env
```

Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

The demo `.env.example` contains:

```dotenv
AWS_REGION=us-east-1
MODEL_ID=us.anthropic.claude-sonnet-4-6
```

Update `MODEL_ID` if your AWS lab specifically provides a different supported model ID or inference profile. A model ID being listed in a file does not guarantee that the current AWS account has access to it.

### 4. Configure AWS credentials

For the Udacity lab, use the classroom's **Load AWS Credentials** option if provided. The credentials should be available in the terminal environment before you run the script.

For a personal AWS environment, configure credentials using your normal AWS CLI profile:

```bash
aws configure
aws sts get-caller-identity
```

For temporary credentials, all three values are required:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN`

Do not commit real credentials, temporary lab credentials, or a populated `.env` file. Keep `.env` out of Git; commit only `.env.example`.

## Run the exercise

From the project root:

```bash
python smart_home_device_mgmt.py
```

The program runs three sample scenarios:

| Device | Sensor condition | Expected diagnosis | Corrective action |
|---|---|---|---|
| `DEV-001` — Living Room Thermostat | Temperature is 92.5°F | `overheating` | `restart_device` |
| `DEV-002` — Front Door Smart Lock | Connectivity is 12% | `firmware_issue` | `push_firmware_update` |
| `DEV-003` — Doorbell Camera | Battery is 7% | `low_battery` | `send_recharge_notification` |

The sample tool implementations use predefined in-memory data. The command tool returns a simulated command confirmation; it does not operate a real thermostat, lock, or camera.

## Learning notes

### 1. Coordinator pattern

A coordinator does not need to perform every task itself. It controls the sequence and passes the result of one specialized agent to the next:

```text
Sensor data
    |
    v
Device Monitor Agent
    |  sensor JSON
    v
Diagnostics Agent
    |  issue + device_id
    v
Command Agent
    |  command confirmation
    v
Action report
```

In this exercise, `run_device_pipeline(device_id)` is the coordinator.

### 2. Specialized agents and tools

Each agent has a focused responsibility:

- **Monitor:** retrieve sensor readings. It does not diagnose the root cause.
- **Diagnostics:** apply the diagnostic rules to the sensor JSON.
- **Command:** select and send the corrective action for each diagnosed issue.

The tool factory functions (`_make_read_sensor_data_tool()`, `_make_diagnose_issue_tool()`, and `_make_send_device_command_tool()`) return functions decorated with Strands' `@tool`. The returned tool function is passed to the relevant agent's `tools` list.

### 3. Passing structured data between agents

The coordinator converts the monitor response to a Python dictionary, serializes it with `json.dumps()`, and includes it in the diagnostics prompt. The diagnosis is parsed into a dictionary so the coordinator can loop through the `issues` list.

This keeps the handoff explicit:

- Monitor → Diagnostics: complete sensor-data JSON.
- Diagnostics → Command: device identifier and issue type.
- Command → Coordinator: action confirmation JSON.

### 4. System prompts and predictable output

The agents initially returned conversational responses, Markdown tables, or explanations around the tool output. Those responses were useful to a person but could not be passed directly to `json.loads()`.

The prompts were tightened to require:

- calling the correct tool,
- treating the tool result as the source of truth,
- returning only the tool's JSON,
- adding no explanations, Markdown, or code fences.

This made the agent output suitable for the next programmatic step. A system prompt guides model behavior but is not an absolute guarantee, so the completed script also includes `parse_json_response()` as a defensive parser. It first attempts normal JSON parsing and then attempts to extract a JSON object if extra text appears.

### 5. Model configuration

`AWS_REGION` and `MODEL_ID` are loaded from `.env` using `python-dotenv`. The same `create_model()` helper is used by all three agents, with `temperature=0.0` to favor consistent outputs.

The model ID must be supported and accessible in the AWS account and Region being used. If Bedrock reports that a model version is retired or unavailable, choose a currently supported model/inference profile that the lab account permits.

### 6. Retry handling

`run_agent_with_retry()` retries failed agent invocations up to three times, waiting 1, 2, and then 4 seconds between attempts. This is intended for transient service errors such as throttling. It does not fix invalid credentials, unavailable model IDs, or missing permissions.

### 7. Deterministic diagnostic rules

The diagnostic tool applies the predefined thresholds:

| Issue | Reading | Rule |
|---|---|---|
| Overheating | Temperature | Greater than 85 |
| Firmware issue | Connectivity | Less than 20 |
| Low battery | Battery | Less than 10 |

The command tool maps each issue to a predefined action. The tools contain the deterministic business logic; the agents coordinate and invoke them.

## Troubleshooting notes

### `ResourceNotFoundException` — model version has reached end of life

The configured model ID may be retired or unavailable. Check the model ID and inference profile allowed by the course lab and set the supported value in `.env`.

### `JSONDecodeError` at `json.loads()`

The agent may have returned natural language or Markdown instead of JSON. Check the relevant agent's system prompt and inspect the response with:

```python
print("DEBUG RESPONSE:", repr(agent_response))
```

The completed script's `parse_json_response()` provides a fallback, but the preferred solution is still to have each agent return only the tool's JSON result.

### `AccessDeniedException`

Check that the AWS credentials are loaded and that the account/role has permission to invoke the selected Bedrock model. In a managed classroom, some permissions and model access may be controlled by the lab.

### Credentials appear missing or expired

Reload the classroom credentials. Temporary credentials expire and must be refreshed. Do not copy credentials into source code or commit them to Git.

## Terminal output screenshots

Add your two terminal screenshots to the `resources` folder:

### Run output 1

![Smart Home Agent terminal output — first capture](resources/smart_home_agent_output1.png)

### Run output 2

![Smart Home Agent terminal output — second capture](resources/smart_home_agent_output2.png)

## Files and safety

- `.env.example` is a safe configuration template. Copy it to `.env` for local use.
- Do not commit `.env` or AWS credentials.
- The sensor readings and corrective actions are demonstration data and simulated operations.
- `architecture.svg` is the architecture diagram used by this README.
