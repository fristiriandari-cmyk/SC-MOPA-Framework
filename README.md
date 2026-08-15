# SC-MOPA-Framework
Source code and empirical dataset for the SC-MOPA Framework for sustainable sharia tourism planning.
from pulp import (
    LpProblem,
    LpVariable,
    LpMinimize,
    lpSum,
    LpStatus,
    PULP_CBC_CMD,
    value
)


TOLERANCE = 1e-6


CRITERIA = {
    "C1":"Location",
    "C2":"Environmental Quality",
    "C3":"Infrastructure",
    "C4":"ICT Facilities",
    "C5":"Accessibility",
    "C6":"Halal Culinary",
    "C7":"Social Environment",
    "C8":"Regional Policy"
}
SC = {
    "C1":0.015,
    "C2":0.015,
    "C3":0.060,
    "C4":0.060,
    "C5":0.000,
    "C6":0.130,
    "C7":0.280,
    "C8":0.280
}
R = {
    "C1":3,
    "C2":3,
    "C3":6,
    "C4":4,
    "C5":5,
    "C6":3,
    "C7":4,
    "C8":2
}
TARGET = {
    c:1.0
    for c in CRITERIA
}

# ============================================================
# ACTIVE GOALS
#
# NOTE:
# Only criteria with SC>0 are counted in the
# Goal Achievement summary.
#
# The optimization model still includes ALL criteria.
# ============================================================

ACTIVE_CRITERIA = [
    c
    for c in CRITERIA
    if SC[c] > 0
]
TOTAL_ACTIVE_GOALS = len(ACTIVE_CRITERIA)
# ============================================================
# TOTAL REQUIRED INVESTMENT
# ============================================================
TOTAL_REQUIRED = sum(R.values())
# ============================================================
# BUDGET SCENARIOS
# ============================================================
SCENARIOS = {
    "50%":0.50*TOTAL_REQUIRED,
    "70%":0.70*TOTAL_REQUIRED,
    "90%":0.90*TOTAL_REQUIRED,
    "100%":1.00*TOTAL_REQUIRED
}
# ============================================================
# RESULT CONTAINER
# ============================================================
results = []
# ============================================================
# INITIAL INFORMATION
# ============================================================
print("="*70)
print("WEIGHTED GOAL PROGRAMMING BEHAVIOR VALIDATION")
print("="*70)
print()
print(f"Number of Criteria          : {len(CRITERIA)}")
print(f"Active Strategic Goals      : {TOTAL_ACTIVE_GOALS}")
print(f"Total Required Investment   : {TOTAL_REQUIRED}")
print()
print("Budget Scenarios")
print("-"*25)
for s,b in SCENARIOS.items():
    print(f"{s:>5} : {b:.2f}")
print()
print("Initialization completed.")
print("Ready to evaluate all scenarios.\n")
# ============================================================
# BUILD WEIGHTED GOAL PROGRAMMING MODEL
# ============================================================
def build_model(budget):
    """
    Construct the Weighted Goal Programming model.
    Parameters
    ----------
    budget : float
        Available investment budget.
    Returns
    -------
    model : LpProblem
    y : dict
        Investment decision variables.
    d_minus : dict
        Negative deviation variables.
    d_plus : dict
        Positive deviation variables.
    """
    model = LpProblem(
        "Weighted_Goal_Programming",
        LpMinimize
    )
    # --------------------------------------------------------
    # Decision Variables
    # --------------------------------------------------------
    y = {
        c: LpVariable(
            f"Investment_{c}",
            lowBound=0
        )
        for c in CRITERIA
    }
    d_minus = {
        c: LpVariable(
            f"d_minus_{c}",
            lowBound=0
        )
        for c in CRITERIA
    }
    d_plus = {
        c: LpVariable(
            f"d_plus_{c}",
            lowBound=0
        )
        for c in CRITERIA
    }
    # --------------------------------------------------------
    # Objective Function
    # --------------------------------------------------------
    model += lpSum(
        SC[c] * (d_minus[c] + d_plus[c])
        for c in CRITERIA
    )
    # --------------------------------------------------------
    # Goal Constraints
    # --------------------------------------------------------
    for c in CRITERIA:
        model += (
            y[c] / R[c]
            +
            d_minus[c]
            -
            d_plus[c]
            ==
            TARGET[c]
        )
    # --------------------------------------------------------
    # Budget Constraint
    # --------------------------------------------------------
    model += (
        lpSum(
            y[c]
            for c in CRITERIA
        )
        <=
        budget
    )
    # --------------------------------------------------------
    # Investment Upper Bound
    # --------------------------------------------------------
    for c in CRITERIA:
        model += (
            y[c]
            <=
            R[c]
        )
    return model, y, d_minus, d_plus
