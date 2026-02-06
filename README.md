
# 📦 Genetic Scheduling with Mesa + Genetic Algorithm

This document explains how the provided Python code uses a **Genetic Algorithm (GA)** to assign jobs to workers (agents in a Mesa simulation) so that a custom **objective value** is minimized. After finding a good assignment, the simulation runs to process jobs over time while tracking progress.

---

## 1) Configuration & Hyperparameters

- **Simulation constants**
  - `SIMULATION_STEPS = 100`: number of time steps to simulate.
  - `MODEL_SEED = 42`: random seed for reproducibility.
  - `MAX_JOB_LENGTH = 67`: maximum possible job size.
  - `JOB_COUNT = 30`: total number of jobs to assign.
  - `WORKER_COUNT = 5`: number of workers (agents).

- **Genetic Algorithm hyperparameters**
  - `POP_SIZE = 80`: population size (number of candidate solutions).
  - `GENERATIONS = 200`: number of evolution iterations.
  - `ELITE_COUNT = 2`: top individuals copied unchanged to the next generation.
  - `TOURNAMENT_K = 3`: size of tournament for parent selection.
  - `CROSSOVER_RATE = 0.9`: probability of applying crossover to selected parents.
  - `MUTATION_RATE = 0.05`: per-gene mutation probability (each job’s assignment may change worker).

---

## 2) Objective & Solution Evaluation

- **`objective(worker, jobs)`**  
  Computes a worker’s contribution to the total objective:  
  `int(sum(jobs) / worker.SALARY)` (heavier loads on higher-salary workers are “cheaper”).  
  If `SALARY` were zero (it isn’t here), it falls back to `int(sum(jobs))`.

- **`evaluate_solution(workers, JOB_LIST)`**  
  Adds up `objective(...)` across all workers. This is the **fitness** value the GA **minimizes**.

---

## 3) Representation (Chromosome) & Decoding

- **Chromosome (`assignments`)**: a list of length `JOB_COUNT`.  
  Each gene is an integer in `[0, WORKER_COUNT - 1]` indicating **which worker** gets that job.

- **`decode_to_joblist(assignments, jobs)`**  
  Converts `assignments` into `JOB_LIST` (list of lists), where `JOB_LIST[i]` contains the job sizes assigned to worker `i`.

- **`evaluate_chromosome(workers, jobs, assignments)`**  
  Decodes the chromosome and evaluates it with `evaluate_solution`.

---

## 4) GA Operators

- **Selection — `tournament_select(population, objectives, k)`**  
  Randomly picks `k` individuals and returns the one with the **lowest** (best) objective.

- **Crossover — `crossover_uniform(p1, p2, p=0.5)`**  
  Produces two children by swapping genes independently with probability `p` at each position.

- **Mutation — `mutate(assignments, rate)`**  
  For each gene, with probability `rate`, reassigns the job to a **different** worker (chosen at random).

> Note: If you see `&lt;` in the snippet, that’s HTML for `<`. In the Python source, it should be `<`.

---

## 5) Genetic Algorithm Loop (`genetic_algorithm`)

1. **Initialization**
   - Create a random population of `POP_SIZE` chromosomes.
   - Evaluate all individuals.
   - Track the global best individual and its objective.

2. **Per-generation metrics (`history`)**
   - `best`: best objective value this generation.
   - `mean`: mean objective of the population.
   - `std`: standard deviation of objectives.
   - `diversity`: ratio of unique individuals to population size.
   - `best_load_std`: standard deviation of per-worker total assigned load for the best individual (a balance proxy).

3. **Elitism**
   - Sort by objective and copy the top `ELITE_COUNT` individuals unchanged.

4. **Breeding**
   - Repeatedly:
     - Select two parents via tournament.
     - With probability `CROSSOVER_RATE`, apply uniform crossover; otherwise copy parents.
     - Mutate both children gene-wise with probability `MUTATION_RATE`.
     - Add children to the new population until it reaches `POP_SIZE`.

5. **Replacement & Tracking**
   - Replace the old population with the new one.
   - Recompute objectives.
   - Update the global best when improved.

6. **Result**
   - Decode the global best chromosome to `best_job_list`.
   - Return `(best_job_list, best_obj, history)`.

---

## 6) Mesa Agents & Simulation

### `Worker` (Agent)
- **State**
  - `JOBS`: a queue (list) of remaining job lengths.
  - `SALARY`: random integer in `[1, 3]`.
- **Step**
  - If there’s a job, reduce the first job by `SALARY`.
  - If it hits `<= 0`, remove it (job completed).

### `WareHouse` (Model)
- **Initialization**
  - Create a random `JOB_LIST` of `JOB_COUNT` jobs with sizes in `[0, MAX_JOB_LENGTH]`.
  - Create `WORKER_COUNT` workers.
  - Run the GA to get `best_job_list`; assign jobs to workers.
  - Store `best_obj_value` and `ga_history`.
  - Initialize simulation tracking (`step_history`) and take an initial snapshot.

- **`_snapshot()`** records per step:
  - `step`: current step counter.
  - `jobs_remaining_count`: total count of queued jobs across workers.
  - `total_work_remaining`: sum of all remaining job lengths.
  - `completed_this_step`: number of jobs completed since the last snapshot.
  - `per_worker_queue_lengths`: queue length for each worker.
  - Updates `_prev_jobs_count` for the next step.

- **`step()`**
  - Advance every worker by one step.
  - Increment the model step.
  - Take a new snapshot.

---

## 7) Running the Model

**`run_model(steps=SIMULATION_STEPS, seed=MODEL_SEED)`**:
- Seed the RNG, instantiate `WareHouse` (which runs the GA and assigns jobs),
- Advance the simulation for `steps`,
- Return:
  - `model` (contains `best_obj_value`, `ga_history`, `step_history`),
  - `ga_history` (per-generation GA metrics),
  - `step_history` (per-step simulation metrics).

Usage:
```python
model, ga_hist, step_hist = run_model()
model.best_obj_value  # The best objective found by the GA
