
***

# **AI-Driven VM Allocation in Google’s Cloud**
Link : https://research.google/blog/solving-virtual-machine-puzzles-how-ai-is-optimizing-cloud-computing/

## **Problem**

*   VM placement in data centers is like **multi-dimensional Tetris**:
    *   VMs arrive/depart unpredictably.
    *   Unknown lifetimes cause **resource stranding** and fewer **empty hosts**.
*   Empty hosts are critical for:
    *   **Maintenance & upgrades**
    *   **Large VM placement**
    *   **Energy savings**

***

## **Goals**
*   Create more **empty machiness** for easy management.
*   Reduce **incremental extension cost**
*   Reduce VM migration effort
*   Minimize resource stranding


## **Key AI Component**
*   Predict **probability distributions** of VM lifetimes using survival analysis.


*   **Machine Learning for Lifetime Prediction**:
    *   Predict **probability distributions** of VM lifetimes using survival analysis.
    *   Continuously **repredict** as VMs age.
    *   Feeds into scheduling algorithms.

***

## **Algorithms**

### **1. NILAS (Non-Invasive Lifetime-Aware Scheduling)**

*   **Goal**: Improve empty-host availability without major changes.
*   **Method**:
    *   Add a **lifetime-aware score** to Borg’s lexicographic host ranking.
    *   Align VMs with hosts of similar lifetime classes.
*   **Effect**: Hosts empty predictably; low-risk deployment.

***

### **2. LAVA (Lifetime-Aware VM Allocation)**

*   **Goal**: Reduce fragmentation and avoid “host lock-in.”
*   **Method**:
    *   Mix short-lived VMs under long-lived anchors.
    *   Score hosts by **minimizing incremental extension** of empty time.
*   **Effect**: More empty hosts, better large-VM obtainability.

***

### **3. LARS (Lifetime-Aware Rescheduling)**

*   **Goal**: Minimize migration cost during maintenance.
*   **Method**:
    *   Wait for VMs likely to exit soon.
    *   Migrate others to hosts where they don’t extend empty horizon.
*   **Effect**: Fewer disruptions, faster maintenance.

***

## **Technical Foundations**

*   **Lexicographic Scoring**:
    *   Multi-objective ranking: hard constraints → business goals → bin-packing → lifetime-aware tweaks.
*   **Survival Analysis**:
    *   Predict ( S(t) = P(T > t) ), update as VM survives longer.
    *   Enables dynamic, probabilistic scheduling decisions.

***

## **Impact**

*   **Empty hosts**: +2.3–9.2 percentage points.
*   **Resource stranding**: ↓ \~2–3%.
*   **Migration disruptions**: ↓ \~4.5%.

***