# ============================================================
# START VALIDATION
# ============================================================
print("=" * 70)
print("STARTING BEHAVIOR VALIDATION")
print("=" * 70)
for scenario, budget in SCENARIOS.items():
    print()
    print("=" * 70)
    print(f"SCENARIO {scenario}")
    print("=" * 70)
    # ========================================================
    # Build & Solve
    # ========================================================
    model, y, d_minus, d_plus = build_model(budget)
    solver = PULP_CBC_CMD(msg=False)
    model.solve(solver)
    status = LpStatus[model.status]
    objective = value(model.objective)
    # ========================================================
    # Investment Allocation
    # ========================================================
    investment = {}
    used_budget = 0.0
    for c in CRITERIA:
        investment[c] = value(y[c])
        used_budget += investment[c]
    unused_budget = budget - used_budget
    # ========================================================
    # Goal Achievement
    # ========================================================
    achievement = {
        c: investment[c] / R[c]
        for c in CRITERIA
    }
    achieved_goal = sum(
        1
        for c in ACTIVE_CRITERIA
        if achievement[c] >= (1 - TOLERANCE)
    )
    # ========================================================
    # Deviations
    # ========================================================
    negative_detail = {
        c: value(d_minus[c])
        for c in CRITERIA
    }
    positive_detail = {
        c: value(d_plus[c])
        for c in CRITERIA
    }
    negative_total = sum(
        negative_detail.values()
    )
    positive_total = sum(
        positive_detail.values()
    )
    # ========================================================
    # Objective Verification
    # ========================================================
    manual_objective = sum(
        SC[c]
        *
        (
            negative_detail[c]
            +
            positive_detail[c]
        )
        for c in CRITERIA
    )
    objective_error = abs(
        objective
        -
        manual_objective
    )
    # ========================================================
    # Constraint Verification
    # ========================================================
    max_constraint_error = 0.0
    for c in CRITERIA:
        lhs = (
            investment[c] / R[c]
            +
            negative_detail[c]
            -
            positive_detail[c]
        )
        err = abs(
            lhs
            -
            TARGET[c]
        )
        max_constraint_error = max(
            max_constraint_error,
            err
        )
    validation = (
        status == "Optimal"
        and
        objective_error <= TOLERANCE
        and
        max_constraint_error <= TOLERANCE
    )
    # ========================================================
    # DISPLAY RESULT
    # ========================================================
    print(f"Available Budget : {budget:.2f}")
    print(f"Used Budget      : {used_budget:.2f}")
    print(f"Unused Budget    : {unused_budget:.2f}")
    print(f"Solver Status    : {status}")
    print(f"Objective Value  : {objective:.6f}")
    print()
    print("Investment Allocation")
    print("-" * 55)
    for c in CRITERIA:
        print(
            f"{c:<3}"
            f"{CRITERIA[c]:30}"
            f"{investment[c]:>10.3f}"
        )
    print("-" * 55)
    print()
    print("Goal Achievement")
    print("-" * 55)
    for c in CRITERIA:
        print(
            f"{c:<3}"
            f"{achievement[c]:>10.4f}"
        )
    print("-" * 55)
    print()
    print(f"Total Negative Deviation : {negative_total:.6f}")
    print(f"Total Positive Deviation : {positive_total:.6f}")
    print()
    print("Verification")
    print("-" * 55)
    print(f"Manual Objective : {manual_objective:.6f}")
    print(f"Objective Error  : {objective_error:.12f}")
    print(f"Constraint Error : {max_constraint_error:.12f}")
    print(
        f"Goals Achieved   : "
        f"{achieved_goal}/{TOTAL_ACTIVE_GOALS}"
    )
    print(
        f"Validation       : "
        f"{'PASSED' if validation else 'FAILED'}"
    )
    # ========================================================
    # STORE RESULT
    # ========================================================
    results.append({
        "Scenario": scenario,
        "Budget": budget,
        "UsedBudget": used_budget,
        "UnusedBudget": unused_budget,
        "Status": status,
        "Objective": objective,
        "ManualObjective": manual_objective,
        "ObjectiveError": objective_error,
        "ConstraintError": max_constraint_error,
        "GoalsAchieved": achieved_goal,
        "NegativeDeviation": negative_total,
        "PositiveDeviation": positive_total,
        "Investment": investment,
        "Achievement": achievement,
        "NegativeDetail": negative_detail,
        "PositiveDetail": positive_detail
    })
print()
print("=" * 70)
print("ALL SCENARIOS COMPLETED")
print("=" * 70)
