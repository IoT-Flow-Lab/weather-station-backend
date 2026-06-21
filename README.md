# 🌊 IoT Water Station Telemetry Backend

A production-grade, highly optimized, hybrid server (HTTP + MQTT) built with **NestJS**, **TypeORM**, and **TimescaleDB** designed to ingest, process, and query real-time time-series telemetry data from distributed weather and water monitoring stations.

---

## 🏗️ System Architecture

The application functions as a **Hybrid Server**, handling HTTP REST API queries and subscribing to real-time MQTT message queues simultaneously.

```
                    ┌────────────────────────┐
                    │ Distributed IoT Sensor │
                    │    Stations (ESP32)    │
                    └───────────┬────────────┘
                                │ MQTT Publish
                                ▼
                    ┌────────────────────────┐
                    │  Public MQTT Broker    │
                    │   (broker.hivemq.com)  │
                    └───────────┬────────────┘
                                │
                        Topic:  │ weather_station/+/+/telemetry
                                ▼
             ┌──────────────────────────────────────┐
             │       NestJS Hybrid Service          │
             ├──────────────────────────────────────┤
             │  ┌────────────────────────────────┐  │
             │  │     MQTT Microservice Ingest   │  │
             │  └───────────────┬────────────────┘  │
             │                  │ Save              │
             │                  ▼                   │
             │  ┌────────────────────────────────┐  │
             │  │   HTTP REST API (Port 3000)    │  │
             │  └───────────────┬────────────────┘  │
             └──────────────────┼───────────────────┘
                                │ Read / Write
                                ▼
             ┌──────────────────────────────────────┐
             │         TimescaleDB (Docker)         │
             │   ┌──────────────────────────────┐   │
             │   │    weather_telemetry         │   │
             │   │   (Partitioned Hypertable)   │   │
             │   └──────────────────────────────┘   │
             └──────────────────────────────────────┘
```

1. **Telemetry Ingestion:** Distributing sensor data (DHT11 temperature/humidity, BMP280 temperature/pressure, boot logs, and uptimes) over MQTT to `broker.hivemq.com`.
2. **NestJS Ingest Engine:** Subscribes to wildcard patterns (`weather_station/+/+/telemetry`) to parse and validate incoming payloads using `class-validator` DTO pipes.
3. **Time-Series Storage:** Saves telemetry records to a TimescaleDB database, where the primary table is converted into a **TimescaleDB Hypertable** along the `timestamp` axis.
4. **Data Query Service:** Exposes REST endpoints supporting pagination (for front-end infinite scrolls/lazy loading), historical date-time range filters, and station sensor aggregate status summaries.

---

## ⚡ Key Features

* **TimescaleDB Hypertable Engine:** Automatic conversion of relational data models into hypertables for lightning-fast querying over massive volumes of time-series IoT data.
* **Dual Broker/Web Mode:** Handles REST requests (port 3000) and asynchronous MQTT broker streaming asynchronously in the same Node lifecycle loop.
* **Industrial Validation DTOs:** Incoming payloads and outgoing responses are strictly typed and sanitized using NestJS validation pipes.
* **Interactive Scalar OpenAPI Interface:** Premium dark-themed interactive API sandbox served directly at `/api` using zero-dependency rendering scripts and the local `openapi.yaml` spec.
* **Lazy Loading & Pagination:** Telemetry endpoints support paginated search inputs to enable low-overhead mobile scrolling and infinite dashboard load steps.

---

## 💾 Database Setup (TimescaleDB)

TimescaleDB is running via Docker. To launch and initialize the DB locally, run the following:

```bash
docker run -d --name timescaledb -p 5432:5432 -e POSTGRES_PASSWORD=RS16802 timescale/timescaledb:latest-pg16
```

### Database Schema Auto-Migration
On startup, TypeORM automatically synchronizes the relational schema. Immediately following, the backend hooks into `onModuleInit` to issue a Timescale query:
```sql
SELECT create_hypertable('weather_telemetry', 'timestamp', if_not_exists => TRUE);
```
This converts the default table into an optimized hypertable chunked chronologically.

---

## 🚀 Getting Started

### Prerequisites
* **Node.js** (v18 or higher)
* **Docker Desktop** (running PostgreSQL / TimescaleDB)

### 1. Install Dependencies
Run the installation command inside the root directory:
```bash
npm install
```

### 2. Build the Application
Compile the TypeScript files and bundle static configuration assets:
```bash
npm run build
```

### 3. Run the Server
* **Development Mode (with auto-watch):**
  ```bash
  npm run start:dev
  ```
* **Production Mode:**
  ```bash
  npm run start:prod
  ```

---

## 🔌 API Documentation (HTTP REST Endpoints)

Exhaustive API documentation is hosted directly on the server at **`http://localhost:3000/api`**. Below is a summary of the REST interface:

### 1. Paginated Telemetry (Supports Lazy Loading)
Retrieves historical logs sorted by time descending. Use this endpoint to feed lists that fetch more data as the user scrolls.

* **Endpoint:** `GET /telemetry`
* **Query Parameters:**
  * `page` (optional): Page offset integer (default: `1`).
  * `limit` (optional): Max records returned per page (default: `20`, max: `100`).
  * `station_id` (optional): Filter results to a specific station (e.g. `WS-C3-E072A17299CC`).
