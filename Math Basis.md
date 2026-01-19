"""
ANNEX_Z_MATHEMATICAL_BASIS_FOR_SIL_T
------------------------------------

This module encodes the mathematical basis for SIL(t) using
time-dependent hazard rates, reliability functions, loop aggregation,
and SIL band mapping.

All equations are expressed in code or code-style pseudocode.
"""


# ---------------------------------------------------------------
# Z.1 TIME-DEPENDENT HAZARD RATE
# ---------------------------------------------------------------

def lambda_i(t, degradation_state):
    """
    Time-dependent hazard rate for component i.

    lambda_i(t) = f(age(t), cycles(t), environment(t),
                    stress(t), maintenance(t),
                    diagnostics(t), degradation(t))

    Inputs:
        t : time [hours]
        degradation_state : dict containing time-dependent factors

    Returns:
        lambda_i(t) : float
    """
    age = degradation_state["age_hours"](t)
    cycles = degradation_state["cycles"](t)
    env = degradation_state["environment_factor"](t)
    stress = degradation_state["stress_factor"](t)
    maint = degradation_state["maintenance_quality"](t)
    diag = degradation_state["diagnostic_coverage"](t)

    # Example multiplicative model (placeholder)
    return base_lambda_i * env * stress / max(maint, 1e-3)


# ---------------------------------------------------------------
# Z.2 RELIABILITY AND UNRELIABILITY
# ---------------------------------------------------------------

def reliability_i(t, lambda_func):
    """
    Reliability of component i over [0, t]:

        R_i(t) = exp( - ∫_0^t lambda_i(τ) dτ )

    Numerical integration required.
    """
    integral = integrate(lambda_func, 0, t)
    return math.exp(-integral)


def unreliability_i(t, lambda_func):
    """
    F_i(t) = 1 - R_i(t)
    """
    return 1.0 - reliability_i(t, lambda_func)


# ---------------------------------------------------------------
# Z.3 COMPONENT-LEVEL PFDavg(t) AND PFH(t)
# ---------------------------------------------------------------

def pfdavg_i(t, lambda_DU_func, proof_test_interval):
    """
    Exact low-demand PFDavg:

        PFDavg_i(t) = (1/T) * ∫_0^T [1 - exp(-∫_0^u lambda_DU(τ)dτ)] du

    Approximation (slowly varying lambda):

        PFDavg_i(t) ≈ lambda_DU(t) * T / 2
    """
    T = proof_test_interval
    lam = lambda_DU_func(t)
    return lam * T / 2.0


def pfh_i(t, lambda_DU_func):
    """
    High/continuous demand PFH:

        PFH_i(t) ≈ lambda_DU(t)
    """
    return lambda_DU_func(t)


# ---------------------------------------------------------------
# Z.4 LOOP-LEVEL AGGREGATION
# ---------------------------------------------------------------

def loop_lambda_series(t, lambda_funcs):
    """
    Series approximation:

        lambda_loop(t) = Σ lambda_i(t)
    """
    return sum(f(t) for f in lambda_funcs)


def loop_pfdavg_series(t, pfdavg_funcs):
    """
    Series approximation:

        PFDavg_loop(t) = Σ PFDavg_i(t)
    """
    return sum(f(t) for f in pfdavg_funcs)


def loop_pfh_series(t, pfh_funcs):
    """
    Series approximation:

        PFH_loop(t) = Σ PFH_i(t)
    """
    return sum(f(t) for f in pfh_funcs)


# ---------------------------------------------------------------
# Z.5 SIL(t) MAPPING
# ---------------------------------------------------------------

def sil_from_pfdavg(pfd):
    """
    SIL bands (IEC 61508 low demand):

        SIL 4: 1e-5 <= pfd < 1e-4
        SIL 3: 1e-4 <= pfd < 1e-3
        SIL 2: 1e-3 <= pfd < 1e-2
        SIL 1: 1e-2 <= pfd < 1e-1
        SIL 0: otherwise
    """
    if 1e-5 <= pfd < 1e-4:
        return 4
    if 1e-4 <= pfd < 1e-3:
        return 3
    if 1e-3 <= pfd < 1e-2:
        return 2
    if 1e-2 <= pfd < 1e-1:
        return 1
    return 0


def sil_from_pfh(pfh):
    """
    SIL bands (IEC 61508 high/continuous demand):

        SIL 4: 1e-9 <= pfh < 1e-8
        SIL 3: 1e-8 <= pfh < 1e-7
        SIL 2: 1e-7 <= pfh < 1e-6
        SIL 1: 1e-6 <= pfh < 1e-5
        SIL 0: otherwise
    """
    if 1e-9 <= pfh < 1e-8:
        return 4
    if 1e-8 <= pfh < 1e-7:
        return 3
    if 1e-7 <= pfh < 1e-6:
        return 2
    if 1e-6 <= pfh < 1e-5:
        return 1
    return 0


# ---------------------------------------------------------------
# Z.6 MISSION TIME
# ---------------------------------------------------------------

def loop_mission_time(component_mission_times):
    """
    Loop mission time:

        T_loop = min(T_1, T_2, ... T_n)
    """
    return min(component_mission_times)


# ---------------------------------------------------------------
# Z.7 SIL(t) VALIDITY
# ---------------------------------------------------------------

def sil_valid(sil_t, sil_target, t, t_loop):
    """
    SIL(t) is valid if:

        1. SIL(t) >= SIL_target
        2. t <= T_loop
    """
    return sil_t >= sil_target and t <= t_loop


# ---------------------------------------------------------------
# Z.8 FORMAL STATEMENT (CODE EDITION)
# ---------------------------------------------------------------

"""
SIL(t) is computed by:

1. Computing lambda_i(t) for each component.
2. Integrating lambda_i(t) to obtain R_i(t) and F_i(t).
3. Computing PFDavg_i(t) and PFH_i(t).
4. Aggregating component values into loop-level PFDavg(t) and PFH(t)
   using architecture-specific equations.
5. Mapping PFDavg(t) or PFH(t) to SIL(t) using IEC 61508 bands.
6. Validating SIL(t) against mission time and drift conditions.

All values are time-dependent and must be recalculated whenever
degradation, diagnostics, maintenance, or environmental conditions change.
"""
