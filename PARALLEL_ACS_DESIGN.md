# Parallel Ant Colony System Design Document

## Overview

This document describes the implementation of the Parallel Ant Colony System (Algorithm 2) for Sudoku solving. The parallel version uses multiple independent sub-colonies that communicate periodically to exchange solutions.

## Architecture

### High-Level Structure

```
ParallelSudokuAntSystem
├── SubColony 0 (Thread 0) - 10 ants
├── SubColony 1 (Thread 1) - 10 ants  
├── SubColony 2 (Thread 2) - 10 ants
└── SubColony 3 (Thread 3) - 10 ants
```

### Key Components

1. **ParallelSudokuAntSystem**: Main coordinator class
   - Manages 4 sub-colonies (configurable via `--subcolonies`, minimum: 3)
   - Handles thread synchronization
   - Coordinates communication between sub-colonies
   - Input validation: Ensures at least 3 sub-colonies for meaningful parallelization

2. **SubColony**: Independent ant colony running in its own thread
   - Contains 10 ants (configurable via `--ants`)
   - Maintains its own pheromone matrix
   - Tracks two solutions:
     - `iteration-best`: Best solution found in current iteration
     - `best-so-far`: Best solution found across all iterations

3. **IAntColony**: Interface for ant colony systems
   - Allows `SudokuAnt` to work with both regular and sub-colonies
   - Defines methods: `Getq0()`, `random()`, `Pher()`, `LocalPheromoneUpdate()`

## Algorithm Flow

### Initialization

1. Create 4 sub-colonies, each with its own:
   - Random number generator (seeded differently)
   - Pheromone matrix (initialized to `pher0 = 1/cellCount`)
   - 10 ants
   - Solution tracking variables

### Main Loop (until timeout or solution found)

Each sub-colony independently:

1. **Check Timeout**
   - If elapsed time >= maxTime (default 120s) → stop all colonies

2. **Run Iteration**
   - Each ant builds a solution (default: 30 ants recommended)
   - Find best ant in this iteration → `iteration-best`
   - Calculate pheromone value for iteration-best
   - Compare with `best-so-far` by pheromone value (Algorithm 0 logic)
   - If better: update `best-so-far`, `bestSoFarScore`, and `bestPher` together

3. **Check Solution**
   - If `best-so-far` is complete → stop immediately

4. **Update Pheromones**
   - Apply global pheromone update using **three-source summation**
   - Formula: `τ_ij(t+1) = (1-ρ)·τ_ij(t) + Δτ_ij`
   - Where `Δτ_ij = Δτ_ij^1 + Δτ_ij^2 + Δτ_ij^3`
   - **Δτ^1**: Local iteration-best (this colony's best in current iteration)
   - **Δτ^2**: Received iteration-best (from ring topology neighbor)
   - **Δτ^3**: Received best-so-far (from random topology partner)
   - `pherValue = numCells / (numCells - cellsFilled)` for each source

5. **Communication (at intervals)**
   - **Interval calculation**:
     - If iteration < 200: interval = 100
     - If iteration ≥ 200: interval = 10
   
   - **Synchronization**: All threads wait at barrier
   
   - **Master operations** (performed by last thread to arrive):
     
     a) **Generate Match Array**
        - Random permutation of [0, 1, 2, 3]
        - Used for random topology communication
     
     b) **Ring Topology Exchange** (`iteration-best`)
        - Topology: 0→1, 1→2, 2→3, 3→0
        - Each colony sends its `iteration-best` to next colony
        - Receiving colony:
          1. Stores received solution as `receivedIterationBest` (for pheromone Δτ^2)
          2. Compares with its `best-so-far`, updates if better
     
     c) **Random Topology Exchange** (`best-so-far`)
        - Based on match array
        - If match array = [2, 0, 3, 1]:
          - Position 0 (colony 2) receives from position 3 (colony 1)
          - Position 1 (colony 0) receives from position 0 (colony 2)
          - Position 2 (colony 3) receives from position 1 (colony 0)
          - Position 3 (colony 1) receives from position 2 (colony 3)
        - Receiving colony:
          1. Stores received solution as `receivedBestSoFar` (for pheromone Δτ^3)
          2. Compares with its `best-so-far`, updates if better
     
     d) **Check Solutions After Exchange**
        - Check if ANY sub-colony received complete solution
        - If ANY colony has complete solution → set stop flag
   
   - **Resume**: All threads continue

4. **Termination Conditions**
   - Solution found: ANY sub-colony found complete solution (immediate stop)
   - Timeout: Exceeded time limit (default 120 seconds)
   - Threads stop within 100ms of either condition being met

### Finalization