* **Example Response:**
  ```json
  {
    "data": [
      {
        "timestamp": "2026-06-21T14:32:33.000Z",
        "station_id": "WS-C3-E072A17299CC",
        "uptime_seconds": 10,
        "boot_count": 20,
        "sensors": {
          "dht11": { "status": "OK", "humidity_pct": 77.2, "temperature_c": 26.6 },
          "bmp280": { "status": "OK", "pressure_hpa": 956.3, "temperature_c": 28.1 }
        },
        "topic": "weather_station/sci.pdn.ac.lk/e97d1b32/telemetry",
        "domain": "sci.pdn.ac.lk",
        "device_id": "e97d1b32",
        "received_at": "2026-06-21T14:32:34.696Z"
      }
    ],
    "meta": {
      "total": 12,
      "page": 1,
      "limit": 20,
      "hasNextPage": false
    }
  }
  ```

### 2. Fetch Latest Telemetry
Retrieves the absolute newest telemetry log written to the database.

* **Endpoint:** `GET /telemetry/latest`
* **Query Parameters:**
  * `station_id` (optional): Filter to find the latest record of a specific station.
* **Example Response:**
  ```json
  {
    "timestamp": "2026-06-21T14:32:33.000Z",
    "station_id": "WS-C3-E072A17299CC",
    "uptime_seconds": 10,
    "boot_count": 20,
    "sensors": {
      "dht11": { "status": "OK", "humidity_pct": 77.2, "temperature_c": 26.6 },
      "bmp280": { "status": "OK", "pressure_hpa": 956.3, "temperature_c": 28.1 }
    },
    "topic": "weather_station/sci.pdn.ac.lk/e97d1b32/telemetry",
    "domain": "sci.pdn.ac.lk",
    "device_id": "e97d1b32",
    "received_at": "2026-06-21T14:32:34.696Z"
  }
  ```

### 3. Date-Time Range Query
Filters time-series data between start and end date parameters (results returned chronologically ascending).

* **Endpoint:** `GET /telemetry/range`
* **Query Parameters (Required):**
  * `start`: Start timestamp in ISO-8601 format (e.g. `2026-06-21T00:00:00Z`).
  * `end`: End timestamp in ISO-8601 format (e.g. `2026-06-21T23:59:59Z`).
* **Query Parameters (Optional):**
  * `station_id`: Filter range results by a specific station.
  * `page`: Pagination offset (default: `1`).
  * `limit`: Max records per page (default: `50`).
* **Example URL:** `http://localhost:3000/telemetry/range?start=2026-06-21T00:00:00Z&end=2026-06-21T23:59:59Z`

### 4. Stations & Sensor Metadata Status
Returns a list of all distinct weather stations found in the database, when they last updated, and the nested listing and health metrics of all individual sensors bound to them.

* **Endpoint:** `GET /telemetry/sensors`
* **Example Response:**
  ```json
  [
    {
      "station_id": "WS-C3-E072A17299CC",
      "device_id": "e97d1b32",
      "domain": "sci.pdn.ac.lk",
      "last_seen": "2026-06-21T14:32:33.000Z",
      "sensors": [
        {
          "name": "dht11",
          "status": "OK",
          "data": {
            "status": "OK",
            "humidity_pct": 77.2,
            "temperature_c": 26.6
          }
        },
        {
          "name": "bmp280",
          "status": "OK",
          "data": {
            "status": "OK",
            "pressure_hpa": 956.3,
            "temperature_c": 28.1
          }
        }
      ]
    }
  ]
  ```

---

## 📖 OpenAPI Specification serving

* **Interactive Sandbox Endpoint:** `http://localhost:3000/api`
* **Raw Spec Document:** `http://localhost:3000/api/openapi.yaml`

The server implements a custom Scalar interactive browser loader inside [src/app.controller.ts](file:///D:/My%20University/CSC4122%20Internet%20of%20Things/project/water-station-backend/src/app.controller.ts) which references the raw specification in [src/openapi.yaml](file:///D:/My%20University/CSC4122%20Internet%20of%20Things/project/water-station-backend/src/openapi.yaml). It bypasses standard bloated Swagger Node packages, keeping dependencies light and execution fast.

---

## 🧪 Simulation Testing (Publishing MQTT Data)

We provide a developer simulation script `test-publish.js` to simulate weather station data transmission. 

To run the simulator:
```bash
node test-publish.js
```

The script connects to the public HiveMQ broker, formats a standard JSON payload containing virtual DHT11 and BMP280 records, and posts to `weather_station/sci.pdn.ac.lk/e97d1b32/telemetry`. If the server is running, the console will output:
```
[TelemetryController] MQTT message received on topic: weather_station/sci.pdn.ac.lk/e97d1b32/telemetry
[TelemetryService] [DATABASE SAVE] Successfully saved telemetry record to TimescaleDB for station WS-C3-E072A17299CC.
```

---

## 📁 Project Structure

```
water-station-backend/
├── src/
│   ├── telemetry/
│   │   ├── dto/
│   │   │   ├── create-telemetry.dto.ts   # MQTT payload validation
│   │   │   └── query-telemetry.dto.ts    # REST query parameters
│   │   ├── telemetry.controller.ts       # MQTT subscription & HTTP routes
│   │   ├── telemetry.entity.ts           # TimescaleDB TypeORM entity schema
│   │   ├── telemetry.module.ts           # Telemetry DI Container Configuration
│   │   └── telemetry.service.ts          # Core persistence & DB hypertable query rules
│   ├── app.controller.ts                 # API docs landing and raw openapi yaml endpoints
│   ├── app.module.ts                     # Main Database Connection bootstrap
│   ├── main.ts                           # Hybrid Server (HTTP Express + MQTT Ingest Microservice)
│   └── openapi.yaml                      # OpenAPI industry-standard endpoints definitions
├── nest-cli.json                         # Build assets configs
├── test-publish.js                       # MQTT testing script
└── tsconfig.json                         # Typescript configuration rules
```
