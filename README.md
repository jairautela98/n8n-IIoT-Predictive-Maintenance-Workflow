Project using n8n with IIoT-Predictive-Maintenance-Workflow for Gas Pipelining  network leakage Prediction with Gas Sensor,Pressure Sensor,Actuator,controller at pipelining joint  and Python code and n8n workflow original

## Key Features
This project builds a real-time monitoring and response system for a gas pipeline joint. By combining physical sensors with an n8n orchestration layer, we can predict leaks before they become catastrophic and trigger immediate physical intervention via actuators. ## 1. System Components (Hardware Layer)

To monitor a pipeline joint effectively, we deploy a localized "Smart Controller" (like an ESP32 or Industrial PLC) equipped with:

    Gas Sensor (MQ-4/MQ-9): Detects trace amounts of Methane (CH4​) in the ambient air around the joint.

    Pressure Sensor: Monitors internal line pressure. A drop at the joint suggests a breach.

    Actuator (Solenoid Valve): A safety shut-off valve that n8n can trigger remotely to isolate the leaking section.

    Controller: Collects sensor data and transmits it to n8n via MQTT or HTTP.

## 2. The "Original" n8n Workflow Logic

In the n8n editor, the workflow follows a linear, logical progression from "Detection" to "Mitigation."
### Workflow Stages:

    Trigger (Polling/Webhook): Receives a JSON payload from the Controller.

    Code Node (Python): Analyzes the relationship between Pressure (P) and Gas Concentration (G).

    Switch Node: Routes the data based on the "Risk Level."

    Actuator Command: Sends a POST request back to the Controller to close the valve if a leak is confirmed.

    Notification: Logs the event in a database and alerts the team.

## 3. Python Predictive Logic (The "Brain")

This script runs inside the n8n Code Node. It uses a simple fusion algorithm: if gas levels rise while pressure drops, the confidence of a leak is near 100%.
Python

# n8n Python Code Node
import json

# Input from the sensors
data = _input.first().json
p_current = data.get("pressure")       # e.g., 42 PSI
gas_level = data.get("gas_ppm")        # e.g., 150 PPM
p_baseline = 45.0                      # Expected operational pressure

# Calculate Deviations
p_drop = (p_baseline - p_current) / p_baseline
gas_threshold_exceeded = gas_level > 100 # PPM threshold for methane

# Logic: Fusion of sensors
# Risk is high if pressure drops > 5% AND gas is detected
leak_probability = (p_drop * 0.7) + (0.3 if gas_threshold_exceeded else 0)
trigger_actuator = leak_probability > 0.6

return {
    "status": "EMERGENCY" if trigger_actuator else "STABLE",
    "leak_confidence": f"{round(leak_probability * 100, 2)}%",
    "action_required": trigger_actuator,
    "cmd": "CLOSE_VALVE" if trigger_actuator else "KEEP_OPEN"
}

## 4. Implementation Strategy (The 5 T's)

To ensure this isn't just a prototype, we apply the 5 T's of future systems:

    Target: Zero-emission pipelining by catching "micro-leaks" early.

    Technology: Using n8n allows for "Original" visual debugging while keeping the Python backend flexible for complex math.
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "PipelineJointTelemetry",
  "type": "object",
  "properties": {
    "metadata": {
      "type": "object",
      "properties": {
        "device_id": { "type": "string", "description": "Unique ID of the joint controller (e.g., JNT-TX-001)" },
        "timestamp": { "type": "string", "format": "date-time" },
        "firmware_version": { "type": "string" }
      },
      "required": ["device_id", "timestamp"]
    },
    "sensors": {
      "type": "object",
      "properties": {
        "pressure_psi": { "type": "number", "minimum": 0, "maximum": 1000 },
        "gas_methane_ppm": { "type": "integer", "minimum": 0 },
        "temperature_c": { "type": "number" },
        "vibration_level": { "type": "number", "description": "G-force units to detect structural shifts" }
      },
      "required": ["pressure_psi", "gas_methane_ppm"]
    },
    "actuator_state": {
      "type": "object",
      "properties": {
        "valve_open": { "type": "boolean" },
        "last_command_id": { "type": "string" }
      }
    }
  },
  "required": ["metadata", "sensors"]
}
CODE-Example Payload (The actual message)