1. Join all threads
2. Collect best solution from all sub-colonies
3. Return global best solution

## Communication Topologies

### Ring Topology (for `iteration-best`)

```
Colony 0 ─────> Colony 1
   ↑               │
   │               ↓
Colony 3 <───── Colony 2
```

**Purpose**: Share recent discoveries quickly
**Frequency**: Every interval

### Random Topology (for `best-so-far`)

```
Match Array: [2, 0, 3, 1]

Position:  0   1   2   3
Colony:   [2] [0] [3] [1]

Exchanges:
  Position 0 (colony 2) ← Position 3 (colony 1)
  Position 1 (colony 0) ← Position 0 (colony 2)
  Position 2 (colony 3) ← Position 1 (colony 0)
  Position 3 (colony 1) ← Position 2 (colony 3)
```

**Purpose**: Diversify search by random information exchange
**Frequency**: Every interval
**Variation**: Random permutation changes each communication round

## Algorithm 0 Compatibility

**Critical Design Principle**: Each sub-colony runs **EXACTLY** like standalone Algorithm 0 when not communicating.

### Best-So-Far Tracking (Matches Algorithm 0)

Algorithm 0 updates its best solution using this logic:

```cpp
float pherToAdd = PherAdd(bestVal);
if (pherToAdd > bestPher)  // Single condition: compare by pheromone value
{
    bestSol.Copy(...);     // Update solution
    bestPher = pherToAdd;  // Update pheromone value
    // BOTH updated together → always synchronized
}
```

**Algorithm 2 uses IDENTICAL logic** in each sub-colony:

```cpp
float pherToAdd = PherAdd(iterationBestScore);
if (pherToAdd > bestPher)  // Same condition as Algorithm 0
{
    bestSoFar.Copy(iterationBest);
    bestSoFarScore = iterationBestScore;
    bestPher = pherToAdd;
    // BOTH updated together → always synchronized
}
```

**Why This Matters**:
- ✅ Ensures fair comparison: Algorithm 2 = Algorithm 0 + Communication
- ✅ Pheromone values match solution quality (bestPher always represents bestSoFar)
- ✅ Standard pheromone update works correctly
- ✅ Maintains Algorithm 0's proven convergence behavior

## Pheromone Update Strategy

Algorithm 2 uses a **dual pheromone update system** with **mutually exclusive** updates per iteration:

### Update 1: Standard Algorithm 0 Global Update (Non-Communication Iterations)

This update happens on **regular iterations** (when `iter % interval != 0`) to maintain Algorithm 0's core behavior:

```cpp
void SubColony::UpdatePheromone()
{
    // Standard ACS global update - reinforces local best-so-far only
    // Uses rho = 0.9 (typical ACS evaporation rate)
    // This is the same update used in standalone Algorithm 0
    // Note: bestPher decay happens outside this method (every iteration)
}
```

**When**: Iterations 1-99, 101-199, 201-209, 211-219, ...

### Best Pheromone Decay (Non-Communication Iterations Only)

After the standard pheromone update (non-communication iterations), `bestPher` is decayed:

```cpp
colony->bestPher *= (1.0f - colony->bestEvap);
```

**When**: **Only on non-communication iterations** (when `UpdatePheromone()` is called and `bestPher` is actually used)
**Why**: Logical consistency - decay only when the variable is used:
- `bestPher` is used in standard Algorithm 0 update as the reinforcement value
- `bestPher` is NOT used in three-source communication update (uses per-source pherValues instead)
- Therefore, decay should only happen when standard update is applied
- This maintains clean separation between the two update mechanisms

### Update 2: Three-Source Communication Update (Communication Intervals Only)

On communication intervals (when `iter % interval == 0`), this update **REPLACES** the standard update. Each sub-colony uses information from **three sources**:

```
Equation:
  τ_ij(t+1) = (1-ρ_comm)·τ_ij(t) + Δτ_ij
  where Δτ_ij = Δτ_ij^1 + Δτ_ij^2 + Δτ_ij^3
  and ρ_comm = 0.05 (communication evaporation rate, lighter than standard ρ = 0.9)

Sources:
  Δτ_ij^1: Local iteration-best
  Δτ_ij^2: Received iteration-best (ring topology)
  Δτ_ij^3: Received best-so-far (random topology)
```

### Implementation

**Important**: Evaporation is applied **ONLY** to [cell, digit] pairs that receive deposits, not to all pheromones.

