# CO-5K Student API — JMeter Performance Test Suite

![JMeter](https://img.shields.io/badge/Apache%20JMeter-5.6.3-D22128?style=flat&logo=apache&logoColor=white)
![Protocol](https://img.shields.io/badge/Protocol-HTTPS-brightgreen?style=flat)
![Threads](https://img.shields.io/badge/Virtual%20Users-7-blue?style=flat)
![Data-Driven](https://img.shields.io/badge/Data--Driven-CSV-orange?style=flat)

A data-driven JMeter performance test suite targeting the **CO-6K Student API** (`co-5k-data.onrender.com`). The suite validates a two-step chained request flow — fetching the student roster and dynamically resolving individual student records — under a configurable concurrent user load.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Test Architecture](#test-architecture)
  - [Thread Group](#thread-group)
  - [CSV Data Source](#csv-data-source)
  - [Sampler 1 — GET /students (List)](#sampler-1--get-students-list)
  - [Sampler 2 — GET /students/{id} (Detail)](#sampler-2--get-studentsid-detail)
  - [Post-Processors](#post-processors)
  - [Listeners](#listeners)
- [Execution Instructions](#execution-instructions)
  - [CLI — Headless (Recommended)](#cli--headless-recommended)
  - [Parametrised Execution](#parametrised-execution)
  - [GUI Mode (Development Only)](#gui-mode-development-only)
- [Reporting](#reporting)
- [Configuration Reference](#configuration-reference)
- [Known Limitations & Recommendations](#known-limitations--recommendations)

---

## Project Overview

This test plan exercises two REST endpoints exposed by the CO-6K academic simulation platform:

| # | Method | Endpoint | Purpose |
|---|--------|----------|---------|
| 1 | `GET` | `/api/mock/co-6k/students` | Retrieve the full student list |
| 2 | `GET` | `/api/mock/co-6k/students/${id}` | Fetch a single student record by extracted ID |

The second request is **dynamically parameterised** using a student `id` extracted from the first response via a Regular Expression Post-Processor, making this a classic correlation pattern. An additional CSV Data Set Config supplements the test with pre-seeded variables (`id`, `name`, `percent`) to support data-driven test scenarios.

The goal is to measure throughput, response times, and error rates under a low-to-moderate concurrent user load, making this well-suited for smoke testing, baseline benchmarking, and regression performance checks against the CO-6K staging environment.

---

## Prerequisites

### Runtime

| Dependency | Version | Notes |
|------------|---------|-------|
| **Apache JMeter** | `5.6.3` (exact) | The `.jmx` was authored against this release. Behaviour on older versions is not guaranteed. |
| **Java** | `11` or `17` (LTS) | JMeter 5.6.x requires Java 11+. Use an OpenJDK distribution. |

### Environment

- Network access to `https://co-5k-data.onrender.com` from the machine running the test.
- The CSV data file `enroll.csv` must be present and accessible (see [CSV Data Source](#csv-data-source) for path details).

### Verify Installation

```bash
java -version
# Expected: openjdk version "17.x.x" or "11.x.x"

jmeter --version
# Expected: ... Apache JMeter 5.6.3 ...
```

> **Note:** Ensure `JMETER_HOME/bin` is on your system `PATH`, or invoke JMeter using its absolute path.

---

## Repository Structure

```
.
├── Thread_Group.jmx       # JMeter test plan (source of truth)
├── data/
│   └── enroll.csv         # CSV data file (see note on path below)
├── results/               # Generated test output (git-ignored)
│   ├── report/            # HTML dashboard output directory
│   └── results.jtl        # Raw JTL results file
└── README.md
```

> **Important:** The `.jmx` currently hardcodes the CSV path as `C:\Program Files\apache-jmeter-5.6.3\bin\enroll.csv`. Before running on any machine, update the `CSVDataSet filename` property to point to the correct absolute path, or override it via a JMeter property at runtime (see [Parametrised Execution](#parametrised-execution)).

---

## Test Architecture

### Thread Group

The single Thread Group is configured as follows:

| Parameter | Value | Description |
|-----------|-------|-------------|
| **Virtual Users (Threads)** | `7` | Concurrent simulated users |
| **Ramp-Up Period** | `1 second` | Time to spin up all 7 threads |
| **Loop Count** | `1` | Each thread executes the scenario once |
| **Delayed Start** | `true` | Threads do not start until the test engine signals them |
| **Same User on Next Iteration** | `true` | Thread reuses its state (cookies, variables) across iterations |
| **On Sample Error** | `continue` | Failures in one sampler do not abort the thread |

The effective load profile is: **7 concurrent users**, fully ramped within 1 second, executing a single pass of the two-request flow.

---

### CSV Data Source

A `CSV Data Set Config` is scoped at the Test Plan level, making its variables globally available to all threads.

| Property | Value |
|----------|-------|
| **File** | `C:\Program Files\apache-jmeter-5.6.3\bin\enroll.csv` |
| **Variable Names** | `id`, `name`, `percent` |
| **Delimiter** | `,` (comma) |
| **Ignore First Line** | `true` (header row skipped) |
| **Recycle on EOF** | `true` (loops back to the start when all rows are consumed) |
| **Stop Thread on EOF** | `false` |
| **Share Mode** | `shareMode.all` (all threads draw from the same shared cursor) |

> **CSV Format (expected):**
> ```
> id,name,percent
> 101,Alice Sharma,87
> 102,Bob Patil,73
> ...
> ```

Variables injected by the CSV — `${id}`, `${name}`, `${percent}` — are available for use in subsequent samplers, assertions, or scripts within the same iteration.

---

### Sampler 1 — GET /students (List)

**Name:** `1-GET-HTTP Request`

```
GET https://co-5k-data.onrender.com/api/mock/co-6k/students
```

| Setting | Value |
|---------|-------|
| Protocol | `HTTPS` |
| Domain | `co-5k-data.onrender.com` |
| Path | `/api/mock/co-6k/students` |
| Method | `GET` |
| Follow Redirects | `true` |
| Keep-Alive | `true` |
| Request Body | None |

This is the **anchor request**. It retrieves the full student listing and its response body is subsequently parsed by the Regular Expression Extractor to extract a dynamic `id` value used in Sampler 2.

---

### Sampler 2 — GET /students/{id} (Detail)

**Name:** `GET-Req_data-${id}`

```
GET https://co-5k-data.onrender.com/api/mock/co-6k/students/${id}
```

| Setting | Value |
|---------|-------|
| Protocol | `HTTPS` |
| Domain | `co-5k-data.onrender.com` |
| Path | `/api/mock/co-6k/students/${id}` |
| Method | `GET` |
| Follow Redirects | `true` |
| Keep-Alive | `true` |

The `${id}` token is resolved at runtime from the value extracted by the post-processor attached to Sampler 1. This request validates that a student record can be individually retrieved after being discovered from the list endpoint — a realistic API consumption pattern.

---

### Post-Processors

A **Regular Expression Extractor** is attached directly to Sampler 1:

| Property | Value | Notes |
|----------|-------|-------|
| **Scope** | Response Body | |
| **Reference Name** | `id` | Stored in `${id}` |
| **Regex** | `"id"\s*:\s*(\d+)` | Matches the first JSON `id` field |
| **Template** | `$66$` | Captures group 66 — note: likely intended to be `$1$` (see [Known Limitations](#known-limitations--recommendations)) |
| **Match No.** | `1` | Extracts the first match |
| **Default Value** | *(empty)* | No fallback; if extraction fails, `${id}` resolves to empty string |

---

### Listeners

Two listeners are configured inside the Thread Group for result collection:

| Listener | GUI Class | Purpose |
|----------|-----------|---------|
| **View Results Tree** | `ViewResultsFullVisualizer` | Per-request debug view: request/response data, assertions, latency, connect time |
| **Summary Report** | `SummaryReport` | Aggregated statistics: throughput, average/min/max response time, error rate |

Both listeners capture: response time, latency, connect time, bytes sent/received, HTTP status code, thread name, assertion results, and URL.

> **Note:** Listeners in GUI mode carry significant memory and CPU overhead. For any load test beyond trivial scale, disable listeners in the `.jmx` and collect results to a `.jtl` file instead (see CLI section below).

---
