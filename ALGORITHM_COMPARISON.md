# Algorithm Comparison Guide

## Three Solving Algorithms

This project now supports three different algorithms for solving Sudoku puzzles:

### Algorithm 0: Standard Ant Colony System (ACS)

**Usage:**
```bash
./sudokusolver --alg 0 --file puzzle.txt --ants 10 --verbose
```

**Characteristics:**
- Single-threaded
- One colony of ants (default: 10)
- Pheromone-based metaheuristic
- Time-based termination (default: 120 seconds)
- Good for exploration

**Best for:**
- Quick solutions
- Single-core systems
- Baseline comparisons

---

### Algorithm 1: Backtracking Search

**Usage:**
```bash
./sudokusolver --alg 1 --file puzzle.txt --verbose
```

**Characteristics:**
- Deterministic algorithm
- Depth-first search with constraint propagation
- Guaranteed to find solution if one exists
- Fast for easy/medium puzzles
- Can be slow for very hard puzzles

**Best for:**
- Logic-solvable puzzles
- When you need guaranteed results
- Benchmarking exact methods

---

### Algorithm 2: Parallel Ant Colony System (NEW!)

**Usage:**
```bash
./sudokusolver --alg 2 --file puzzle.txt --subcolonies 6 --ants 30 --timeout 120 --verbose
```

**Characteristics:**
- Multi-threaded (default: 4 threads, configurable)
- Multiple independent sub-colonies
- Periodic communication between colonies
- Timeout-based OR first-solution termination (default 120 seconds)
- Stops as soon as ANY colony finds complete solution
- Adaptive communication intervals
- Diverse search strategies
- Deadlock-free synchronization

**Best for:**
- Hard puzzles (45% or less fixed cells)
- Multi-core systems
- Research on parallel metaheuristics
- When standard ACS struggles
- 25×25 and 16×16 puzzles

---

## Feature Comparison Table

| Feature | Algorithm 0 (ACS) | Algorithm 1 (Backtrack) | Algorithm 2 (Parallel ACS) |
|---------|-------------------|-------------------------|----------------------------|
| **Parallelism** | No | No | Yes (configurable) |
| **Deterministic** | No | Yes | No |
| **Guaranteed Solution** | No | Yes | No |
| **Number of Ants** | 10 (configurable) | N/A | 30-40 recommended per colony |
| **Termination** | Time-based (120s) | Complete/Fail | First solution or timeout (120s) |
| **Pheromones** | Single matrix | N/A | Multiple independent matrices |
| **Communication** | N/A | N/A | Ring + Random topology |
| **Memory Usage** | Low | Low | Medium (N× pheromones) |
| **CPU Usage** | 1 core | 1 core | N cores (N=subcolonies) |
| **Best For** | Simple puzzles | Logic-solvable | Hard puzzles |

---

## Performance Comparison

### Speed (9×9 puzzles)

**Easy puzzles:**
1. Algorithm 1 (Backtrack) - **Fastest** (~0.01s)
2. Algorithm 0 (ACS) - Fast (~0.1s)
3. Algorithm 2 (Parallel ACS) - Fast (~0.1s, overhead from threading)

**Medium puzzles:**
1. Algorithm 1 (Backtrack) - Fast (~0.5s)
2. Algorithm 2 (Parallel ACS) - **Faster than ACS** (~1s)
3. Algorithm 0 (ACS) - Slower (~3s)

**Hard puzzles:**
1. Algorithm 2 (Parallel ACS) - **Best** (~5s)
2. Algorithm 0 (ACS) - Slower (~20s or timeout)
3. Algorithm 1 (Backtrack) - Slow (~30s or more)

---

## When to Use Each Algorithm

### Use Algorithm 0 (Standard ACS) when:
- You have a single-core system
- You want a simple metaheuristic baseline
- Puzzle is not too difficult
- You want to understand basic ACS behavior

### Use Algorithm 1 (Backtracking) when:
- Puzzle is logic-solvable
- You need guaranteed correctness
- You're comparing against exact methods
- Puzzle is easy to medium difficulty

### Use Algorithm 2 (Parallel ACS) when:
- You have a multi-core system (2+ cores)
- Puzzle is very difficult
- Standard ACS times out or struggles
- You want to leverage parallel processing
- You're researching parallel metaheuristics
- You want best solution quality

---

## Parameter Recommendations

### For Algorithm 0 (ACS)
```bash
# Quick attempt (easy puzzles)
--alg 0 --ants 10 --timeout 10

# Standard (medium puzzles)
--alg 0 --ants 20 --timeout 60

# Intensive (hard puzzles)
--alg 0 --ants 50 --timeout 300
```

