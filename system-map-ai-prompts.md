# System Map AI Prompts

## 1. Describe the System

Draw the system architecture of this system. Find every service and every connection between them. For each connection note the direction and type (HTTP, database query, pub/sub event, etc.) port number and frameworks like React, Django etc should be labeled. Output as ASCII art diagram. Each service gets its own labeled box. Every dependency between services gets a directional arrow labeled with the connection type. Do not add anything beyond services and their connections.

## 2. Convert to a Mermaid Diagram

Check your memory for the system diagram. Convert it to a Mermaid flowchart. Use graph LR layout. Each service is a node. Each connection is a labeled edge showing direction and connection type (HTTP, DB query, pub/sub, etc.)