# Shift Scheduling Optimiser

A Python tool that builds a shift schedule for a part-time team from the employees' own availability requests. The planning period can be any number of whole weeks; a four-week period has roughly 90 shift places to fit to about ten people's requests.

The problem is formulated as an **integer program** and solved with Google OR-Tools' **CP-SAT** solver. The tool runs in Google Colab: the planner uploads an Excel file and downloads a finished schedule.

## Background

The tool was inspired by a real need: shift planning for a part-time team I worked in was done by hand from everyone's availability requests. The shift codes and rules below are that team's actual ones, which is why they are so specific. They are all set in the first code cell, so the same model can be adapted to a different shift structure.

## The problem

- Each day has a set of shifts that depends on the weekday (weekday evening shifts `I1–I3`, Saturday shifts `L1–L4`, Sunday/holiday shifts `S1–S3`, and an optional Monday shift `X` that may need several people).
- Each employee states their availability for each day in free text: "all shifts work", "day off", or a list of shift codes such as "I2 and I3 work".
- The schedule must follow labour rules (one shift per day, a weekly cap, and at least 11 hours of rest between consecutive shifts).
- When not every shift can be covered, the planner wants to know exactly which ones are left open, and the most important shifts should be the ones that get filled.

## Model

**Sets:** employees *E*, days *D*, shifts on day *d* *S(d)*, allowed shifts for employee *e* on day *d* *A(e,d) ⊆ S(d)*.

**Decision variables**

- `works[e,d,s] ∈ {0,1}`: employee *e* works shift *s* on day *d*. Created **only** for *s ∈ A(e,d)*, so availability requests are respected by construction.
- `unfilled[d,s] ∈ {0, …, demand[d,s]}`: places in shift *s* on day *d* that nobody covers.

**Constraints (all hard)**

1. Coverage: `Σ_e works[e,d,s] + unfilled[d,s] = demand[d,s]` for every day and shift
2. At most one shift per employee per day
3. At most `MAX_SHIFTS_PER_WEEK` (2) shifts per employee per Monday–Sunday week
4. 11-hour rest: for each forbidden pair (e.g. late `I3` followed by early `L1` the next day), `works[e,d,s₁] + works[e,d+1,s₂] ≤ 1`

**Objective**

Minimise `Σ cost(s) · unfilled[d,s]`, where regular shifts cost 10 and `X` shifts cost 1, so the solver gives up an `X` place before any regular shift.

## Example run

`example_input.xlsx` is an anonymised four-week planning period (11 employees, 28 days, one public holiday, 93 shift places in total):

| | |
|---|---|
| Solver status | OPTIMAL (proven best schedule) |
| Model size | 682 binary assignment variables |
| Solve time | < 0.1 s |
| Places filled | 85 / 93 (5 of the 8 open places are low-priority `X` places) |

The notebook also lists 8 availability entries it could not interpret (e.g. "evening shifts work", which matches no shift code) so the planner can check them, rather than guessing.

## How to use

1. Fill in the input Excel:
   - **First sheet** (calendar): one row per day; mark public holidays with `x`, optionally set the number of `X` shifts per day.
   - **One sheet per employee**, with a `Vuorotoive` (availability) cell for every day: `Kaikki käy` (all shifts), `Vapaatoive` (day off), or shift codes such as `I2 ja I3 käy`.
2. Open `shift_optimiser.ipynb` in Google Colab and upload the Excel file to the session.
3. Check that `INPUT_FILE` matches the file name and click **Run all**.
4. Download `vuorot_ratkaisu.xlsx`: one row per employee, one column per day, and the unfilled shifts listed in red below the schedule.

The planning period must start on a Monday and consist of whole weeks. Shift types, forbidden pairs, the weekly cap and priorities are set in the first code cell.

The user-facing parts (instructions, input labels, printed messages) are in Finnish because the tool was built for a Finnish team. The code and documentation are in English.

## Limitations and next steps

- **Fairness:** the objective only counts unfilled shifts, so the workload may be uneven between employees. A next version would add a term that balances shift counts.
- **Soft preferences:** requests are hard constraints. Preferences such as "I3 preferred" are currently ignored and could be added as weighted terms in the objective.
- **Free-text input:** availability is parsed with simple rules. A dropdown-based input sheet would remove most of the entries that need manual review.

## Tech

Python · Google OR-Tools (CP-SAT) · pandas · openpyxl · Google Colab
