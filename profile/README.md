# MADNESS (MADs - Natural Energy Scheduling System)

A decentralized energy management system for rural off-grid habitations. This project implements a multi-layered distributed architecture using **C++** and **MADS** framework to manage energy sources, storage systems, and domestic loads through distributed state estimation and cooperative control.

---

## System Architecture

The system is modeled as a collection of independent nodes communicating via a peer-to-peer network. There is no central master controller; the global power balance emerges from local estimation and collective consensus.

### Node Types
* **Load Node (The House):** Monitors domestic consumption in real-time and provides user's strategy.
* **Generation Nodes:** Variable sources including Solar (PV), Wind (Turbines), and Hydro.
* **Storage Node:** Battery systems acting as an energy buffer.

---

## Technical Workflow

The control loop follows a three-stage pipeline:

### 1. Local State Estimation (EKF)
Each generation and storage node runs a local **Extended Kalman Filter (EKF)** to estimate power output. Inspired by automotive slip-control systems, the filter dynamically adjusts its trust in sensors based on the environmental conditions.

### 2. Multi-Layered Consensus Protocol
Nodes reach a shared "Global Truth" via the **MADS**.

#### Layer A: Perception Consensus (Supply Estimation)
Nodes broadcast their estimated power ($P_i$) and uncertainty ($\sigma_i^2$).
* **Goal:** Agreement on the total available energy ($P_{total}$) and total weight ($W_{tot}$) in the grid.

#### Layer B: Control Consensus (Distributed Dispatching)
Once $P_{total}$ is known, nodes negotiate **how** to distribute the power. This is not a simple 50/50 split; it is a distributed weighted estimation of the total grid power that must balance the request:

* **Priority Routing:** The "House" requires stable, high-quality power. The "Storage" (pumping water) is resilient to fluctuations.
  
* **The Surplus Logic ($P_{total} > P_{house}$):**
In a surplus scenario, the system performs a "Quality-of-Source" partitioning. Instead of a simple proportional split, nodes negotiate based on their **Estimation Confidence ($\sigma^2$)**:

  + **Stable Source Prioritization (Low $\sigma^2$):** Nodes with low variance (e.g., a steady hydro stream or clear-sky solar) are assigned by the consensus to feed the **House** directly. 
    + *Implementation:* Each node $i$ proposes a share $P_{house,i}$ proportional to its weight $W_i = 1/\sigma_i^2$. 
    + *Result:* The domestic load receives "cleaner" energy, reducing voltage flickers and protecting sensitive electronics.

  + **Turbulent Source Buffering (High $\sigma^2$):** Nodes with high variance (e.g., gusty wind or nodes flagged by the **Ergodic Supervisor**) are directed to the **Storage Node**.
    + **The "Physical Low-Pass Filter":** The Pumped-Hydro system acts as a mechanical buffer. The high-frequency fluctuations (noise) of turbulent sources are absorbed by the inertia of the pumps and the water mass, effectively "low-pass filtering" the energy before it is stored as potential energy.

  + **The Negotiation Mechanism:** The nodes iteratively exchange their proposed contributions via MADS until the following equilibrium is reached:
    1.  **Balance Constraint:** $\sum P_{house,i} = P_{house}$.
    2.  **Saturations:** No node proposes more than its current EKF estimate $P_{est,i}$.
    3.  **Surplus Diversion:** Any remaining power $P_{storage,i} = P_{est,i} - P_{house,i}$ is automatically routed to the storage actuators.
   
* **Deficit Logic ($P_{total} < P_{house}$):**
    * The Storage Node joins the consensus to provide the missing power ($P_{gap}$). If multiple storage units exist, they negotiate based on their **State of Charge (SoC)** to maintain ergodic balance across the battery/reservoir bank. If the accumulator is not enough, then it is assumed to fetch power from the public grid.
---

## Ergodic Supervision and Adaptive R-Matrix

The core innovation of this project is the integration of **Ergodic Theory** into the distributed control loop. 

### Ergodicity
In natural environments, resources like wind and water flow follow stochastic processes. We assume these processes are **locally ergodic**, meaning their time averages reflect their statistical properties over a specific window.

### The Ergodic Feedback Loop
Each node features an **Ergodic Supervisor** that monitors the "statistical health" of the sensors:
1.  **Consistency Check:** It compares the current sensor readings with the long-term expected distribution (the ergodic profile of the site).
2.  **R-Matrix Adaptation:** If a sensor behaves in a non-ergodic way (e.g., due to a mechanical failure, sudden debris in the turbine, or extreme turbulence), the supervisor increases the **Measurement Noise Covariance (R)**.
    * **High R = Low Trust:** When $R$ increases, the EKF relies more on the physical model and less on the "dirty" sensor data.
