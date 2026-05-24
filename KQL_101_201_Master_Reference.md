# KQL 101 & 201 Master Reference Guide

## 1. Data Transformation (The Shaping Layer)

### `project`
Selects, renames, or removes specific columns from the output.
* **Usage:** `project Column1, Column2` or `project NewName = OldName`
```kusto
SigninLogs
| where ResultType == "0"
| project TimeGenerated, UserPrincipalName, TargetIP = IPAddress, Location
```

### `extend`
Creates a brand new calculated column or overwrites an existing one.
* **Usage:** `extend NewColumn = Expression`
```kusto
DeviceNetworkEvents
| extend TotalBytes = ActionType + RemotePort 
| extend IsLocal = ipv4_is_private(RemoteIP) 
```

### `iff()`
An inline conditional function (if / then / else). 
* **Usage:** `iff(Condition, ValueIfTrue, ValueIfFalse)`
```kusto
SecurityEvent
| extend AccountType = iff(Account contains "Admin", "Privileged", "Standard")
```

### `case()`
Evaluates multiple conditions in sequence. Stops at the first true match. 
* **Usage:** `case(Condition1, Result1, Condition2, Result2, DefaultResult)`
```kusto
DeviceEvents
| extend ThreatLevel = case(
    ActionType == "AntivirusReport", "High",
    ActionType == "FirewallBlock", "Medium",
    "Unknown" 
)
```

---

## 2. Advanced Filtering (The Needle in the Haystack)

### `startswith` / `endswith`
Matches the beginning or end of a string. Highly optimized.
```kusto
DeviceProcessEvents
| where FolderPath startswith @"C:\Windows\Temp"
| where FileName endswith ".exe"
```

### `in` / `!in`
Checks if a column value matches any item within a defined list/array.
```kusto
let MaliciousIPs = dynamic(["192.168.1.50", "10.0.0.99", "172.16.5.5"]);
DeviceNetworkEvents
| where RemoteIP in (MaliciousIPs)
```

### `has_any`
Searches for any of the specified values across a text column or an array. 
```kusto
let SuspiciousTerms = dynamic(["mimikatz", "bloodhound", "psexec"]);
DeviceProcessEvents
| where ProcessCommandLine has_any (SuspiciousTerms)
```

---

## 3. Time Analysis (The Chronology Layer)

### `ago()`
Generates a timestamp relative to the exact moment the query is executed.
```kusto
SecurityEvent
| where TimeGenerated >= ago(24h) 
```

### `between`
Filters for events occurring within a specific window of time.
```kusto
SigninLogs
| where TimeGenerated between (ago(7d) .. ago(1d))
```

### `datetime()`
Casts a static string into a readable timestamp object. 
```kusto
SecurityEvent
| where TimeGenerated >= datetime(2026-05-15T14:30:00Z)
```

### `startofday()` / `endofday()`
Takes a timestamp and snaps it backward to 00:00:00 or forward to 23:59:59.
```kusto
SigninLogs
| where TimeGenerated between (startofday(ago(1d)) .. endofday(ago(1d)))
```

### `bin()`
Rounds timestamps down to a specific "bucket" or "bin" of time. Mandatory for charting.
```kusto
SecurityEvent
| summarize EventCount = count() by bin(TimeGenerated, 1h)
| render timechart 
```

---

## 4. Aggregation (The Summarization Layer)

### `summarize`
The foundational grouping command. Apply an aggregation function, optionally group `by` a column.

### `count()` vs. `dcount()`
* `count()`: Counts the total number of raw rows. 
* `dcount()`: (Distinct Count) Counts the number of *unique* items.
```kusto
SigninLogs
| summarize 
    TotalLogins = count(),                  
    UniqueUsers = dcount(UserPrincipalName) 
  by Location
```

### `min()` / `max()`
Finds the lowest or highest value in a dataset (often First Seen / Last Seen).
```kusto
DeviceProcessEvents
| where FileName == "powershell.exe"
| summarize 
    FirstExecution = min(TimeGenerated), 
    LastExecution = max(TimeGenerated) 
  by DeviceName
```

### `make_set()`
Takes all unique values of a column and packs them into a single JSON array.
```kusto
SigninLogs
| summarize 
    SuccessfulLogins = count(),
    IPsUsed = make_set(IPAddress) 
  by UserPrincipalName
```

### `top` / `sort`
* `top`: Returns the first *N* records sorted by a specific column (Highly performant).
* `sort` (or `order`): Sorts the *entire* dataset.
```kusto
DeviceNetworkEvents
| summarize EventCount = count() by DeviceName
| top 5 by EventCount desc
```
