# mannyai_technical_assesment
# Forward Planning Apparel Scheduling

This project simulates a simplified apparel production process with three sequential stages: **Cut → Sew → Pack**. The simulation considers multiple constraints such as machine availability, product-specific setup times, and additional delays. A Monte Carlo simulation is used to explore various scheduling permutations with the aim of minimizing overall lateness and maximizing on-time order completions.

---

## Table of Contents

1. [Overview](#overview)
2. [Project Structure](#project-structure)
3. [Step-by-Step Explanation](#step-by-step-explanation)
    - [Data Loading](#data-loading)
    - [Class Definitions](#class-definitions)
    - [Scheduling Core](#scheduling-core)
    - [Permutation Generator](#permutation-generator)
    - [Optimization Engine](#optimization-engine)
    - [Visualization Functions](#visualization-functions)
4. [Detailed Tables for Functions and Variables](#detailed-tables-for-functions-and-variables)
5. [Usage](#usage)
6. [License](#license)
7. [Contact](#contact)

---

## Overview

The simulation models the following production process:

1. **Cutting Stage:**  
   - Uses 2 cutting machines.
   - Incurs a 10-time-unit setup time if the product type changes.
2. **Sewing Stage:**  
   - Uses 3 sewing machines.
   - For some orders, a post-cutting delay of 48 time units is required.
3. **Packing Stage:**  
   - Uses 1 packing station.

Each order, represented by its product type, processing times, deadline, and delay flag, is scheduled through these stages. The simulation uses a forward-planning scheduler to assign orders to the first available machine and then runs a Monte Carlo simulation (with 100–500 iterations) to find the best schedule in terms of on-time completions and minimal lateness.

---

## Project Structure

- **Main Notebook (`version_01.ipynb`):**
  - **Data Loading:** Reads CSV data and converts required columns.
  - **Class Definitions:**  
    - `Order` for storing order attributes.
    - `Machine` for tracking machine availability and utilization.
  - **Scheduling Core:**  
    - `simulate_schedule(orders)` simulates the production process.
  - **Permutation Generator:**  
    - `generate_permutation(orders)` creates new order sequences using heuristics.
  - **Optimization Engine:**  
    - `optimize_schedules(data, iterations)` runs the Monte Carlo simulation.
  - **Visualization Functions:**  
    - Functions to generate timeline, progress, lateness distribution, and heuristic breakdown charts.

---

## Step-by-Step Explanation

### Data Loading

**Function: `load_data(csv_path)`**

- **Purpose:** Reads the CSV file containing order data and converts the delay flag to boolean.
  
**Table:**

| **Variable/Parameter** | **Description**                                          | **Logic/Purpose**                                                        |
|------------------------|----------------------------------------------------------|--------------------------------------------------------------------------|
| `csv_path`             | Path to the CSV file                                     | Specifies where to load the order data from.                           |
| `df`                   | DataFrame loaded using `pd.read_csv(csv_path)`           | Contains order data; the delay flag is converted from text to Boolean.   |

---

### Class Definitions

#### Order Class

**Purpose:** Encapsulates the attributes of each order.

**Table:**

| **Attribute**         | **Source (CSV Column)**           | **Description**                                    | **Logic/Purpose**                                                    |
|-----------------------|-----------------------------------|----------------------------------------------------|----------------------------------------------------------------------|
| `order_id`            | `order_id`                        | Unique identifier for each order.                | Used to track and reference orders.                                  |
| `product_type`        | `Product type`                    | Type of product for the order.                   | Determines if a setup time is needed on cutting machines.            |
| `cut_time`            | `cut time`                        | Time units required for cutting.                 | Used in scheduling the cutting stage.                                |
| `sew_time`            | `sew time`                        | Time units required for sewing.                  | Used in scheduling the sewing stage.                                 |
| `pack_time`           | `pack time`                       | Time units required for packing.                 | Used in scheduling the packing stage.                                |
| `deadline`            | `deadline`                        | Order delivery deadline (in time units).         | Used to compute lateness.                                              |
| `requires_delay`      | `requires_out_of_factory_delay`   | Boolean indicating if a post-cut delay is needed.  | Determines whether to add a 48-unit delay after cutting.             |

#### Machine Class

**Purpose:** Models machines for each production stage, tracking their availability and utilization.

**Table:**

| **Attribute**    | **Default/Initialization**           | **Description**                                        | **Logic/Purpose**                                                       |
|------------------|----------------------------------------|--------------------------------------------------------|-------------------------------------------------------------------------|
| `machine_id`     | Combination of type and ID (e.g., `Cut-1`) | Identifier for each machine.                          | Distinguishes machines and tracks order assignment.                   |
| `available_at`   | `0` (initially)                        | Next available time of the machine.                   | Updated after each task to schedule subsequent orders.                 |
| `last_product`   | `None` (initially)                     | Last product type processed by the machine.           | Determines if setup time is incurred when switching product types.     |
| `utilization`    | `[]` (empty list)                      | List of tuples with start and end times for tasks.    | Used later for visualizing machine usage in a timeline (Gantt chart).     |

---

### Scheduling Core

**Function: `simulate_schedule(orders)`**

- **Purpose:** Schedules each order through the Cut → Sew → Pack stages and computes lateness.
- **Process:**
  1. **Cutting Stage:**  
     - Evaluate each cutting machine.
     - Apply a 10-unit setup time if the product type changes.
     - Choose the machine with the earliest available start time.
  2. **Sewing Stage:**  
     - Adjust start time for a potential 48-unit delay.
     - Choose the earliest available sewing machine.
  3. **Packing Stage:**  
     - Schedule on the packing machine based on availability.
  4. **Lateness Calculation:**  
     - Compare finishing time with the deadline and compute lateness.

**Table:**

| **Variable/Parameter** | **Description**                                               | **Logic/Purpose**                                                                 |
|--------------------------|---------------------------------------------------------------|-----------------------------------------------------------------------------------|
| `orders`                 | List of `Order` objects                                       | The set of orders to be scheduled.                                                |
| `cutting_machines`       | List of 2 `Machine` objects                                   | Simulates the two cutting tables.                                                 |
| `sewing_machines`        | List of 3 `Machine` objects                                   | Simulates the three sewing stations.                                              |
| `packing_machine`        | Single `Machine` object                                       | Simulates the packing station.                                                    |
| `schedule`               | Dictionary mapping order IDs to schedule details              | Stores assignment and timing details for each order across stages.                |
| `lateness_summary`       | Dictionary mapping order IDs to lateness values               | Tracks lateness per order.                                                        |
| `total_lateness`         | Accumulated lateness for all orders                           | Provides an overall measure of schedule performance.                            |
| `on_time_count`          | Count of orders with zero lateness                            | Used to evaluate on-time performance.                                             |
| `cut_options`            | List of tuples (`start_time`, machine) for cutting stage      | Considers each machine’s availability and setup time to choose the earliest start.  |
| `sew_options`            | List of tuples (`start_time`, machine) for sewing stage       | Determines the earliest start times for sewing, considering post-cut delay.       |
| `lateness`               | `max(0, end_pack - order.deadline)`                           | Computes how late an order is (if at all).                                        |

---

### Permutation Generator

**Function: `generate_permutation(orders)`**

- **Purpose:** Creates different order sequences using three heuristics to introduce variability in the simulation.
- **Heuristics:**
  - **Deadline:** 60% chance (sorts by deadline).
  - **Product Type:** 20% chance (groups by product type).
  - **Processing Time:** 20% chance (sorts by total processing time).

**Table:**

| **Variable/Parameter** | **Description**                                           | **Logic/Purpose**                                                                |
|--------------------------|-----------------------------------------------------------|----------------------------------------------------------------------------------|
| `orders`                 | List of `Order` objects                                   | The original order sequence to be permuted.                                      |
| `r`                      | Random float between 0 and 1                              | Determines which heuristic is used for this permutation.                       |
| `sorted_orders`          | Orders sorted by the selected key (deadline, product, time) | Organizes orders by the chosen heuristic.                                        |
| `groups`                 | Dictionary grouping orders by heuristic key               | Allows shuffling within groups to add variability.                               |
| `shuffled`               | Final list of permuted orders                             | New order sequence returned for simulation.                                    |
| `heuristic`              | String indicating the heuristic used                    | Useful for later analysis and tracking of heuristic performance.               |

---

### Optimization Engine

**Function: `optimize_schedules(data, iterations=500)`**

- **Purpose:** Uses Monte Carlo simulation to iterate through multiple order permutations to find the schedule that minimizes lateness while maximizing on-time completions.
- **Process:**
  1. Convert CSV data into `Order` objects.
  2. For a fixed number of iterations:
     - Generate a permutation using `generate_permutation`.
     - Simulate the schedule using `simulate_schedule`.
     - Track and update the best performing schedule.
  3. Record history for on-time counts, total lateness, and used heuristics.

**Table:**

| **Variable/Parameter** | **Description**                                                     | **Logic/Purpose**                                                               |
|--------------------------|---------------------------------------------------------------------|---------------------------------------------------------------------------------|
| `data`                   | DataFrame containing order information                             | Raw input data for simulation.                                                  |
| `iterations`             | Number of iterations to run (default: 500)                           | Controls how many permutations are tested.                                    |
| `orders`                 | List of `Order` objects (converted from the DataFrame)               | Represents all orders to be scheduled.                                          |
| `best`                   | Dictionary tracking the best schedule and performance metrics        | Stores the best schedule found during simulation.                             |
| `on_time_history`        | List recording on-time order counts for each iteration               | Used for visualizing progress.                                                  |
| `total_lateness_history` | List recording the total lateness for each iteration                   | Used for visualizing progress.                                                  |
| `heuristic_history`      | List recording the heuristic used in each iteration                    | Provides insight into the effectiveness of different heuristics.                |
| `permuted`               | Permuted list of orders (generated by `generate_permutation`)          | Current order sequence for simulation.                                        |
| `schedule`, `lateness_map`, `total_lat`, `on_time`, `cut_util`, `sew_util`, `pack_util` | Output values from `simulate_schedule`             | Metrics and schedule details for the current simulation iteration.              |

---

### Visualization Functions

These functions create visual representations of the simulation results:

#### plot_machine_timeline(schedule)

**Purpose:**  
Creates a Gantt chart showing the utilization timeline for each machine.

**Table:**

| **Variable/Parameter** | **Description**                                     | **Logic/Purpose**                                                                 |
|--------------------------|-----------------------------------------------------|-----------------------------------------------------------------------------------|
| `schedule`               | Dictionary with scheduling details per order        | Used to group tasks by machine and display start and duration times graphically.   |

**Logic Flow:**  
- Aggregate tasks by machine.
- Plot horizontal bars representing each task’s duration.
- Set axis labels and chart title.

---

#### plot_optimization_progress(on_time_history, lateness_history)

**Purpose:**  
Plots the evolution of on-time order counts and total lateness over simulation iterations.

**Table:**

| **Variable/Parameter** | **Description**                                  | **Logic/Purpose**                                                                  |
|--------------------------|--------------------------------------------------|------------------------------------------------------------------------------------|
| `on_time_history`        | List of on-time order counts per iteration        | Visualizes how on-time performance improves over iterations.                      |
| `lateness_history`       | List of total lateness values per iteration         | Shows the trend in total lateness over the simulation.                             |

**Logic Flow:**  
- Plot on-time counts on the primary y-axis.
- Plot total lateness on a secondary y-axis.
- Label axes and display the combined progress chart.

---

#### plot_lateness_distribution(lateness_map)

**Purpose:**  
Visualizes lateness for each order using a bar chart.

**Table:**

| **Variable/Parameter** | **Description**                                     | **Logic/Purpose**                                                                 |
|--------------------------|-----------------------------------------------------|-----------------------------------------------------------------------------------|
| `lateness_map`           | Dictionary mapping order IDs to their lateness values | Displays lateness per order to identify which orders are most delayed.             |

**Logic Flow:**  
- Sort lateness values.
- Plot a bar chart with order IDs on the x-axis and lateness on the y-axis.
- Add a reference line at zero to indicate on-time completion.

---

#### plot_heuristic_breakdown(heuristic_history)

**Purpose:**  
Generates a pie chart showing the distribution of heuristics used during the simulation.

**Table:**

| **Variable/Parameter** | **Description**                                 | **Logic/Purpose**                                                                |
|--------------------------|-------------------------------------------------|----------------------------------------------------------------------------------|
| `heuristic_history`      | List of heuristic names used in each iteration   | Counts occurrences to display which heuristics were most frequently applied.      |

**Logic Flow:**  
- Count the frequency of each heuristic.
- Plot a pie chart to display the proportional use of each heuristic.
- Display the breakdown chart.

---

## Detailed Tables for Functions and Variables

The tables above provide a clear mapping between the code structure and its functionality. They detail the variables/parameters used in each function or class and describe the underlying logic and purpose.

---

## Usage

1. **Install Dependencies:**  
   Ensure you have the necessary Python packages installed (e.g., `pandas`, `numpy`, `matplotlib`, `tqdm`):
   ```bash
   pip install -r requirements.txt