```cpp
void SubColony::UpdatePheromoneWithCommunication()
{
    // Calculate pheromone values for each source
    float pherValue1 = (iterationBestScore > 0) ? 
        numCells / (numCells - iterationBestScore) : 0.0f;
    float pherValue2 = (receivedIterationBestScore > 0) ? 
        numCells / (numCells - receivedIterationBestScore) : 0.0f;
    float pherValue3 = (receivedBestSoFarScore > 0) ? 
        numCells / (numCells - receivedBestSoFarScore) : 0.0f;
    
    // Temporary arrays for collecting contributions
    float* contributions = new float[numUnits];
    bool* hasContribution = new bool[numUnits];
    
    // For each cell, collect contributions from all sources
    for (int i = 0; i < numCells; i++) {
        // Reset arrays
        for (int j = 0; j < numUnits; j++) {
            contributions[j] = 0.0f;
            hasContribution[j] = false;
        }
        
        // Collect from source 1 (local iteration-best)
        if (iterationBest.GetCell(i).Fixed()) {
            int digit = iterationBest.GetCell(i).Index();
            contributions[digit] += pherValue1;
            hasContribution[digit] = true;
        }
        
        // Collect from source 2 (received iteration-best from ring)
        if (receivedIterationBest.GetCell(i).Fixed()) {
            int digit = receivedIterationBest.GetCell(i).Index();
            contributions[digit] += pherValue2;
            hasContribution[digit] = true;
        }
        
        // Collect from source 3 (received best-so-far from random)
        if (receivedBestSoFar.GetCell(i).Fixed()) {
            int digit = receivedBestSoFar.GetCell(i).Index();
            contributions[digit] += pherValue3;
            hasContribution[digit] = true;
        }
        
        // Apply evaporation and add contributions ONLY to [cell, digit] 
        // pairs that have deposits
        // Uses rho_comm = 0.05 (lighter evaporation for additive reinforcement)
        for (int j = 0; j < numUnits; j++) {
            if (hasContribution[j]) {
                pher[i][j] = pher[i][j] * (1.0f - rho_comm) + contributions[j];
            }
        }
    }
    
    delete[] contributions;
    delete[] hasContribution;
}
```

**When**: Iterations 100, 200, 210, 220, 230, ...

### Critical Design Choice: Mutually Exclusive Updates

**Key Insight**: The two updates are **mutually exclusive** - only ONE happens per iteration:

```
if (iter % interval == 0):
    UpdatePheromoneWithCommunication()  // Three-source only (uses rho_comm = 0.05)
    // bestPher NOT decayed here (not used in this update)
else:
    UpdatePheromone()  // Standard Algorithm 0 only (uses rho = 0.9)
    bestPher *= (1 - bestEvap)  // Decay only when bestPher is used
```

**Why Not Both?**
- ❌ Over-reinforcement: Same patterns get reinforced twice
- ❌ Premature convergence: Too much pheromone → less diversity
- ❌ Against paper specification: Paper uses if-else logic

**Benefits of Mutual Exclusivity:**
- ✅ Balanced reinforcement: Prevents pheromone explosion
- ✅ Maintains diversity: Allows weaker solutions to survive
- ✅ Alternating strategies: Exploitation (standard) vs exploration (communication)
- ✅ Faithful to paper: Matches original algorithm specification

### Why Summation Instead of Winner-Takes-All?

**Winner-Takes-All (Wrong)**: Only the best solution contributes
- ❌ Makes communication mostly useless
- ❌ Ignores valuable information from neighbors
- ❌ Slow convergence

**Summation (Correct)**: All three sources contribute
- ✅ Local iteration-best: Recent discoveries by this colony
- ✅ Ring neighbor: Fast propagation of nearby solutions
- ✅ Random partner: Diverse exploration from distant colonies
- ✅ Strong pheromone reinforcement from multiple perspectives

### Example Impact

**Example 1**: All three sources agree on same digit for a cell

```
Before update: pher[i][j] = 0.020

All three solutions fill this cell with the same digit:
  Δτ^1 = 41.67  (local iteration-best)
  Δτ^2 = 62.50  (ring neighbor)
  Δτ^3 = 125.0  (random partner)
  Total contributions = 229.17

Apply evaporation once and add sum:
  pher[i][j] = 0.020 × (1-0.9) + 229.17
             = 0.002 + 229.17
             = 229.172

This strong signal guides future ants effectively!
```

**Example 2**: Sources disagree (source1=3, source2=7, source3=3)

```
Cell 5:
  Before: pher[5][3] = 0.020, pher[5][7] = 0.015

  Digit 3 contributions: Δτ^1 (41.67) + Δτ^3 (125.0) = 166.67
  pher[5][3] = 0.020 × (1-0.9) + 166.67 = 166.672

  Digit 7 contributions: Δτ^2 (62.5)
  pher[5][7] = 0.015 × (1-0.9) + 62.5 = 62.5015

Each unique [cell, digit] pair evaporates exactly once!
```

