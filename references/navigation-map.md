# Tideways Navigation Map

## URL Structure

Base: `https://app.tideways.io`

All project URLs follow: `/o/{Organization}/{Project}/...`

Example: `/o/MyCompany/MyProject/...`

## Sections and Navigation

### Dashboard
```
URL: /dashboard
Command: playwright-cli goto https://app.tideways.io/dashboard
```
Shows all projects in organization with summary metrics (Response Time, RPM, Failure Rate) for 15min/yesterday/7days.

### Performance Overview
```
URL: /o/{Org}/{Project}
URL with more rows: /o/{Org}/{Project}?count=40
Command: playwright-cli click [project-link-ref]
```
**Main view.** Transaction table sorted by Impact. Shows:
- 95th percentile response time
- Max memory
- Total requests
- Page Cache Hit Rate
- Failure rate
- Response time chart (percentile + request volume)

**Filters available:**
- Service: `app` (frontend), `app:cli` (CLI/cron), `backend` (admin), `All`
- Environment: `production`, `staging`, etc.
- Time span: 60 minutes, 3 hours, 12 hours, 24 hours
- Sub-views: PHP Performance (`?st=php`), External HTTP Calls (`?st=external`)

**Transaction table columns:**
| Column | Meaning |
|--------|---------|
| Name | Controller class with bottleneck prefix |
| Requests | Request count in time window |
| Average Response | Mean response time |
| Slowest Response | Maximum response time |
| Memory | Peak memory usage |
| Failures | Error count |
| Impact | Percentage of total server time consumed |

**Bottleneck prefixes in transaction names:**
| Prefix | Meaning |
|--------|---------|
| SQL | Database queries are the primary bottleneck |
| COMPILE | PHP compilation/autoloading is the bottleneck |
| SESSION | Session read/write is blocking |
| HTTP | External HTTP calls are the bottleneck |
| (none) | No single dominant bottleneck |

### Traces
```
URL: /o/{Org}/{Project}/traces?hours=24
Command: playwright-cli click [traces-link-ref]
```
Individual request traces with:
- Timeline view (waterfall of function calls)
- Flame graph
- SQL queries executed
- External calls
- Memory allocation

### Tracepoints
```
URL: /o/{Org}/{Project}/traces/boosts
Command: playwright-cli click [tracepoints-link-ref]
```
Custom profiling points. Can trigger detailed tracing for specific functions or transactions.

### History
```
URL: /o/{Org}/{Project}/history
Granularity options:
  Day: /history?granularity=day&date=YYYY-MM-DD
  Week: /history?granularity=week&date=YYYY-MM-DD
  Month: /history?granularity=month&date=YYYY-MM-DD
```
Historical performance data. Use for:
- Before/after deploy comparison
- Trend analysis over time
- Seasonal traffic pattern identification

### Errors / Exceptions
```
URL: /o/{Org}/{Project}/issues/errors
Command: playwright-cli click [errors-link-ref]
```
Fatal errors with:
- Error message and class
- Stacktrace
- Occurrence count and frequency
- Affected transactions
- First/last occurrence timestamps

**Filters:** Status (Open/Resolved/Ignored), Service, Environment

### Non-Fatal Errors
```
URL: /o/{Org}/{Project}/issues/non-fatals
```
PHP warnings, notices, deprecations. Often signal:
- Upcoming PHP version incompatibilities
- Incorrect type usage
- Missing configuration

### Slow SQLs
```
URL: /o/{Org}/{Project}/issues/slow-sql
```
Queries exceeding threshold (default: 1000ms). Each entry shows:
- Full SQL statement (parameterized)
- Calling function (stacktrace)
- Query time
- Occurrence count since last release
- Affected transactions with counts
- Frequency chart over time

**Detail view includes:**
- Stacktrace (full PHP call chain)
- Context tab (request metadata)
- Notification log
- Changelog

**Filters:** Status (Open/Resolved/Ignored), Service, Transaction

### Releases
```
URL: /o/{Org}/{Project}/issues/release
```
Deploy tracking. Shows:
- Release timestamps
- New issues introduced per release
- Performance changes per release

### Incidents
```
URL: /o/{Org}/{Project}/issues/incident
```
Grouped high-impact events. Automatic detection of:
- Response time spikes
- Error rate increases
- Availability issues

### Observations
```
URL: /o/{Org}/{Project}/issues/observation
```
Automated performance hints from Tideways analysis engine:
- Cache Hit Ratio too low
- Significant Autoloading time
- Significant Compiling time
- Slow modules (e.g., Google Tag Manager)
- N+1 Queries
- And other detected anti-patterns

### Project Settings
```
URL: /o/{Org}/{Project}/settings
Services: /o/{Org}/{Project}/settings/services
Environments: /o/{Org}/{Project}/settings/environments
```

## Snapshot Reading Guide

Tideways snapshots contain structured data. Key patterns:

**Transaction table rows:** Look for `row` elements containing `cell` elements with:
- Transaction name (with link to detail)
- Numeric values (requests, response times, memory, failures, impact)

**Slow SQL entries:** Look for `link` elements containing:
- Status indicator (Open/Resolved)
- Occurrence count (e.g., "1 234x")
- SQL fragment (truncated)
- Source function
- Query time
- Timestamp

**Observation entries:** Look for `link` elements with descriptive text about the observation type and affected servers.

## Time Controls

Most views support time range selection:
- Buttons for preset ranges (60min, 3h, 12h, 24h)
- Navigation arrows (left/right/newest)
- Live mode indicator

For Slow SQLs and Errors, timeframe links:
- 2 hours, 5 days, 7 days, 14 days, 30 days
