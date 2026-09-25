# Thunder Intelligence

**Real-time War Thunder telemetry analytics · Python · Confluent Cloud · Apache Kafka · Apache Flink**

![Thunder Intelligence](Thunder_Intelligence_Cover.png)

Thunder Intelligence is a personal data-streaming project built around the local telemetry interfaces exposed by War Thunder. It captures live vehicle data while the game is running, publishes that data to Kafka in Confluent Cloud, processes session statistics with Apache Flink, and presents the result in a local web dashboard.

I built the project to experiment with a complete real-time pipeline rather than a static dataset: game telemetry enters the system continuously, is transported through Kafka, processed, stored and then displayed while the session is still taking place.

## Architecture

<img width="1920" height="1080" alt="geometric-instagram-183-landscape-1920x1080" src="https://github.com/user-attachments/assets/15faff8d-11c8-4b0d-bef2-4974b780b2fb" />


The dashboard consumes the aggregated statistics produced through Flink and also reads raw telemetry and event data from Kafka where required. SQLite is used locally to preserve captured session information between executions.

## Features

- Live telemetry capture from the War Thunder local API
- Kafka producers and consumers using Confluent Cloud
- Stream processing with Apache Flink
- Per-vehicle session tracking
- Aircraft, helicopter and ground-vehicle support
- Route recording and map reconstruction
- HUD/combat event capture
- Local SQLite session history
- Vehicle search and filtering by type, nation and class
- Route replay controls
- Faster capture mode for demonstrations and recordings
- Local self-tests and a Kafka-free dry-run mode

## Requirements

You will need:

- Python 3
- War Thunder
- A Confluent Cloud account and Kafka cluster
- Apache Flink available in the Confluent environment
- The Python packages listed in `requirements.txt`

The project was developed and tested primarily on Windows, so the commands below use PowerShell.

## 1. Installation

Clone or download the repository and open PowerShell in the project directory.

Create a virtual environment:

```powershell
python -m venv .venv
```

Activate it:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install the dependencies:

```powershell
pip install -r requirements.txt
```

## 2. Confluent Cloud configuration

The repository includes `client.properties.example`. Copy it and rename the copy to:

```text
client.properties
```

Then replace the placeholders with your own Confluent Cloud credentials:

```properties
bootstrap.servers=<YOUR_CONFLUENT_BOOTSTRAP_HOST>:9092
security.protocol=SASL_SSL
sasl.mechanisms=PLAIN
sasl.username=<YOUR_API_KEY>
sasl.password=<YOUR_API_SECRET>
session.timeout.ms=45000
client.id=thunder-intelligence-public
```

`client.properties` is intentionally excluded by `.gitignore`. Do not commit or share this file because it contains your credentials.

## 3. Kafka topics

The current version uses three main topics:

```text
warthunder-telemetry-v2
warthunder-events
warthunder-stats-v2
```

`warthunder-telemetry-v2` receives the live telemetry captured from the game. `warthunder-events` receives detected HUD/game events. `warthunder-stats-v2` contains the aggregated session statistics consumed by the dashboard.

Make sure the required topics and schemas for your Confluent environment are configured before running the complete pipeline.

## 4. Apache Flink

The aggregated statistics are produced through Apache Flink.

Before recording a complete live session, start the Flink SQL statement that writes the processed telemetry into `warthunder-stats-v2`. Only **one** instance of the existing `INSERT INTO warthunder-stats-v2 ...` statement should be running.

A permanent test `SELECT` is not required. Routes and HUD events are consumed directly from Kafka and do not need another continuous Flink query.

If you are using your own Confluent environment rather than the original project environment, you will need to create the corresponding Flink tables and aggregation statement for the topics above before this stage can run.

## 5. Run Thunder Intelligence

For a complete live session I use **two PowerShell terminals**.

War Thunder must also be running, and the local game API must be available at `127.0.0.1:8111`.

### Terminal 1 — Dashboard

From the project directory:

```powershell
.\.venv\Scripts\python.exe thunder_dashboard.py
```

Then open:

```text
http://127.0.0.1:8765
```

### Terminal 2 — Telemetry collector

Open a second PowerShell window in the same directory.

Normal capture:

```powershell
.\.venv\Scripts\python.exe thunder_collector_v2_6.py --seconds 0
```

Faster capture mode used for demonstrations/video:

```powershell
.\.venv\Scripts\python.exe thunder_collector_v2_6.py --video --seconds 0
```

`--seconds 0` keeps the collector running until it is stopped manually.

`--video` targets a sampling interval of roughly 0.5 seconds instead of the normal 1 second. This is a capture target, not a guaranteed end-to-end Kafka → Flink → dashboard latency.

## Recommended startup order

```text
1. Start War Thunder
2. Start the Flink INSERT statement in Confluent Cloud
3. Start thunder_dashboard.py in Terminal 1
4. Open http://127.0.0.1:8765
5. Start thunder_collector_v2_6.py in Terminal 2
6. Enter a match or test session in War Thunder
```

Once the game begins exposing telemetry, the collector should start reading the local API and publishing records to Kafka.

## Quick checks

Before starting the complete pipeline, both main Python components have a local self-test.

Collector:

```powershell
.\.venv\Scripts\python.exe thunder_collector_v2_6.py --self-test
```

Dashboard:

```powershell
.\.venv\Scripts\python.exe thunder_dashboard.py --self-test
```

These checks do not require a live War Thunder session.

## Test the collector without Kafka

For changes to the capture logic, the collector can also run without publishing anything to Kafka:

```powershell
.\.venv\Scripts\python.exe thunder_collector_v2_6.py --video --dry-run --seconds 30
```

This captures for 30 seconds and is useful for checking the War Thunder side of the pipeline independently.

## Stopping the project

Use `Ctrl+C` in the collector terminal and in the dashboard terminal.

When you have finished testing, stop the Flink `INSERT` statement in Confluent Cloud as well so that stream-processing resources are not left running unnecessarily.

## Main files

```text
thunder_collector_v2_6.py   Current telemetry collector
thunder_dashboard.py        Local backend and Kafka consumers
index.html                  Dashboard interface
app.js                      Frontend logic
style.css                   Dashboard styling
vehicles_index.json         Vehicle metadata used by the application
client.properties.example   Safe Confluent configuration template
requirements.txt            Python dependencies
media/                      Local vehicle media used by the dashboard
data/                       Local application data directory
```

Older collector versions are kept in the repository as part of the development history. The current collector is `thunder_collector_v2_6.py`.

## Data notes

Thunder Intelligence only uses information available through the War Thunder local interfaces used by the project. Not every value visible in the game is necessarily exposed there.

Vehicle nation and class can be inferred from a conservative list of known vehicle identifiers. Unknown vehicles are left unidentified or unclassified rather than being assigned arbitrary metadata.

Statistics shown by the dashboard describe sessions actually captured by Thunder Intelligence. They should not be interpreted as a complete database of every War Thunder vehicle.

## Security

Never publish `client.properties` or any other file containing:

- Confluent API keys
- Confluent API secrets
- Cloud credentials
- Private connection information

The public repository contains `client.properties.example` so that every user can configure their own environment without exposing credentials.

## Project presentation

A separate project overview is available in:

```text
Thunder_Intelligence_Presentation_EN.pdf
```

It provides a higher-level introduction to the idea, architecture and technologies used in the project.

## Technologies

**Python · Apache Kafka · Apache Flink · Confluent Cloud · SQLite · JavaScript · HTML · CSS · War Thunder Local API**

## Author

**Eduardo Romera Martínez**

Personal software and data-streaming project focused on real-time telemetry, event processing and visualization.