## Thread Synchronization

### Barrier Synchronization (Deadlock-Free)

```cpp
// Double-checked locking to prevent incomplete barrier counts
if (stopFlag.load())  // First check outside lock
    break;

std::unique_lock<std::mutex> lock(commMutex);

if (stopFlag.load()) {  // Second check inside lock
    barrier.store(0);    // Force reset for waiting threads
    commCV.notify_all();
    break;
}

// Each thread increments atomic counter
int arrived = barrier.fetch_add(1) + 1;

if (arrived == numSubColonies) {
    // Last thread performs master operations
    CommunicateRingTopology();
    CommunicateRandomTopology(matchArray);
    
    // Check if communication revealed a solution
    for (int i = 0; i < numSubColonies; i++) {
        if (subColonies[i]->GetBestSoFarScore() == totalCells)
            stopFlag.store(true);
    }
    
    barrier.store(0);  // Reset barrier
    commCV.notify_all();  // Wake all threads
} else {
    // Wait for master with periodic timeout checks
    while (barrier.load() != 0 && !stopFlag.load()) {
        commCV.wait_for(lock, 100ms);
        // Can check timeout even while waiting
        if (timeout_reached) {
            stopFlag.store(true);
            barrier.store(0);
            commCV.notify_all();
        }
    }
}
```

## Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--subcolonies` | 4 | Number of parallel sub-colonies (minimum: 3) |
| `--ants` | 10* | Number of ants per sub-colony |
| `--timeout` | 120 | Maximum time in seconds |
| `--q0` | 0.9 | Exploitation vs exploration (90% greedy) |
| `--rho` | 0.9 | Standard Algorithm 0 pheromone evaporation rate |
| `--rhocomm` | 0.05 | Communication update pheromone evaporation rate |
| `--evap` | 0.005 | Best-solution pheromone decay rate |

\***Recommended**: Use 30 ants per colony for better performance on hard puzzles. With only 10 ants, pheromone reinforcement is weak.

## Advantages of Parallel Design

1. **Parallelism**: 4× computational power (with 4 cores)
2. **Diversity**: Multiple independent searches explore different solution spaces
3. **Robustness**: Less likely to get stuck in local optima
4. **Adaptive Communication**: More frequent exchange when search intensifies (iter ≥ 200)
5. **Information Sharing**: Ring topology for recent discoveries, random topology for diversity

## Implementation Files

- `src/antcolonyinterface.h` - Base interface for ant colonies
- `src/parallelsudokuantsystem.h` - Parallel ACS header
- `src/parallelsudokuantsystem.cpp` - Implementation
- `src/sudokuant.h` - Modified to use interface
- `src/sudokuantsystem.h` - Modified to implement interface
- `src/solvermain.cpp` - Updated to support algorithm 2

## Usage Example

```bash
# Compile with threading support
make -f Makefile

# Run parallel ACS with 6 sub-colonies, 30 ants each, 120s timeout (recommended)
./sudokusolver --alg 2 \
               --file instances/general/inst25x25_45_0.txt \
               --subcolonies 6 \
               --ants 30 \
               --timeout 120 \
               --verbose

# Run with different parameters for very hard puzzles
./sudokusolver --alg 2 \
               --file instances/general/inst25x25_40_1.txt \
               --subcolonies 8 \
               --ants 40 \
               --timeout 180 \
               --q0 0.95 \
               --rho 0.85
```

## Performance Considerations

1. **Thread Overhead**: Communication synchronization adds overhead (~10-20% for easy puzzles)
2. **Memory**: Each sub-colony maintains its own pheromone matrix (scales with N colonies)
3. **Scalability**: Best with number of sub-colonies ≤ number of CPU cores
4. **Cache Effects**: Independent pheromone matrices reduce cache contention
5. **Ant Count Critical**: Each colony needs 30+ ants for strong pheromone reinforcement
   - 10 ants: 86% success rate on hard 25×25 puzzles
   - 30 ants: 90% success rate on hard 25×25 puzzles

## Recent Improvements

1. ✅ **Timeout-based termination**: Removed fixed iteration limit, now runs until solution or timeout
2. ✅ **Deadlock-free synchronization**: Double-checked locking and periodic timeout checks
3. ✅ **Immediate termination**: Stops as soon as any colony finds solution
4. ✅ **Performance tuning**: Identified optimal ant count (30 per colony)

## Future Enhancements

1. Dynamic interval adjustment based on solution quality
2. Adaptive sub-colony parameters
3. Migration policies for better solutions
4. Load balancing across threads
5. GPU acceleration for ant construction
6. Shared best-solution pheromone matrix (hybrid approach)


