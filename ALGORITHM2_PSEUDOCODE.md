# Algorithm 2: Parallel Ant Colony System - Pseudocode

## High-Level Overview

```
Algorithm: Parallel Ant Colony System for Sudoku
Input: Puzzle (partially filled Sudoku board)
Parameters: nSubColonies, nAnts, maxTime, q0, ρ, evap
Output: Complete solution or best partial solution found
```

## Main Algorithm

```pseudocode
ALGORITHM ParallelACS(puzzle, nSubColonies, nAnts, maxTime, q0, ρ, ρ_comm, evap)
    // Input validation
    IF nSubColonies < 3 THEN
        PRINT "Warning: numSubColonies must be >= 3 for proper parallel execution"
        nSubColonies ← 3
    END IF
    
    // Initialization Phase
    FOR i = 0 TO nSubColonies - 1 DO
        subColony[i] ← CreateSubColony(i, nAnts, q0, ρ, ρ_comm, evap)
        subColony[i].InitializePheromones(puzzle)
    END FOR
    
    globalBest ← puzzle
    globalBestScore ← CountFixedCells(puzzle)
    StartTimer()
    
    // Create parallel threads
    FOR i = 0 TO nSubColonies - 1 DO
        thread[i] ← StartThread(SubColonyWorker, i, puzzle, maxTime)
    END FOR
    
    // Wait for all threads to complete
    FOR i = 0 TO nSubColonies - 1 DO
        WaitForThread(thread[i])
    END FOR
    
    // Collect best solution from all sub-colonies
    FOR i = 0 TO nSubColonies - 1 DO
        IF subColony[i].bestSolScore > globalBestScore THEN
            globalBest ← subColony[i].bestSol
            globalBestScore ← subColony[i].bestSolScore
        END IF
    END FOR
    
    RETURN (globalBestScore == totalCells, globalBest)
END ALGORITHM
```

## Sub-Colony Worker Thread

```pseudocode
PROCEDURE SubColonyWorker(colonyId, puzzle, maxTime)
    colony ← subColony[colonyId]
    colony.Initialize(puzzle)
    
    iteration ← 0
    WHILE stopFlag = FALSE DO
        // ==========================================
        // Check timeout
        // ==========================================
        IF ElapsedTime() >= maxTime THEN
            stopFlag ← TRUE
            NotifyAllThreads()
            BREAK
        END IF
        
        iteration ← iteration + 1
        
        // ==========================================
        // Phase 1: Ant Construction
        // ==========================================
        colony.RunIteration(puzzle)
        
        // ==========================================
        // Phase 2: Pheromone Update (Mutually Exclusive)
        // ==========================================
        interval ← CalculateInterval(iteration)
        
        IF iteration MOD interval == 0 THEN
            // ==========================================
            // Communication + Three-Source Pheromone Update
            // ==========================================
            
            // Check stopFlag before entering barrier
            IF stopFlag = TRUE THEN
                BREAK
            END IF
            
            // Synchronization barrier - wait for all threads
            WaitAtBarrier()
            
            IF IsLastThreadAtBarrier() THEN
                // Master thread operations
                matchArray ← GenerateRandomPermutation(0..nSubColonies-1)
                
                // Ring topology communication (iteration-best)
                ExchangeIterationBest()
                
                // Random topology communication (best-so-far)
                ExchangeBestSol(matchArray)
                
                // Check if any colony found solution during communication
                FOR i = 0 TO nSubColonies - 1 DO
                    IF subColony[i].bestSolScore == totalCells THEN
                        stopFlag ← TRUE
                        BREAK
                    END IF
                END FOR
                
                // Release all waiting threads
                ReleaseBarrier()
            ELSE
                // Slave threads wait with periodic timeout checks
                WHILE Barrier NOT released AND stopFlag = FALSE DO
                    WaitWithTimeout(100ms)
                    IF ElapsedTime() >= maxTime THEN
                        stopFlag ← TRUE
                        ForceBarrierReset()
                        NotifyAllThreads()
                    END IF
                END WHILE
            END IF
            
            // Three-Source Communication Pheromone Update
            // Uses: local iteration-best + received iteration-best + received best-so-far
            colony.UpdatePheromoneWithCommunication()
            
            // After communication, check if we should stop
            IF stopFlag = TRUE THEN
                BREAK
            END IF
        ELSE
            // ==========================================
            // Standard Algorithm 0 Global Pheromone Update
            // ==========================================
            // Uses: only local best-so-far
            colony.UpdatePheromoneStandard()
            
            // Decay Best Pheromone (only on non-comm iterations, when bestPher is used)
            bestPher ← bestPher × (1 - bestEvap)
        END IF
        
        
        // ==========================================
        // Check if solution found
        // ==========================================
        IF colony.bestSolScore == totalCells THEN
            stopFlag ← TRUE
            NotifyAllThreads()
            BREAK
        END IF
    END WHILE
END PROCEDURE
```

