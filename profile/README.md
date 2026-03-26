# MADNESS (MADs - Natural Energy Scheduling System)

A decentralized energy management system (EMS) for rural off-grid habitations. This project implements a multi-layered distributed architecture using **C++** and the **MADS** framework to manage diverse energy sources, storage systems, and domestic loads through advanced state estimation and cooperative control.

---

## System Architecture

The system is modeled as a collection of independent nodes communicating via a peer-to-peer network. There is no central master controller; the global power balance emerges from local estimation and collective consensus.

### Node Types
* **Load Node (The House):** Monitors domestic consumption in real-time.
* **Generation Nodes:** Variable sources including Solar (PV), Wind (Turbines), and Hydro (Small-scale stream).
* **Storage Node:** Pumped-hydro storage (PHS) or battery systems acting as an energy buffer.

---

## Technical Workflow

The control loop follows a three-stage pipeline:

### 1. Local State Estimation (EKF)
Each generation and storage node runs a local **Extended Kalman Filter (EKF)** to estimate power output. Inspired by automotive slip-control systems, the filter dynamically adjusts its trust in sensors based on the environmental conditions.

### 2. Multi-Layered Consensus Protocol
Nodes reach a shared "Global Truth" via the **MADS**.

#### Layer A: Perception Consensus (Supply Estimation)
Nodes broadcast their estimated power ($P_i$) and uncertainty ($\sigma_i^2$).
* **Goal:** Agreement on the total available energy ($P_{total}$) in the micro-grid.

#### Layer B: Control Consensus (Distributed Dispatching)
Once $P_{total}$ is known, nodes negotiate **how** to distribute the power. This is not a simple 50/50 split; it is an optimization based on **Variance-Aware Dispatching**:

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
    * The Storage Node joins the consensus to provide the missing power ($P_{gap}$). If multiple storage units exist, they negotiate based on their **State of Charge (SoC)** to maintain ergodic balance across the battery/reservoir bank.
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

## System Logic Flow

The following flow describes the interaction between local estimation and global coordination:

1.  **Sensors** → Local readings provided to the node.
2.  **EKF** → Filters noise and handles asynchronous data.
3.  **Ergodic Supervisor** → Analyzes sensor history and adjusts the **R Matrix** (Confidence).
4.  **Consensus Layer 1** → Nodes exchange $(P, \sigma^2)$ to agree on total available energy.
5.  **Consensus Layer 2** → Nodes agree on the final power distribution (House vs. Storage).
6.  **Actuators** → Turbines and pumps adjust their physical state based on the consensus.

---

## Tech Stack
* **C++20**: Core logic and high-performance matrix math.
* **Eigen Library**: For EKF matrix operations and linear algebra.
* **MADS Framework**: For decentralized process communication.

## Practical Tests

## Experimental Validation

To validate the decentralized logic, the system is tested using a distributed hardware testbed. This setup moves beyond pure simulation by introducing real-world electronic noise, communication latency, and physical stochasticity.

### Hardware Setup
The testbed consists of multiple independent nodes connected via a local network:
* **Sensor Nodes (Raspberry Pi):** Each Pi represents a generation node (Wind, Solar, or Hydro). 
    * **Source Emulation:** Breadboard-mounted potentiometers are used to simulate real-time physical inputs (e.g., turbine RPM, solar irradiance, or water flow pressure).
    * **Data Acquisition:** Analog signals are processed through an **ADS1115 (16-bit ADC)** via I2C to introduce realistic quantization noise and sampling jitter.
* **Central Monitor & Load Node (PC/Laptop):** Acts as the "House" node and runs the global telemetry dashboard to monitor consensus convergence.


### Test Scenarios
The following tests are performed to stress-test the **Ergodic Supervisor** and the **Consensus Layers**:

#### 1. Sensor Noise & EKF Robustness
* **Action:** Introducing high-frequency electrical noise by manually jittering the potentiometers.
* **Expected Result:** The **EKF** should maintain a stable power estimate, filtering out transient spikes while updating the state covariance.

#### 2. Non-Ergodic Fault Injection (The "Broken Blade" Scenario)
* **Action:** Abruptly changing the potentiometer value to a state that contradicts the natural resource profile (e.g., zero RPM during high wind simulation).
* **Expected Result:** The **Ergodic Supervisor** detects a high Sobolev distance ($\mathcal{E}$), triggers a $K_{ergodic}$ penalty, and inflates the $R$ matrix. The node's weight in the **Control Consensus** should drop significantly, isolating the faulty source.

#### 3. Network Resilience & Latency
* **Action:** Simulating network congestion or physically disconnecting a Raspberry Pi.
* **Expected Result:** **MADS** must handle packet loss. The remaining nodes must re-negotiate the power dispatching in Layer 2 to compensate for the missing source, ensuring the "House" demand is still met.

### Telemetry and Data Flow
The experimental data flow follows this path:
`Physical Potentiometer` → `ADS1115 (ADC)` → `I2C Bus` → `Raspberry Pi (EKF + Ergodic Logic)` → `MADS Network (TCP/IP)` → `Global Consensus Result`.