Your ESP32 or PLC would publish a message like this every 5–10 seconds:
JSON

{
  "metadata": {
    "device_id": "JOINT-77-ALPHA",
    "timestamp": "2026-03-30T14:45:00Z",
    "firmware_version": "v2.4.1"
  },
  "sensors": {
    "pressure_psi": 43.2,
    "gas_methane_ppm": 12,
    "temperature_c": 18.5,
    "vibration_level": 0.02
  },
  "actuator_state": {
    "valve_open": true,
    "last_command_id": "CMD-998"
  }
}


Command JSON Schema (n8n to Controller)

This payload is sent via an MQTT Out node or an HTTP Request node in n8n when the Python logic returns action_required: true.
JSON

{
  "header": {
    "command_id": "CMD-2026-0330-001",
    "origin": "n8n-IIoT-Predictive-Engine",
    "timestamp": "2026-03-30T14:45:31Z"
  },
  "instruction": {
    "target_device": "JOINT-77-ALPHA",
    "action": "CLOSE_VALVE",
    "priority": "CRITICAL",
    "timeout_ms": 5000
  },
  "diagnostic_request": {
    "force_recalibration": false,
    "report_status_after": true
  },
  "security": {
    "auth_token": "SHA256_HASH_HERE"
  }
}
In the n8n interface, you would use a Set Node or the Expression Editor inside an HTTP/MQTT node to build this JSON dynamically.

    Target Device: Linked to {{ $node["Webhook"].json["metadata"]["device_id"] }}.

    Action: Linked to the output of your Python Code Node.

### The Workflow Logic Path

    Python Node: Detects pressure drop (P<40) + Gas spike (G>100).

    IF Node: Checks {{ $json.action_required }} == true.

    Command Node: Generates the JSON above and pushes it to the topic pipelines/joint-77/commands.

## 3. Actionable Hardware Response

Once the controller receives this JSON, it performs the following:

    Verifies the auth_token.

    Triggers the Digital Output pin connected to the Solenoid Actuator.

    Publishes a confirmation back to n8n: {"status": "VALVE_CLOSED", "command_id": "CMD-2026-0330-001"}.

## 4. The 5 T's Checklist for this Command

    Trust: The auth_token ensures that a hacker cannot spoof a "Close Valve" command to disrupt the gas supply.

    Technology: Using timeout_ms prevents the actuator from getting stuck in a loop if it encounters mechanical resistance.

    Timeline: By sending a command_id, we can track exactly how many milliseconds elapsed between "Leak Detected" and "Valve Closed."
{
  "nodes": [
    {
      "parameters": { "topic": "pipeline/joint1/sensors" },
      "name": "MQTT Trigger",
      "type": "n8n-nodes-base.mqttTrigger",
      "typeVersion": 1
    },
    {
      "parameters": {
        "command": "python3 leak_pred.py '{{JSON.stringify($json)}}'"
      },
      "name": "Run Prediction",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1
    },
    {
      "parameters": {
        "conditions": {
          "boolean": [{ "value1": "={{$json.leak_detected}}", "value2": true }]
        }
      },
      "name": "Is Leak?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1
    }
  ],
  "connections": {
    "MQTT Trigger": { "main": [[{ "node": "Run Prediction", "index": 0 }]] },
    "Run Prediction": { "main": [[{ "node": "Is Leak?", "index": 0 }]] }
  }
}












    

    Timeline: Implementing a "Heartbeat" every 5 seconds to ensure the controller is online.

    Talent: Empowering maintenance crews with a mobile dashboard instead of manual "sniffing" patrols.

    Trust: Encrypting the command sent to the Actuator to prevent unauthorized valve shutdowns.