3.  **Impact on Consensus:** Because the Consensus Layer weights each node's contribution based on its covariance, a node with high $R$ (low ergodic consistency) will automatically have less influence on the grid's decision-making, preventing the "propagation of error" to the whole system.

The idea is to keep a stabilized energy providence to the house consume.

This feature allows the environment to consider also the weather forecasts: the ergodic feedback is based on a past window until the current time step, but it can be expanded to consider a future window.

---

## Github Organization Structure
The organization contains all the used code. Agents (like energy sources: `wind_agent`, `hydro_agent`, `solar_agent`) have their own source code that born as a MADS plugin. The same goes for the `load_agent` that is the user-side interface.
All agents include some fundamental libraries:

* `negotiator_lib`: this is the library devoted to the distributed balance between energy sources whose distributed power is the balance of the total load request;
* `socialist_lib`: this is the library shared between loads that help to optimize loads' power requests;
* `ekf_lib`: it implements a virtual class `EKF` that implements the base functions such as the prediction and the correction, based on generic variables that are defined in a second moment in each specific agent by means of inheritance.

### .ini Configuration file example
```
[forecast_agent]
pub_topic = "forecast"

[wind_agent_1]
pub_topic = "source_wind_1"
sub_topic = ["omega_1", "forecast", "load_1", "load_2", "load_3", "load_4", "source_wind_2", "source_wind_3", "source_hydro_1", "source_solar_1", "accumulator_1"]

[wind_agent_2]
pub_topic = "source_wind_2"
sub_topic = ["omega_1", "load_1", "load_2", "load_3", "load_4", "forecast", "source_wind_1", "source_wind_3", "source_hydro_1", "source_solar_1", "accumulator_1"]

[wind_agent_3]
pub_topic = "source_wind_3"
sub_topic = ["omega_3", "load_1", "load_2", "load_3", "load_4", "forecast", "source_wind_1", "source_wind_2", "source_hydro_1", "source_solar_1", "accumulator_1"]

[hydro_agent_1]
pub_topic = "source_hydro_1"
sub_topic = ["omega_4", "load_1", "load_2", "load_3", "load_4", "forecast", "source_wind_1", "source_wind_2", "source_wind_3", "source_solar_1", "accumulator_1"]

[solar_agent_1]
pub_topic = "source_solar_1"
sub_topic = ["dc", "load_1", "load_2", "load_3", "load_4", "forecast", "source_wind_1", "source_wind_2", "source_wind_3", "source_hydro_1", "accumulator_1"]

[load_agent_1]
socialism = true
pub_topic = "load_1"
sub_topic = ["forecast", "load_3", "load_2", "load_4", "source_wind_1", "source_wind_2", "source_wind_3", "source_hydro_1", "source_solar_1", "accumulator_1"]

[load_agent_2]
socialism = true
pub_topic = "load_2"
sub_topic = ["forecast", "load_1", "load_3", "load_4", "source_wind_1", "source_wind_2", "source_wind_3", "source_hydro_1", "source_solar_1", "accumulator_1"]

[load_agent_3]
socialism = true
pub_topic = "load_3"
sub_topic = ["forecast", "load_1", "load_2", "load_4", "source_wind_1", "source_wind_2", "source_wind_3", "source_hydro_1", "source_solar_1", "accumulator_1"]

[load_agent_4]
socialism = true
pub_topic = "load_4"
sub_topic = ["forecast", "load_1", "load_2", "load_3", "source_wind_1", "source_wind_2", "source_wind_3", "source_hydro_1", "source_solar_1", "accumulator_1"]

[accumulator_1]
pub_topic = "accumulator_1"
sub_topic = ["forecast", "load_1", "load_2", "load_3", "load_4", "source_wind_1", "source_wind_2", "source_wind_3", "source_hydro_1", "source_solar_1"]


[rerunner]
sub_topic = "source_wind_1"
ketypaths = ["source_wind_1/state/proposed_power", "source_wind_1/state/covariance", "source_wind_1/state/p_max"]

[fmu_wind_turbine]
period = 100
pub_topic = "omega_1"
sub_topic = "source_wind_1"
relative_tol = 1e-4
absolute_tol = 1e-5
hmin_tol = 1e-10

[fmu_hydro_plant]
period = 100
pub_topic = "omega_4"
sub_topic = ["source_hydro_1"]
relative_tol = 1e-4
absolute_tol = 1e-5
hmin_tol = 1e-10

[fmu_solar_plant]
period = 100
pub_topic = "dc"
sub_topic = ["source_solar_1"]
relative_tol = 1e-4
absolute_tol = 1e-5
hmin_tol = 1e-10

```

