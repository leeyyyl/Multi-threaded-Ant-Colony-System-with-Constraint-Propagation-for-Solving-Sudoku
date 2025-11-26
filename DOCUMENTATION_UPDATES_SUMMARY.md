# Documentation Updates Summary

## Date: Current Session

This document summarizes all documentation updates made to reflect the latest implementation of Algorithm 2 (Parallel Ant Colony System).

## Key Changes Implemented in Code

### 1. **bestPher Decay Logic** ⚠️ IMPORTANT
**Change**: `bestPher` decay now only happens on non-communication iterations

**Before**:
```cpp
// Decay happened EVERY iteration
colony->UpdatePheromone() or colony->UpdatePheromoneWithCommunication()
colony->bestPher *= (1.0f - colony->bestEvap);  // Always
```

**After**:
```cpp
if (iter % interval == 0) {
    colony->UpdatePheromoneWithCommunication();
    // bestPher NOT decayed here (not used in this update)
} else {
    colony->UpdatePheromone();
    colony->bestPher *= (1.0f - colony->bestEvap);  // Only here
}
```

**Rationale**: 
- `bestPher` is used as reinforcement value in standard Algorithm 0 update
- `bestPher` is NOT used in three-source communication update (uses per-source pherValues)
- Therefore, logical to decay only when it's actually used
- Maintains clean separation between the two update mechanisms

---

### 2. **Input Validation** 🛡️
**Change**: Minimum 3 sub-colonies enforced

**Implementation** (src/parallelsudokuantsystem.cpp lines 314-320):
```cpp
// === INPUT VALIDATION ===
// Ensure at least 3 sub-colonies for meaningful parallel execution and communication
if (numSubColonies < 3)
{
    std::cerr << "Warning: numSubColonies must be >= 3 for proper parallel execution. Setting to 3." << std::endl;
    numSubColonies = 3;
}
```

**Rationale**:
- With < 3 colonies, communication topologies break down
- Ring: Need at least 3 for meaningful neighbor exchange
- Random: Need at least 3 for diverse pairings
- Prevents self-communication edge cases

---

### 3. **Parameter Separation: rho vs rho_comm** 📊
**Change**: Two distinct evaporation rates

| Parameter | Value | Usage |
|-----------|-------|-------|
| `rho` | 0.9 | Standard Algorithm 0 global update (non-communication iterations) |
| `rho_comm` | 0.05 | Three-source communication update (communication intervals) |

**Standard Update** (uses `rho = 0.9`):
```cpp
pher[i][j] = pher[i][j] * (1.0f - rho) + rho * bestPher;
```

**Communication Update** (uses `rho_comm = 0.05`):
```cpp
pher[i][j] = pher[i][j] * (1.0f - rho_comm) + contributions[j];
```

**Rationale**:
- Standard update: Strong reinforcement (90%) of single best solution
- Communication update: Light evaporation (5%) allows additive reinforcement from 3 sources
- Prevents over-reinforcement when combining multiple solutions

---

### 4. **Variable Naming Consistency** 📝
**Decision**: Keep `bestSolScore` in Algorithm 2 (NOT changed to `bestVal`)

**Algorithm 0**: Uses `bestVal` (member variable unused, local variable used)
**Algorithm 2**: Uses `bestSolScore` (member variable actively used)

**Rationale**:
- `bestSolScore` is more descriptive and self-documenting
- Clear for thesis presentation
- Algorithm 0 left untouched per user request
- Local loop variables can use `bestVal` (temporary scope)

---

## Documentation Files Updated

### ✅ ALGORITHM2_PSEUDOCODE.md
**Changes**:
- Added input validation pseudocode (lines 16-19)
- Moved `bestPher` decay inside `ELSE` block (non-communication only)
- Updated all `ρ` references to distinguish `ρ` vs `ρ_comm`
- Updated parameter table: `nSubColonies` minimum noted
- Clarified `UpdatePheromoneStandard()` procedure comments

**Key Sections**:
- Lines 14-20: Input validation and initialization
- Lines 131-138: Mutually exclusive pheromone updates
- Lines 303-316: Three-source communication update equation
- Lines 484-495: Parameter table with both `rho` and `rho_comm`

---

### ✅ PARALLEL_ACS_DESIGN.md
**Changes**:
- Updated "Best Pheromone Decay" section (non-communication only)
- Fixed equation to use `ρ_comm = 0.05` for communication update
- Added input validation note in architecture section
- Updated implementation code examples with correct `rho_comm` usage
- Updated parameter table

**Key Sections**:
- Lines 210-223: Best Pheromone Decay (corrected timing)
- Lines 228-238: Communication update equation with `ρ_comm`
- Lines 304-316: Mutually exclusive update logic
- Lines 427-437: Parameter table

---

### ✅ IMPLEMENTATION_SUMMARY.md
**Changes**:
- Updated "Core Features" - minimum 3 colonies noted
- Fixed pheromone update flow (bestPher decay timing)
- Updated "Best Pheromone Decay" section
- Corrected equation to use `ρ_comm`
- Updated parameter table
- Fixed code examples

