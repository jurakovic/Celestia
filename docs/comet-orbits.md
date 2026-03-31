# Comet Orbit Implementation – Celestia 1.6.x Branch

## Goal

Add support for **parabolic** (e = 1) and **hyperbolic** (e > 1) comet orbits in Celestia 1.6.x. The stock 1.6.x `EllipticalOrbit` class only handled e < 1. Comets with open trajectories were silently ignored or broke rendering.

---

## Files Changed

### `src/celengine/orbit.h`

Added two virtual overrides to `EllipticalOrbit` that already existed on the base `Orbit` class but were not overridden, causing wrong behaviour:

```cpp
class EllipticalOrbit : public Orbit
{
public:
    // ... existing declarations ...
    double getPeriod() const;
    double getBoundingRadius() const;
    virtual bool isPeriodic() const;           // NEW
    virtual void getValidRange(double& begin, double& end) const; // NEW
    virtual void sample(double, double, int, OrbitSampleProc&) const;
};
```

### `src/celengine/orbit.cpp`

All changes are inside or adjacent to `EllipticalOrbit`.

#### 1. `eccentricAnomaly()` – extended to all conic sections

```
e == 0          → E = M  (circular)
0 < e < 0.2    → standard fixed-point iteration (5 steps)
0.2 ≤ e < 0.9  → Kepler/Meeus iteration (6 steps)
0.9 ≤ e < 1.0  → Laguerre-Conway elliptic (8 steps)
e == 1          → Barker's equation, Cardano closed-form (no iteration)
e > 1           → Laguerre-Conway hyperbolic (30 steps), sign-correct for inbound leg
```