## Interval Calculation

```pseudocode
FUNCTION CalculateInterval(iteration) → integer
    // Adaptive communication frequency
    IF iteration < 200 THEN
        RETURN 100  // Infrequent early communication
    ELSE
        RETURN 10   // Frequent late communication
    END IF
END FUNCTION
```

## Ant Colony Iteration

```pseudocode
PROCEDURE RunIteration(puzzle)
    // Step 1: Initialize all ants with random starting cells
    FOR each ant IN antList DO
        startCell ← Random(0, numCells - 1)
        ant.InitSolution(puzzle, startCell)
    END FOR
    
    // Step 2: Each ant constructs a complete solution
    FOR cellStep = 0 TO numCells - 1 DO
        FOR each ant IN antList DO
            ant.StepSolution()
        END FOR
    END FOR
    
    // Step 3: Find best ant in this iteration
    iterationBest ← NULL
    iterationBestScore ← 0
    
    FOR each ant IN antList DO
        score ← ant.NumCellsFilled()
        IF score > iterationBestScore THEN
            iterationBest ← ant.GetSolution()
            iterationBestScore ← score
        END IF
    END FOR
    
    // Step 4: Update iteration-best
    this.iterationBest ← iterationBest
    this.iterationBestScore ← iterationBestScore
    
    // Step 5: Calculate pheromone value
    pherToAdd ← numCells / (numCells - iterationBestScore)
    
    // Step 6: Update best-so-far (EXACTLY like Algorithm 0)
    // Compare by pheromone value, update BOTH solution and bestPher together
    IF pherToAdd > bestPher THEN
        bestSol ← iterationBest
        bestSolScore ← iterationBestScore
        bestPher ← pherToAdd
    END IF
END PROCEDURE
```

## Ant Solution Construction

```pseudocode
PROCEDURE StepSolution()
    cell ← currentCell
    
    IF GetCell(cell).IsEmpty() THEN
        failCells ← failCells + 1
    ELSE IF NOT GetCell(cell).IsFixed() THEN
        // Cell has multiple possibilities - choose one
        
        IF Random(0,1) > q0 THEN
            // ========================================
            // EXPLOITATION (Greedy Selection)
            // ========================================
            bestValue ← NULL
            maxPheromone ← -∞
            
            FOR each possibleValue IN GetCell(cell) DO
                pheromone ← Pher(cell, possibleValue)
                IF pheromone > maxPheromone THEN
                    maxPheromone ← pheromone
                    bestValue ← possibleValue
                END IF
            END FOR
            
            SetCell(cell, bestValue)
            LocalPheromoneUpdate(cell, bestValue)
        ELSE
            // ========================================
            // EXPLORATION (Roulette Wheel Selection)
            // ========================================
            totalPheromone ← 0
            choices ← []
            
            FOR each possibleValue IN GetCell(cell) DO
                pheromone ← Pher(cell, possibleValue)
                totalPheromone ← totalPheromone + pheromone
                choices.Add(possibleValue, totalPheromone)
            END FOR
            
            randomValue ← Random(0, totalPheromone)
            
            FOR each choice IN choices DO
                IF choice.cumulativePheromone > randomValue THEN
                    SetCell(cell, choice.value)
                    LocalPheromoneUpdate(cell, choice.value)
                    BREAK
                END IF
            END FOR
        END IF
    END IF
    
    // Move to next cell (circular)
    currentCell ← (currentCell + 1) MOD numCells
END PROCEDURE
```

