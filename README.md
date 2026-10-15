# Shipment Tracker

A command-line app for tracking shipments, written in Python with SQLite. You can create shipments, move them through delivery statuses, search them, and see which ones are late or due soon.

It uses only the Python standard library, so there is nothing to install.

## Features

- Create a shipment and get a tracking code such as `SHP-00001`.
- Change a shipment's status. The app enforces the allowed order and rejects invalid moves.
- Every status change is saved in a history with a timestamp and an optional note.
- Search by tracking code, origin, or destination.
- List active shipments that are past their expected date.
- List the shipments that are due soonest.
- 14 unit tests.

## How to run

You need Python 3.10 or newer.

```bash
git clone https://github.com/YOUR-USERNAME/shipment-tracker.git
cd shipment-tracker

python -m tracker add Hyderabad Pune 2026-10-10
python -m tracker add Hyderabad Delhi 2026-10-30
python -m tracker status SHP-00001 in_transit --note "Left Hyderabad depot"
python -m tracker list
```

The data is stored in `shipments.db` in the folder where you run the command. Use `--db path/to/file.db` to choose another file.

## Commands

| Command | What it does |
|---|---|
| `add ORIGIN DESTINATION YYYY-MM-DD` | Create a shipment |
| `status CODE NEW_STATUS [--note TEXT]` | Change a shipment's status |
| `show CODE` | Show one shipment and its history |
| `list` | List all shipments |
| `search TEXT` | Find shipments by code, origin, or destination |
| `delayed` | Active shipments past their expected date |
| `due [-n 5]` | The shipments due soonest |

## Example

```
$ python -m tracker status SHP-00001 delivered
Error: Cannot move SHP-00001 from in_transit to delivered. Allowed: cancelled, out_for_delivery.

$ python -m tracker delayed
CODE       FROM       TO    STATUS      EXPECTED
---------  ---------  ----  ----------  ----------
SHP-00001  Hyderabad  Pune  in_transit  2026-10-10

$ python -m tracker show SHP-00001
SHP-00001: Hyderabad to Pune
Status:   in_transit
Expected: 2026-10-10
History:
  2026-10-15T04:52:35  created (Shipment created)
  2026-10-15T04:52:36  in_transit (Left Hyderabad depot)
```

## Status rules

A shipment can only move along these paths:

```
created ──> in_transit ──> out_for_delivery ──> delivered
   │            │                  │
   │            │                  └──> in_transit   (failed delivery attempt)
   └────────────┴──> cancelled
```

`delivered` and `cancelled` are final. These rules are stored as a dictionary in `tracker/db.py` (`ALLOWED_TRANSITIONS`), so changing them means editing one place.

## How it works

- **`tracker/db.py`** holds the `Tracker` class: all database access and business rules. It has no printing code, so it can be reused or tested on its own.
- **`tracker/cli.py`** parses the command line with `argparse` and prints results.
- **SQLite tables:** `shipments` stores the current state, and `status_history` stores every change.
- **Transactions:** creating a shipment writes to both tables inside one transaction, so a failure cannot leave half-written data.
- **Parameterised queries:** every SQL query passes values separately with `?`, which prevents SQL injection.
- **`due` command:** uses `heapq.nsmallest`, which finds the earliest dates without sorting the whole list.

## Run the tests

```bash
python -m unittest discover -v
```

The tests use an in-memory database, so they never touch your real data. They cover creating shipments, valid and invalid status changes, searching, the delayed and due reports, and the command-line interface.

## Ideas for extending it

- Add a `--format json` option to the `list` command.
- Add a carrier field and a report of delays per carrier.
- Import shipments from a CSV file.
- Put a small web API (Flask or FastAPI) in front of the `Tracker` class.