### For Algorithm 1 (Backtracking)
```bash
# Standard usage
--alg 1 --timeout 60

# Long timeout for hard puzzles
--alg 1 --timeout 300
```

### For Algorithm 2 (Parallel ACS)
```bash
# Quick attempt (easy puzzles) - not recommended, use alg 0 or 1
--alg 2 --subcolonies 4 --ants 10 --timeout 30

# Standard (medium puzzles, 40-50% fixed)
--alg 2 --subcolonies 4 --ants 30 --timeout 120

# Intensive (hard puzzles, 35-45% fixed)
--alg 2 --subcolonies 6 --ants 30 --timeout 120

# Maximum effort (very hard puzzles, <35% fixed)
--alg 2 --subcolonies 8 --ants 40 --timeout 180

# Recommended for 25x25 @ 45% fixed (best tested config)
--alg 2 --subcolonies 6 --ants 30 --timeout 120
# Result: 90% success rate, 14s average time
```

---

## Example Comparisons

### Test Case: Platinum Blond (9×9, logic-solvable)

```bash
# Algorithm 1: Backtracking
./sudokusolver --alg 1 --file instances/logic-solvable/platinumblond.txt --verbose
# Result: ~0.01 seconds (FASTEST)

# Algorithm 0: Standard ACS
./sudokusolver --alg 0 --file instances/logic-solvable/platinumblond.txt --ants 10 --verbose
# Result: ~0.5 seconds

# Algorithm 2: Parallel ACS (not recommended for easy puzzles)
./sudokusolver --alg 2 --file instances/logic-solvable/platinumblond.txt --subcolonies 4 --ants 10 --timeout 30 --verbose
# Result: ~0.3 seconds (overhead from threading makes it slower)
```

### Test Case: Hard 9×9 (30% fixed cells)

```bash
# Algorithm 1: Backtracking
./sudokusolver --alg 1 --file instances/general/inst9x9_30_1.txt --verbose
# Result: ~15 seconds or timeout

# Algorithm 0: Standard ACS
./sudokusolver --alg 0 --file instances/general/inst9x9_30_1.txt --ants 40 --verbose
# Result: ~30 seconds or timeout

# Algorithm 2: Parallel ACS
./sudokusolver --alg 2 --file instances/general/inst9x9_30_1.txt --subcolonies 4 --ants 30 --timeout 120 --verbose
# Result: ~10 seconds (BEST)
```

### Test Case: 16×16 puzzle

```bash
# Algorithm 1: Backtracking (slow but reliable)
./sudokusolver --alg 1 --file instances/general/inst16x16_45_10.txt --verbose
# Result: Variable (30-120 seconds)

# Algorithm 2: Parallel ACS (recommended)
./sudokusolver --alg 2 --file instances/general/inst16x16_45_10.txt --subcolonies 6 --ants 30 --timeout 120 --verbose
# Result: Better performance on multi-core systems
```

---

## Experimentation Tips

### Finding Optimal Parameters

1. **Start with defaults**
   ```bash
   ./sudokusolver --alg 2 --file yourpuzzle.txt --verbose
   ```

2. **Increase sub-colonies for more cores**
   ```bash
   # If you have 8 cores:
   --subcolonies 8
   ```

3. **Increase ants for harder puzzles (IMPORTANT!)**
   ```bash
   --ants 30-40  # Essential for strong pheromone reinforcement
   # With only 10 ants, each colony is too weak (86% success)
   # With 30 ants per colony, much better (90% success)
   ```

4. **Adjust timeout based on difficulty**
   ```bash
   --timeout 180  # More time for very hard puzzles
   --timeout 60   # Less time for medium puzzles
   ```

5. **Tune ACS parameters**
   ```bash
   --q0 0.95    # More exploitation (greedy)
   --rho 0.85   # Slower pheromone evaporation
   ```

### Benchmarking

To compare all three algorithms on the same puzzle:

```bash
echo "Testing puzzle.txt with all algorithms"
echo "========================================"

echo "Algorithm 0 (ACS):"
time ./sudokusolver --alg 0 --file puzzle.txt --ants 40 --timeout 120

echo "Algorithm 1 (Backtracking):"
time ./sudokusolver --alg 1 --file puzzle.txt --timeout 120

echo "Algorithm 2 (Parallel ACS):"
time ./sudokusolver --alg 2 --file puzzle.txt --subcolonies 6 --ants 30 --timeout 120
```

---

## Conclusion

- **Algorithm 0**: Simple, baseline ACS
- **Algorithm 1**: Exact, deterministic, best for easy puzzles
- **Algorithm 2**: Parallel, adaptive, best for hard puzzles on multi-core systems

Choose based on your:
- Hardware (single-core vs multi-core)
- Puzzle difficulty
- Time constraints
- Research objectives