## Pheromone Updates

Algorithm 2 uses **THREE types of pheromone updates**:

### 1. Local Pheromone Update (after each ant choice)

```pseudocode
PROCEDURE LocalPheromoneUpdate(cell, value)
    // Reduce pheromone on used path (encourage diversity)
    pher[cell][value] ← pher[cell][value] × 0.9 + pher0 × 0.1
END PROCEDURE
```

### 2. Standard Algorithm 0 Global Update (non-communication iterations)

**IMPORTANT**: This update and Update 3 are **mutually exclusive** - only ONE happens per iteration.

```pseudocode
PROCEDURE UpdatePheromoneStandard()
    // Standard ACS global update using only local best-so-far
    // This maintains Algorithm 0's core behavior
    // Uses rho = 0.9 (standard ACS evaporation rate)
    FOR each [cell, digit] in bestSol DO
        pher[cell][digit] ← (1-ρ)·pher[cell][digit] + ρ·bestPher
    END FOR
    
    // Note: bestPher decay happens AFTER this procedure (only on non-comm iterations)
END PROCEDURE
```

### 3. Three-Source Communication Update (communication intervals only)

**IMPORTANT**: This update **replaces** the standard update on communication intervals. The pheromone update follows the equation:
```
τ_ij(t+1) = (1-ρ_comm)·τ_ij(t) + Δτ_ij
where Δτ_ij = Δτ_ij^1 + Δτ_ij^2 + Δτ_ij^3
and ρ_comm is the communication evaporation rate (typically 0.05)
```

