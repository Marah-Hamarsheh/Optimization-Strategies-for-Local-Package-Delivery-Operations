# 🚚 Delivery Route Optimizer

A Python system that solves the **Capacitated Vehicle Routing Problem (CVRP)** — optimally assigning packages to vehicles and finding efficient delivery routes using metaheuristic algorithms.

## ✨ Features

- Assigns packages to vehicles respecting **weight capacity constraints**
- Optimizes routes to **minimize total travel distance**
- Respects **package priority** (1 = highest, 5 = lowest)
- Implements and compares two optimization algorithms:
  - **Simulated Annealing (SA)**
  - **Genetic Algorithm (GA)**
- Interactive **GUI visualization** via Tkinter + Matplotlib

## 🗂️ Project Structure

```
delivery-route-optimizer/
├── src/
│   └── main.py          # Core system — models, algorithms, GUI
├── data/
│   ├── packages.txt     # Package locations, weights, and priorities
│   └── vehicles.txt     # Vehicle capacities
└── docs/
    ├── report.pdf       # Project report
    └── assignment.pdf   # Original assignment brief
```

## 🚀 Getting Started

### Prerequisites

```bash
pip install matplotlib
```

> Tkinter ships with most Python installations. If missing: `sudo apt install python3-tk`

### Run

```bash
python src/main.py
```

### Input Format

**`data/packages.txt`** — one package per line: `x y weight priority`
```
# x,y coordinates [0-100], weight in kg, priority 1-5 (1 is highest)
50 10 30 1
70 15 15 2
```

**`data/vehicles.txt`** — one vehicle capacity per line (kg):
```
100
100
```

## ⚙️ How It Works

The system models the delivery problem as a CVRP and evaluates solutions using a weighted fitness function balancing:

| Factor | Description |
|---|---|
| Total distance | Minimize travel across all routes |
| Capacity violations | Penalize overloaded vehicles |
| Priority score | Deliver high-priority packages first |
| Load balance | Distribute packages evenly across vehicles |

Both **Simulated Annealing** and **Genetic Algorithm** explore the solution space to escape local optima and converge toward near-optimal routes.

## 🛠️ Tech Stack

- **Python 3**
- **Matplotlib** — route visualization
- **Tkinter** — desktop GUI

## 👥 Done by:

- Marah Hamarsheh
- Joud Thaher
