# n8n-IIoT-Predictive-Maintenance-Workflow
{
  "name": "IIoT Predictive Maintenance Workflow",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "minutes",
              "minutesInterval": 5
            }
          ]
        }
      },
      "id": "schedule-trigger",
      "name": "Schedule Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.1,
      "position": [240, 300]
    },
    {
      "parameters": {
        "url": "={{$env.MQTT_BROKER_URL}}/api/sensors/data",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {}
      },
      "id": "fetch-sensor-data",
      "name": "Fetch Sensor Data",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [460, 300],
      "credentials": {
        "httpHeaderAuth": {
          "id": "1",
          "name": "IIoT API Auth"
        }
      }
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "a1",
              "name": "deviceId",
              "value": "={{$json.device_id}}",
              "type": "string"
            },
            {
              "id": "a2",
              "name": "temperature",
              "value": "={{$json.sensors.temperature}}",
              "type": "number"
            },
            {
              "id": "a3",
              "name": "vibration",
              "value": "={{$json.sensors.vibration}}",
              "type": "number"
            },
            {
              "id": "a4",
              "name": "pressure",
              "value": "={{$json.sensors.pressure}}",
              "type": "number"
            },
            {
              "id": "a5",
              "name": "operatingHours",
              "value": "={{$json.operating_hours}}",
              "type": "number"
            },
            {
              "id": "a6",
              "name": "timestamp",
              "value": "={{$json.timestamp}}",
              "type": "string"
            }
          ]
        },
        "options": {}
      },
      "id": "parse-sensor-data",
      "name": "Parse Sensor Data",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.2,
      "position": [680, 300]
    },
    {
      "parameters": {
        "operation": "insert",
        "table": "sensor_readings",
        "columns": "device_id, temperature, vibration, pressure, operating_hours, timestamp",
        "values": "={{$json.deviceId}}, {{$json.temperature}}, {{$json.vibration}}, {{$json.pressure}}, {{$json.operatingHours}}, '{{$json.timestamp}}'",
        "options": {}
      },
      "id": "store-readings",
      "name": "Store Readings in DB",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.4,
      "position": [900, 300],
      "credentials": {
        "postgres": {
          "id": "2",
          "name": "PostgreSQL IIoT"
        }
      }
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "c1",
              "leftValue": "={{$json.temperature}}",
              "rightValue": 85,
              "operator": {
                "type": "number",
                "operation": "gt"
              }
            },
            {
              "id": "c2",
              "leftValue": "={{$json.vibration}}",
              "rightValue": 7.5,
              "operator": {
                "type": "number",
                "operation": "gt"
              }
            },
            {
              "id": "c3",
              "leftValue": "={{$json.pressure}}",
              "rightValue": 150,
              "operator": {
                "type": "number",
                "operation": "gt"
              }
            }
          ],
          "combinator": "or"
        },
        "options": {}
      },
      "id": "check-thresholds",
      "name": "Check Alert Thresholds",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [1120, 300]
    },
    {
      "parameters": {
        "operation": "executeQuery",
        "query": "SELECT AVG(temperature) as avg_temp, AVG(vibration) as avg_vib, AVG(pressure) as avg_press\nFROM sensor_readings\nWHERE device_id = '{{$json.deviceId}}'\nAND timestamp > NOW() - INTERVAL '24 hours'",
        "options": {}
      },
      "id": "get-historical-avg",
      "name": "Get 24h Average",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.4,
      "position": [1340, 180],
      "credentials": {
        "postgres": {
          "id": "2",
          "name": "PostgreSQL IIoT"
        }
      }
    },
    {
      "parameters": {
        "jsCode": "// Calculate anomaly score using statistical deviation\nconst current = $input.first().json;\nconst historical = $input.last().json;\n\n// Calculate deviations\nconst tempDev = Math.abs(current.temperature - historical.avg_temp) / historical.avg_temp;\nconst vibDev = Math.abs(current.vibration - historical.avg_vib) / historical.avg_vib;\nconst pressDev = Math.abs(current.pressure - historical.avg_press) / historical.avg_press;\n\n// Weighted anomaly score (0-100)\nconst anomalyScore = Math.min(100, \n  (tempDev * 30 + vibDev * 40 + pressDev * 30) * 100\n);\n\n// Determine severity\nlet severity = 'low';\nif (anomalyScore > 70) severity = 'critical';\nelse if (anomalyScore > 50) severity = 'high';\nelse if (anomalyScore > 30) severity = 'medium';\n\n// Predict remaining useful life (RUL) in hours\nconst baseLife = 10000; // hours\nconst degradationFactor = anomalyScore / 100;\nconst rul = baseLife - (current.operatingHours * (1 + degradationFactor));\n\nreturn {\n  deviceId: current.deviceId,\n  anomalyScore: Math.round(anomalyScore * 100) / 100,\n  severity: severity,\n  remainingUsefulLife: Math.max(0, Math.round(rul)),\n  currentReadings: {\n    temperature: current.temperature,\n    vibration: current.vibration,\n    pressure: current.pressure\n  },\n  historicalAvg: historical,\n  deviations: {\n    temperature: Math.round(tempDev * 100),\n    vibration: Math.round(vibDev * 100),\n    pressure: Math.round(pressDev * 100)\n  },\n  maintenanceRecommended: anomalyScore > 50,\n  timestamp: current.timestamp\n};"
      },
      "id": "ml-analysis",
      "name": "ML Anomaly Detection",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1560, 180]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "c1",
              "leftValue": "={{$json.severity}}",
              "rightValue": "low",
              "operator": {
                "type": "string",
                "operation": "notEquals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "filter-alerts",
      "name": "Filter Actionable Alerts",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [1780, 180]
    },
    {
      "parameters": {
        "operation": "insert",
        "table": "maintenance_alerts",
        "columns": "device_id, anomaly_score, severity, remaining_useful_life, alert_data, created_at",
        "values": "='{{$json.deviceId}}', {{$json.anomalyScore}}, '{{$json.severity}}', {{$json.remainingUsefulLife}}, '{{JSON.stringify($json)}}', NOW()",
        "options": {}
      },
      "id": "create-alert",
      "name": "Create Maintenance Alert",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.4,
      "position": [2000, 80],
      "credentials": {
        "postgres": {
          "id": "2",
          "name": "PostgreSQL IIoT"
        }
      }
    },
    {
      "parameters": {
        "authentication": "oAuth2",
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "C12345678",
          "mode": "id"
        },
        "text": "=🚨 **Predictive Maintenance Alert**\n\n**Device:** {{$json.deviceId}}\n**Severity:** {{$json.severity.toUpperCase()}}\n**Anomaly Score:** {{$json.anomalyScore}}%\n**Remaining Useful Life:** {{$json.remainingUsefulLife}} hours\n\n**Current Readings:**\n• Temperature: {{$json.currentReadings.temperature}}°C\n• Vibration: {{$json.currentReadings.vibration}} mm/s\n• Pressure: {{$json.currentReadings.pressure}} PSI\n\n**Deviations from 24h Average:**\n• Temperature: +{{$json.deviations.temperature}}%\n• Vibration: +{{$json.deviations.vibration}}%\n• Pressure: +{{$json.deviations.pressure}}%\n\n{{$json.maintenanceRecommended ? '⚠️ **Immediate maintenance recommended**' : ''}}",
        "otherOptions": {}
      },
      "id": "slack-notification",
      "name": "Send Slack Alert",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.1,
      "position": [2220, 80],
      "credentials": {
        "slackOAuth2Api": {
          "id": "3",
          "name": "Slack OAuth"
        }
      }
    },
    {
      "parameters": {
        "fromEmail": "alerts@iiot-system.com",
        "toEmail": "maintenance-team@company.com",
        "subject": "=⚠️ {{$json.severity.toUpperCase()}} Priority - Device {{$json.deviceId}} Maintenance Required",
        "text": "=Device {{$json.deviceId}} requires attention.\n\nAnomaly Score: {{$json.anomalyScore}}%\nSeverity: {{$json.severity}}\nEstimated Remaining Life: {{$json.remainingUsefulLife}} hours\n\nPlease review the maintenance dashboard for detailed diagnostics.",
        "options": {
          "allowUnauthorizedCerts": false
        }
      },
      "id": "email-notification",
      "name": "Send Email Alert",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2.1,
      "position": [2220, 280],
      "credentials": {
        "smtp": {
          "id": "4",
          "name": "SMTP Account"
        }
      }
    },
    {
      "parameters": {
        "method": "POST",
        "url": "={{$env.MAINTENANCE_API_URL}}/work-orders",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "device_id",
              "value": "={{$json.deviceId}}"
            },
            {
              "name": "priority",
              "value": "={{$json.severity}}"
            },
            {
              "name": "description",
              "value": "=Predictive maintenance alert - Anomaly score: {{$json.anomalyScore}}%"
            },
            {
              "name": "estimated_rul_hours",
              "value": "={{$json.remainingUsefulLife}}"
            }
          ]
        },
        "options": {}
      },
      "id": "create-work-order",
      "name": "Create Work Order",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [2440, 180],
      "credentials": {
        "httpHeaderAuth": {
          "id": "5",
          "name": "Maintenance System API"
        }
      }
    },
    {
      "parameters": {},
      "id": "no-action-needed",
      "name": "No Action Needed",
      "type": "n8n-nodes-base.noOp",
      "typeVersion": 1,
      "position": [1780, 400]
    },
    {
      "parameters": {
        "operation": "executeQuery",
        "query": "UPDATE sensor_readings\nSET processed = true\nWHERE device_id = '{{$json.deviceId}}'\nAND timestamp = '{{$json.timestamp}}'",
        "options": {}
      },
      "id": "mark-processed",
      "name": "Mark as Processed",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.4,
      "position": [1120, 500],
      "credentials": {
        "postgres": {
          "id": "2",
          "name": "PostgreSQL IIoT"
        }
      }
    }
  ],
  "connections": {
    "Schedule Trigger": {
      "main": [
        [
          {
            "node": "Fetch Sensor Data",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Fetch Sensor Data": {
      "main": [
        [
          {
            "node": "Parse Sensor Data",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Sensor Data": {
      "main": [
        [
          {
            "node": "Store Readings in DB",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Store Readings in DB": {
      "main": [
        [
          {
            "node": "Check Alert Thresholds",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Check Alert Thresholds": {
      "main": [
        [
          {
            "node": "Get 24h Average",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Mark as Processed",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Get 24h Average": {
      "main": [
        [
          {
            "node": "ML Anomaly Detection",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "ML Anomaly Detection": {
      "main": [
        [
          {
            "node": "Filter Actionable Alerts",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Filter Actionable Alerts": {
      "main": [
        [
          {
            "node": "Create Maintenance Alert",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "No Action Needed",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Create Maintenance Alert": {
      "main": [
        [
          {
            "node": "Send Slack Alert",
            "type": "main",
            "index": 0
          },
          {
            "node": "Send Email Alert",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Send Slack Alert": {
      "main": [
        [
          {
            "node": "Create Work Order",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Send Email Alert": {
      "main": [
        [
          {
            "node": "Create Work Order",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "settings": {
    "executionOrder": "v1"
  },
  "staticData": null,
  "tags": [],
  "triggerCount": 1,
  "updatedAt": "2026-01-20T12:00:00.000Z",
  "versionId": "1"
}