Three sources contribute to pheromone deposits:
- **Δτ_ij^1**: Local iteration-best (this sub-colony's best solution in current iteration)
- **Δτ_ij^2**: Received iteration-best (from ring topology neighbor)
- **Δτ_ij^3**: Received best-so-far (from random topology partner)

**Important**: Evaporation is applied **only** to [cell, value] pairs that receive deposits, not to all pheromones.

```pseudocode
PROCEDURE UpdatePheromoneWithCommunication()
    // Calculate pheromone values for each source
    pherValue1 ← (iterationBestScore > 0) ? numCells / (numCells - iterationBestScore) : 0
    pherValue2 ← (receivedIterationBestScore > 0) ? numCells / (numCells - receivedIterationBestScore) : 0
    pherValue3 ← (receivedBestSolScore > 0) ? numCells / (numCells - receivedBestSolScore) : 0
    
    // For each cell, collect contributions from all sources
    FOR cell = 0 TO numCells - 1 DO
        // Track contributions for each unique digit at this cell
        contributions[numUnits] ← {0.0}
        hasContribution[numUnits] ← {false}
        
        // Collect from source 1 (local iteration-best)
        IF iterationBest.GetCell(cell).IsFixed() THEN
            digit ← iterationBest.GetCell(cell).Value()
            contributions[digit] ← contributions[digit] + pherValue1
            hasContribution[digit] ← true
        END IF
        
        // Collect from source 2 (received iteration-best from ring)
        IF receivedIterationBest.GetCell(cell).IsFixed() THEN
            digit ← receivedIterationBest.GetCell(cell).Value()
            contributions[digit] ← contributions[digit] + pherValue2
            hasContribution[digit] ← true
        END IF
        
        // Collect from source 3 (received best-so-far from random)
        IF receivedBestSol.GetCell(cell).IsFixed() THEN
            digit ← receivedBestSol.GetCell(cell).Value()
            contributions[digit] ← contributions[digit] + pherValue3
            hasContribution[digit] ← true
        END IF
        
        // Apply evaporation and add contributions ONLY to [cell, digit] pairs with deposits
        FOR digit = 0 TO numUnits - 1 DO
            IF hasContribution[digit] THEN
                pher[cell][digit] ← pher[cell][digit] × (1 - ρ_comm) + contributions[digit]
            END IF
        END FOR
    END FOR
END PROCEDURE
```

**Key features**:
- **Selective evaporation**: Only [cell, digit] pairs receiving deposits are evaporated
- **Contribution summation**: When multiple sources choose the same digit for a cell, their contributions are summed
- **Single evaporation**: Each unique [cell, digit] pair is evaporated exactly once, regardless of how many sources contribute
- **Meaningful communication**: All three sources contribute to pheromone reinforcement

## Communication Topologies

### Ring Topology (Iteration-Best Exchange)

```pseudocode
PROCEDURE ExchangeIterationBest()
    // Topology: 0→1, 1→2, 2→3, 3→0
    
    // Step 1: Collect all iteration-best solutions
    FOR i = 0 TO nSubColonies - 1 DO
        iterBest[i] ← subColony[i].GetIterationBest()
    END FOR
    
    // Step 2: Send to next colony in ring
    FOR i = 0 TO nSubColonies - 1 DO
        nextId ← (i + 1) MOD nSubColonies
        subColony[nextId].ReceiveIterationBest(iterBest[i])
    END FOR
END PROCEDURE
```

### Random Topology (Best-So-Far Exchange)

```pseudocode
PROCEDURE ExchangeBestSol(matchArray)
    // matchArray is a random permutation: [2, 0, 3, 1]
    // Colony at position i receives from position (i-1+n) mod n
    
    // Step 1: Collect all best-so-far solutions
    FOR i = 0 TO nSubColonies - 1 DO
        bsf[i] ← subColony[i].GetBestSol()
    END FOR
    
    // Step 2: Exchange according to match array
    FOR i = 0 TO nSubColonies - 1 DO
        colonyId ← matchArray[i]
        fromPos ← (i - 1 + nSubColonies) MOD nSubColonies
        fromColonyId ← matchArray[fromPos]
        
        subColony[colonyId].ReceiveBestSol(bsf[fromColonyId])
    END FOR
END PROCEDURE
```

### Receiving Solutions

```pseudocode
PROCEDURE ReceiveIterationBest(solution)
    // Store received iteration-best for the three-source pheromone update
    // Note: This does NOT update our own bestSoFar (which stays independent)
    this.receivedIterationBest ← solution
    this.receivedIterationBestScore ← solution.GetCellsFixed()
END PROCEDURE

PROCEDURE ReceiveBestSol(solution)
    // Store received best-so-far for the three-source pheromone update
    // Note: This does NOT update our own bestSol (which stays independent)
    this.receivedBestSol ← solution
    this.receivedBestSolScore ← solution.GetCellsFixed()
END PROCEDURE
```

## Termination Conditions

```pseudocode
The algorithm terminates when ANY of these conditions is met:

1. **Solution Found**: 
   - Any sub-colony finds complete solution (bestSoFarScore == totalCells)
   - Checked after every iteration
   - Also checked after communication phase

2. **Timeout Reached**:
   - ElapsedTime() >= maxTime (default 120 seconds)
   - Checked at start of each iteration
   - Also checked by threads waiting at barriers

Note: The old convergence check that waited for all colonies to converge
has been removed as it was redundant - we stop immediately when any colony solves.
```

## Key Variables and Data Structures

### Sub-Colony State
```
SubColony:
    - colonyId: integer               // Unique identifier
    - numAnts: integer                // Number of ants in this colony
    - antList: Array[Ant]             // The ants
    - pher[cell][value]: float        // Pheromone matrix
    - iterationBest: Board            // Best in current iteration
    - iterationBestScore: integer     // Score of iteration-best
    - bestSol: Board                  // Best ever found
    - bestSolScore: integer           // Score of best-so-far
    - q0, ρ, ρ_comm, pher0, evap: float  // ACS parameters
```

### Ant State
```
Ant:
    - sol: Board                      // Current working solution
    - currentCell: integer            // Current cell being filled
    - failCells: integer              // Number of unsolvable cells
    - parent: SubColony               // Reference to parent colony
```

### Synchronization
```
Global:
    - barrier: atomic integer         // Barrier counter
    - stopFlag: atomic boolean        // Stop signal
    - commMutex: mutex                // Communication mutex
    - commCV: condition_variable      // Synchronization
```

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| nSubColonies | 4 | Number of parallel sub-colonies (minimum: 3) |
| nAnts | 30* | Number of ants per sub-colony |
| maxTime | 120.0 | Maximum time in seconds before termination |
| q0 | 0.9 | Exploitation probability (0.9 = 90% greedy) |
| ρ (rho) | 0.9 | Standard Algorithm 0 pheromone evaporation rate |
| ρ_comm (rhocomm) | 0.05 | Communication update pheromone evaporation rate |
| evap | 0.005 | Best-solution pheromone decay rate |
| pher0 | 1/numCells | Initial pheromone level |

\* **Note**: While the old default was 10 ants, testing showed that 30 ants per colony provides much better performance (90% vs 86% success rate on hard puzzles). Each colony needs sufficient ants for strong pheromone reinforcement.

## Complexity Analysis

### Time Complexity (per iteration)
- **Ant construction:** O(nAnts × numCells × numUnits)
- **Pheromone update:** O(numCells)
- **Communication:** O(nSubColonies × numCells)
- **Total per iteration:** O(nAnts × numCells × numUnits)

### Space Complexity
- **Pheromone matrices:** O(nSubColonies × numCells × numUnits)
- **Solutions:** O(nSubColonies × numCells)
- **Total:** O(nSubColonies × numCells × numUnits)

For 25×25 puzzle with 4 sub-colonies:
- numCells = 625
- numUnits = 25
- Pheromone memory: 4 × 625 × 25 × 4 bytes = ~244 KB

## Example Execution Trace

### Initialization (4 sub-colonies, 10 ants each)
```
t=0: Create 4 sub-colonies
     Each colony initializes pheromones to pher0 = 1/625 = 0.0016
     Each colony creates 10 ants
```

### Iteration 1-99 (No communication)
```
Each colony independently:
  - 10 ants build solutions
  - Find iteration-best
  - Update best-so-far if better
  - Update pheromones
```

### Iteration 100 (First communication)
```
All colonies finish iteration 100
Wait at barrier
Master thread:
  1. Generate matchArray = [2, 0, 3, 1]
  2. Ring exchange: 0→1, 1→2, 2→3, 3→0 (iteration-best)
  3. Random exchange using matchArray (best-so-far)
  4. Check convergence: Not converged
  5. Release barrier
Continue to iteration 101
```

### Iteration 200 (Interval changes)
```
Communication interval changes: 100 → 10
Now communicate every 10 iterations
More frequent information exchange
```

### Iteration 573 (Solution Found - NEW BEHAVIOR)
```
Colony 2 finds complete solution (625/625 cells) in iteration 573

IMMEDIATE TERMINATION:
  - Colony 2 checks: bestSolScore == 625? YES!
  - Colony 2 sets stopFlag = TRUE
  - Colony 2 calls NotifyAllThreads()
  - All other colonies (at any iteration) see stopFlag = TRUE
  - All threads terminate within 100ms

Note: No need to wait for next communication!
Algorithm stops immediately when ANY colony succeeds.
Other colonies may still have incomplete solutions (e.g., 610/625).

If a solution is found DURING communication:
  - Master thread checks all colonies after exchange
  - If any has complete solution → stopFlag = TRUE
  - All threads exit after communication
```

## Summary

The Parallel Ant Colony System:

1. **Parallelizes** search across multiple independent colonies
2. **Maintains diversity** through independent pheromone matrices
3. **Shares information** periodically via two topologies:
   - Ring: Fast propagation of recent discoveries
   - Random: Diverse information exchange
4. **Adapts** communication frequency over time
5. **Terminates** as soon as ANY colony finds complete solution OR timeout is reached

This design balances **exploration** (independent search) with **exploitation** (sharing good solutions) to achieve robust performance on hard Sudoku puzzles.

**Key Features:**
- **Timeout-based termination**: No fixed iteration limit, runs until solution found or timeout
- **Immediate stop**: Stops as soon as any single colony solves the puzzle
- **Deadlock-free**: Double-checked locking and periodic timeout checks prevent deadlocks
- **Scalable performance**: 30 ants per colony recommended for strong pheromone reinforcement