**Key Sections**:
- Lines 9-14: Core features with input validation
- Lines 179-200: Algorithm flow with correct decay timing
- Lines 234-251: Best Pheromone Decay explanation
- Lines 256-263: Three-source equation with `ρ_comm`

---

### ✅ ALGORITHM2_EXPLANATION.md
**Changes**:
- Updated "Best Pheromone Decay" in Phase 4 breakdown
- Fixed all pheromone calculation examples to use `ρ_comm = 0.05`
- Updated equation notation throughout
- Corrected numerical examples (0.9 → 0.05 in calculations)

**Key Sections**:
- Lines 186-190: Best Pheromone Decay corrected
- Lines 193-239: Updated equation and examples
- Lines 219-233: Fixed numerical calculations
- Lines 488-515: Updated local vs global update examples

---

## Summary of Current Implementation

### Algorithm Flow (Correct as of this update):
```
FOR each iteration:
    1. Check timeout
    2. Run colony iteration (ant construction, solution tracking)
    
    3. Pheromone Update (MUTUALLY EXCLUSIVE):
       
       IF (iter % interval == 0):  // Communication interval
           a. Synchronize all threads (barrier)
           b. Exchange solutions (ring + random topologies)
           c. UpdatePheromoneWithCommunication() 
              - Uses ρ_comm = 0.05
              - Combines 3 sources (Δτ^1 + Δτ^2 + Δτ^3)
              - bestPher NOT decayed
       
       ELSE:  // Regular iteration
           a. UpdatePheromone()
              - Uses ρ = 0.9
              - Uses only local bestSol
              - bestPher IS decayed here: bestPher *= (1 - bestEvap)
    
    4. Check solution found
```

### Parameters (Current Defaults):
```
--subcolonies   4      (minimum: 3, enforced)
--ants          10     (recommended: 30)
--timeout       120    (seconds)
--q0            0.9    (exploitation probability)
--rho           0.9    (standard Algorithm 0 evaporation)
--rhocomm       0.05   (communication evaporation, NEW)
--evap          0.005  (bestPher decay rate)
```

---

## Verification Checklist

✅ All documentation files consistently describe:
- Input validation (minimum 3 colonies)
- bestPher decay timing (non-communication only)
- Two evaporation rates (rho vs rho_comm)
- Mutually exclusive pheromone updates
- Variable naming (bestSolScore in Alg 2)

✅ No references to "Equation 5" or "Equation 6" (per user request)

✅ All numerical examples use correct values:
- Standard update: ρ = 0.9
- Communication update: ρ_comm = 0.05

✅ Code implementation matches documentation:
- src/parallelsudokuantsystem.cpp
- src/parallelsudokuantsystem.h
- scripts/run_general.py

---

## Files Not Modified (Intentionally)

- **src/sudokuantsystem.cpp** - Algorithm 0 left untouched per user request
- **src/sudokuantsystem.h** - Algorithm 0 left untouched
- **README.md** - Main README (may need separate update if desired)
- **ALGORITHM_COMPARISON.md** - Comparative analysis (may need update)

---

## Testing Recommendations

After these documentation updates, verify:
1. ✅ Build compiles without errors
2. ✅ Algorithm 2 runs with default parameters
3. ✅ Input validation works (try `--subcolonies 1` or `2`)
4. ✅ Performance matches expectations on benchmark puzzles
5. ✅ Documentation accurately describes observable behavior

---

## Notes for Thesis Writing

### Key Points to Emphasize:

1. **Dual Pheromone Update System**
   - Two distinct mechanisms (Algorithm 0 + Communication)
   - Mutually exclusive (not additive) to prevent over-reinforcement
   - Different evaporation rates optimize for different goals

2. **Logical Consistency**
   - `bestPher` decay only when used (clean design principle)
   - Input validation prevents edge cases
   - Each sub-colony maintains Algorithm 0 behavior when not communicating

3. **Research Integrity**
   - Algorithm 2 = Algorithm 0 + Communication (true enhancement)
   - Fair comparison enabled by maintaining baseline behavior
   - Performance differences attributable to parallelization, not side effects

4. **Parameter Tuning**
   - `rho = 0.9`: Proven ACS value, used for local convergence
   - `rho_comm = 0.05`: Research contribution, enables multi-source blending
   - `numSubColonies ≥ 3`: Design constraint for topology validity

---

## Change Log

| Date | Component | Change | Rationale |
|------|-----------|--------|-----------|
| Current Session | bestPher decay | Moved to non-comm iterations only | Logical consistency - decay when used |
| Current Session | Input validation | Enforce min 3 colonies | Prevent topology edge cases |
| Current Session | Documentation | Updated all files | Reflect actual implementation |
| Current Session | Parameters | Clarified rho vs rho_comm | Clear separation of concerns |

---

**Documentation Status**: ✅ UP TO DATE

All documentation now accurately reflects the current implementation as of this session.

