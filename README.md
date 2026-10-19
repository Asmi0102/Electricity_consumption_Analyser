# Electricity Usage Efficiency and Cost Optimizer

## Description and problem statement
Households use electricity across broad categories such as cooling, kitchen, lighting, and entertainment. This Python application estimates category-level energy use and cost, flags categories above a configurable daily-use threshold, and estimates savings from reducing daily operating hours. It is an educational estimate, not a utility bill or a measurement device.

## Objectives
- Validate household usage input with clear errors.
- Calculate daily and billing-period kWh and estimated cost.
- Identify high-consumption and highest-consuming categories.
- Estimate energy and money saved by reducing operating hours.
- Demonstrate functional programming, TCP sockets, an explicit FSM, SymPy, and automated testing.

## Features
- JSON request/response over TCP on `127.0.0.1:50515`.
- Newline-delimited message framing, timeouts, malformed JSON handling, and validation errors.
- FSM trace included in every analyzer response.
- SymPy symbolic formula and numeric substitution example.
- `unittest` suite and a readable test report.

## Requirements
- Python 3.10 or newer recommended.
- SymPy (only external dependency). Networking uses Python's built-in `socket` module.

## Install (Windows)
Open Command Prompt or PowerShell in this project folder:

```powershell
python -m pip install -r requirements.txt
```

Optional virtual environment:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

In Command Prompt, activate with `.venv\Scripts\activate.bat` instead.

## Run client and server
1. Terminal 1: `python server.py`
2. Terminal 2: `python client.py`

The client sends the sample dataset from `analyzer.py`. To send a custom JSON file, run `python client.py household.json`. The JSON file must contain `categories`, `tariff_per_kwh`, `billing_days`, and `high_consumption_threshold_kwh`; each category has `name`, `power_watts`, `hours_per_day`, and optional `reduce_hours_per_day`.

Stop the server with Ctrl+C. The server handles one request per TCP connection and then waits for another client.

## Tests
```powershell
python test_report.py
python -m unittest -v test_analyzer.py
```
`test_report.py` runs four user-readable scenarios and compares actual analyzer results with expected values. `test_analyzer.py` runs six automated unit tests. `TEST_RESULTS.txt` is a reminder/sample placeholder; regenerate results in your own environment rather than relying on a copied report.

## Functional programming techniques
- `map()` with a `lambda` computes daily kWh for each category.
- `filter()` with a `lambda` finds categories above the configured daily kWh threshold.
- `reduce()` with `lambda` sums daily consumption and potential savings.
- A list comprehension builds the names of high-consumption categories; report rows are built by iterating normalized category records.

## TCP client-server architecture
`client.py` serializes the household request as UTF-8 JSON followed by `\n`, sends it to `server.py`, reads one newline-terminated JSON response, and prints the report. The server binds only to loopback (`127.0.0.1`), port `50515`. Each connection carries one request and one response. Socket timeouts prevent indefinite waits. JSON parse failures and invalid input return an error response rather than a normal report. A local TCP connection should be tested on the target machine because firewall or environment restrictions may affect networking.

## Finite State Machine (FSM)
The analyzer records its workflow in `fsm_trace`.

| Current state | Condition/event | Next state |
|---|---|---|
| `WAITING_FOR_DATA` | Request received by analyzer | `DATA_CHECK` |
| `DATA_CHECK` | All required fields and records valid | `EFFICIENCY_ANALYSIS` |
| `DATA_CHECK` | Missing, malformed, or out-of-range data | `INVALID_INPUT` |
| `EFFICIENCY_ANALYSIS` | Calculations and symbolic example complete | `REPORT_READY` |

Successful trace: `WAITING_FOR_DATA → DATA_CHECK → EFFICIENCY_ANALYSIS → REPORT_READY`. Invalid analyzer input: `WAITING_FOR_DATA → DATA_CHECK → INVALID_INPUT`. TCP-level malformed JSON is rejected by the server and returned with an invalid-input trace.

## Equations and assumptions
- Daily energy (kWh) = power (W) × operating hours/day ÷ 1000.
- Billing-period energy = daily energy × billing days.
- Estimated cost = billing-period kWh × tariff per kWh.
- Savings cost = power (W) × reduced hours/day × billing days × tariff per kWh ÷ 1000.

The SymPy expression is formed from symbols for power, reduced hours, days, and tariff, then substituted with the first category's values. The aggregate savings sum the same formula across all categories. The sample category-level powers are estimates, not measured data. Tariff is assumed constant. Fixed charges, taxes, time-of-use prices, appliance duty cycles, and interactions between appliances are excluded. Reduction is capped at actual operating hours, so remaining usage cannot become negative. The threshold applies to daily kWh per category, not billing-period kWh.

For the built-in sample: daily energy is 7.34 kWh; 30-day energy is 220.20 kWh; estimated cost is ₹1,761.60. Proposed reductions save 46.50 kWh, or ₹372.00, leaving an estimated cost of ₹1,389.60. These are calculated sample values only.

## Algorithm / pseudocode
1. Receive JSON request.
2. Move FSM to `DATA_CHECK`; validate required fields, categories, numeric ranges, and duplicate names.
3. On validation error, return the error and `INVALID_INPUT` trace.
4. Move to `EFFICIENCY_ANALYSIS`; calculate daily kWh with `map`, and build category reports.
5. Calculate total energy and savings with `reduce`; identify high categories with `filter`.
6. Build and evaluate the symbolic savings expression with SymPy.
7. Return the report with `REPORT_READY` trace.

## Sample input
The default sample is defined by `DEFAULT_DATA` in `analyzer.py`. It contains four categories, a tariff of ₹8/kWh, 30 billing days, and a 3 kWh/day high-consumption threshold.

## Sample output (key values)
```text
Daily energy: 7.340 kWh
Billing-period energy: 220.20 kWh
Estimated cost: ₹1761.60
Potential savings: 46.50 kWh / ₹372.00
Estimated cost after reduction: ₹1389.60
FSM: WAITING_FOR_DATA -> DATA_CHECK -> EFFICIENCY_ANALYSIS -> REPORT_READY
```

## Test-case summary
The formatted report covers normal household data, zero reduction, invalid operating hours, and high-consumption detection. The unit suite additionally checks zero tariff and a missing required field. Always run both commands on your machine and inspect their actual output.

## Limitations and future improvements
This tool uses user-supplied estimates, not smart-meter readings. It does not model variable tariffs, taxes, fixed charges, seasonal changes, or appliance cycling. Future versions could import smart-meter CSV files, support time-of-day tariffs, compare multiple months, and provide recommendations based on category-specific efficiency targets.