**Parabolic (Barker's equation – Cardano closed form):**
```
D + D³/3 = M      where D = tan(ν/2)
W = 3M/2
Y = ∛(W + √(W² + 1))
D = Y − 1/Y
```

**Hyperbolic (Laguerre-Conway, sign fix):**
```cpp
double absM = fabs(M);
double E = log(2.0 * absM / eccentricity + 1.85);   // sign-correct initial guess
Solution sol = solve_iteration_fixed(SolveKeplerLaguerreConwayHyp(eccentricity, absM), E, 30);
return (M > 0.0) ? sol.first : -sol.first;           // restore sign for inbound leg
```
Without the sign fix, negative M (inbound leg) produced NaN.

#### 2. `positionAtE()` – extended to all conic sections

```
e < 1  (elliptic):   x = a(cos E − e),   y = a√(1−e²) sin E
e > 1  (hyperbolic): x = −a(e − cosh E), y = −a√(e²−1) sinh E,  a < 0 so x = pericenter at max
e == 1 (parabolic):  x = q(1 − D²),      y = 2qD
```

Note: for hyperbolic `a = pericenterDistance / (1 − e)` → a is negative since e > 1. The `−a(...)` signs produce positive x at pericenter, matching the elliptic convention.

#### 3. `velocityAtE()` – correct formulas for all conic sections

**Elliptic:**
```
ẋ = −a sin(E) · Ė,   ẏ = a√(1−e²) cos(E) · Ė
Ė = n / (1 − e cos E),   n = 2π/T
```

**Hyperbolic:**
```
ẋ = a sinh(E) · Ė,   ẏ = −a√(e²−1) cosh(E) · Ė
Ė = n / (e cosh E − 1),  n = 2π/T
```
(Was originally copy-pasted from `positionAtE` and completely wrong.)

**Parabolic:**
```
dD/dt = n / (1 + D²)
ẋ = −2q D · (dD/dt),   ẏ = 2q · (dD/dt)
```

#### 4. `getBoundingRadius()` – 500 AU cap, no negative return

```cpp
static const double MaxOrbitRadius = 500.0 * 1.4959787e8;  // km

double EllipticalOrbit::getBoundingRadius() const
{
    if (eccentricity < 1.0)
    {
        double apocenter = pericenterDistance * (1.0 + eccentricity) / (1.0 - eccentricity);
        return min(apocenter, MaxOrbitRadius);
    }
    else
        return MaxOrbitRadius;
}
```

Before this fix, `getBoundingRadius()` returned the apocenter directly for e > 1, which is negative (since a < 0), causing the renderer to produce zero samples.

#### 5. `isPeriodic()` – prevents renderer from closing open orbit loops

```cpp
bool EllipticalOrbit::isPeriodic() const
{
    if (eccentricity >= 1.0)
        return false;
    // Near-parabolic elliptic: apocenter beyond 500 AU → treat as open arc
    double apocenter = pericenterDistance * (1.0 + eccentricity) / (1.0 - eccentricity);
    return apocenter <= MaxOrbitRadius;
}
```

When `isPeriodic()` returns true the renderer appends an extra sample connecting the last point back to the first, creating a visually incorrect closed loop for open orbits.

#### 6. `getValidRange()` – provides the JD window for non-periodic orbits

The renderer calls `getValidRange()` to determine `nSamples`. If `begin == end` (the default), it assumes the orbit is always valid and uses the configured sample count. For non-periodic orbits the renderer uses the JD range to set nSamples proportionally.

```cpp
void EllipticalOrbit::getValidRange(double& begin, double& end) const
{
    double meanMotion = 2.0 * PI / period;
    double M_max;

    if (eccentricity < 1.0)
    {
        double apocenter = ...;
        if (apocenter <= MaxOrbitRadius) { begin = end = 0.0; return; }  // full ellipse
        // Near-parabolic: E where r = MaxOrbitRadius
        double a = pericenterDistance / (1.0 - eccentricity);
        double cosE = (1.0 - MaxOrbitRadius / a) / eccentricity;
        double E_max = acos(clamp(cosE, -1.0, 1.0));
        M_max = E_max - eccentricity * sin(E_max);
    }
    else if (eccentricity > 1.0)
    {
        double a_abs = pericenterDistance / (eccentricity - 1.0);
        double coshEmax = (MaxOrbitRadius / a_abs + 1.0) / eccentricity;
        double E_max = acosh(clamp(coshEmax, 1.0, 1e6));
        M_max = eccentricity * sinh(E_max) - E_max;
    }
    else  // parabolic
    {
        double D_max = sqrt(max(MaxOrbitRadius / pericenterDistance - 1.0, 0.0));
        M_max = D_max + D_max³ / 3;
    }

    begin = epoch + (-M_max - meanAnomalyAtEpoch) / meanMotion;
    end   = epoch + ( M_max - meanAnomalyAtEpoch) / meanMotion;
}
```

#### 7. `sample()` – adaptive curvature-based stepping for all orbit types

The existing elliptic sampler uses a curvature factor `k` to shorten the anomaly step near pericenter (where the orbit bends sharply) and lengthen it far out. All three non-elliptic cases now use the same pattern.

**Hyperbolic:**
```cpp
double b = sqrt(square(eccentricity) - 1.0);
double E = -E_max;
while (E < E_max)
{
    proc.sample(tsamp, positionAtE(E));
    double k = b / pow(square(sinh(E)) + square(b) * square(cosh(E)), 1.5);
    E += dE / max(min(k, 20.0), 1.0);
}
```

**Parabolic:**
```cpp
double D = -D_max;
while (D < D_max)
{
    proc.sample(tsamp, positionAtE(D));
    double k = 1.0 / pow(1.0 + D * D, 1.5);
    D += dD / max(min(k, 20.0), 1.0);
}
```

**Near-parabolic elliptic (apocenter > 500 AU):**
```cpp
double w = 1.0 - square(eccentricity);
double E = -E_max;
while (E < E_max)
{
    proc.sample(tsamp, positionAtE(E));
    double k = w * pow(square(sin(E)) + w * w * square(cos(E)), -1.5);
    E += dE / max(min(k, 20.0), 1.0);
}
```

Clamping `k` to [1, 20] keeps the total vertex count within [nSamples, 20×nSamples].

---

### `src/celengine/parseobject.cpp`

#### Period is now optional for e ≥ 1

Previously `Period` was required for all orbits. For open orbits it can be auto-computed from Gauss's gravitational constant and the orbital elements.

```cpp
double eccentricity = 0.0;
orbitData->getNumber("Eccentricity", eccentricity);

double period = 0.0;
if (!orbitData->getNumber("Period", period))
{
    if (eccentricity < 1.0)
    {
        clog << "Period missing!  Skipping planet . . .\n";
        return NULL;
    }
    // For e >= 1, period computed below from orbital elements.
}
```

After `pericenterDistance` is finalised:

```cpp
if (period == 0.0 && eccentricity >= 1.0)
{
    const double k = 0.01720209895;                       // Gauss gravitational constant (AU^3/2 / day)
    const double GM_sun = k * k * (KM_PER_AU * KM_PER_AU * KM_PER_AU);  // km³/day²

    if (eccentricity > 1.0)
    {
        double a = pericenterDistance / (eccentricity - 1.0);  // semi-major axis (km)
        period = 2.0 * PI * sqrt(a * a * a / GM_sun);
    }
    else  // parabolic
    {
        double q = pericenterDistance;
        period = 2.0 * PI * sqrt(2.0 * q * q * q / GM_sun);
    }
}
```

For hyperbolic/parabolic orbits `period` is a purely nominal timescale used to convert mean-anomaly deltas to JD offsets – it has no physical meaning as a repetition period (the orbit never repeats), but the internal math requires it.

#### SemiMajorAxis sign fix for hyperbolic

If `SemiMajorAxis` is given (rather than `PericenterDistance`) it must be negative for hyperbolic orbits so that `pericenterDistance = a*(1 − e)` is positive:

```cpp
if (semiMajorAxis != 0.0)
{
    if (eccentricity > 1.0 && semiMajorAxis > 0.0)
        semiMajorAxis = -semiMajorAxis;   // force negative for hyperbolic
    pericenterDistance = semiMajorAxis * (1.0 - eccentricity);
}
```

---

## Example .ssc Entries

### Hyperbolic comet (1I/ʻOumuamua-like)
```
"1I `Oumuamua" "Sol"
{
	Class "comet"
	Mesh "asteroid.cms"
	Texture "asteroid.jpg"
	Radius 5
	Albedo 0.1
	EllipticalOrbit
	{
		PericenterDistance         0.255240
		Eccentricity               1.199252
		Inclination              122.6778
		AscendingNode             24.5997
		ArgOfPericenter          241.6845
		MeanAnomaly                0.0
		Epoch                2458005.9886     # 2017 Sep 09.4886
	}
}
```

### Parabolic comet
```
"C 2012 E2 (SWAN)" "Sol"
{
	Class "comet"
	Mesh "asteroid.cms"
	Texture "asteroid.jpg"
	Radius 5
	Albedo 0.1
	EllipticalOrbit
	{
		PericenterDistance         0.006952
		Eccentricity               1.000000
		Inclination              144.2218
		AscendingNode              7.7078
		ArgOfPericenter           83.3435
		MeanAnomaly                0.0
		Epoch                2456001.5314     # 2012 Mar 15.0314
	}
}
```

---

## Key Bugs Fixed Along the Way

| Bug | Root cause | Fix |
|-----|-----------|-----|
| `getBoundingRadius()` returned negative for e > 1 | `a = q/(1−e) < 0` so apocenter formula returned negative | Return `MaxOrbitRadius` constant for e ≥ 1 |
| `eccentricAnomaly()` returned NaN for inbound hyperbolic leg | Iteration initialised for positive M, then given negative M | Iterate on `|M|`, restore sign at end |
| Hyperbolic `velocityAtE()` was wrong | Copy-pasted from `positionAtE` | Rewrote with correct `sinh/cosh·Ė` formula |
| Open orbits rendered as closed loops | `isPeriodic()` defaulted to `true` | Override to return `false` for e ≥ 1 and near-parabolic elliptic |
| Non-periodic orbits produced zero samples | `getValidRange()` returned begin == end (default "always valid") which renderer misinterpreted | Override `getValidRange()` to return actual JD window |
| All hyperbolic/parabolic samples had the same time tag | `sample()` passed period constant `t` as every sample time | Compute `tsamp = epoch + (M − M₀) / n` per sample |
| Near-parabolic elliptic orbits extended to huge distances | Apocenter could be 10 000+ AU with e = 0.9999 | Cap all bounding radii at 500 AU; treat such orbits as open arcs |
| "Pointy" orbit near pericenter at low zoom | Uniform anomaly stepping → sparse vertices where curvature is highest | Adaptive curvature factor k for all three non-elliptic cases |

---

## Orbital Mechanics Reference

### Conic section parameter mapping

| Parameter | Symbol | Elliptic (e<1) | Hyperbolic (e>1) | Parabolic (e=1) |
|-----------|--------|----------------|-----------------|-----------------|
| Semi-major axis | a | q/(1−e) > 0 | q/(1−e) < 0 | ∞ |
| Anomaly param | E, D | eccentric anomaly E | hyperbolic anomaly H | D = tan(ν/2) |
| Kepler's eq | – | E − e sin E = M | e sinh H − H = M | D + D³/3 = M |
| Position x | – | a(cos E − e) | −a(e − cosh H) | q(1 − D²) |
| Position y | – | a√(1−e²) sin E | −a√(e²−1) sinh H | 2qD |

### Gauss gravitational constant
```
k  = 0.01720209895  AU^(3/2) day^{-1}
GM = k² × (km/AU)³  =  k² × (1.4959787×10⁸)³  km³ day^{-2}
T  = 2π √(a³ / GM)  days   (Kepler's third law; a in km)
```

For a parabolic orbit a → ∞; use the Tisserand approximation:
```
T_para = 2π √(2q³ / GM)
```
This is the time for a body at pericenter distance q on a parabolic orbit to travel from −∞ to pericenter to +∞ (a conventional finite timescale for internal use).

---

## 500 AU Cutoff – Rationale

All orbit trail rendering is capped at 500 AU regardless of orbit type:

- **Hyperbolic / parabolic**: trail naturally extends to infinity; cap gives a finite, visually meaningful arc centred on pericenter.
- **Near-parabolic elliptic** (e slightly < 1): apocenter can be 10 000 AU or more.  
  Without the cap the orbit appears as a hairline arc that disappears into nothing.  
  With the cap it matches the visual appearance of a true parabolic orbit.
- **Normal elliptic**: apocenter ≤ 500 AU → no cap applied, full orbit drawn.

The threshold is set in one place:
```cpp
static const double MaxOrbitRadius = 500.0 * 1.4959787e8;  // 500 AU in km
```
