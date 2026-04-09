# Axis-Constrained Camera Navigation

This fork adds two new mouse shortcuts for rotating the camera around a fixed axis, useful for navigating the solar system and inspecting individual objects without the viewpoint drifting "above" or "below" a reference plane.

## Shortcuts

| Shortcut | Axis | Description |
|---|---|---|
| **Ctrl + Right drag** | Ecliptic north pole | Keep camera in the solar system's plane |
| **Ctrl + Shift + Right drag** | Selected object's rotation pole | Rotate around the object's spin axis |

Only horizontal mouse motion is used; vertical drag is ignored.

## Ecliptic Pole (Ctrl + Right drag)

Rotates the camera around the **J2000 ecliptic north pole** — the fixed "up" axis of Celestia's universal coordinate system. This axis is the same regardless of which object is selected.

Use this when navigating the solar system at a large scale: it keeps the camera in the ecliptic plane as you pan around, preventing the view from tilting out of the plane of the planets.

Works the same for any selected object type (planet, star, galaxy, etc.) and in any star system, since the ecliptic north pole is a fixed world-space direction.

## Rotation Pole (Ctrl + Shift + Right drag)

Rotates the camera around the **selected object's rotation pole** — its spin axis — keeping that axis fixed in world space. Behavior depends on the type of selected object:

**Planets and moons (`Body`):**
Uses `getRotationPoleDirection()`, which derives the pole from the rotation model's angular velocity vector. The angular velocity direction is unambiguously the rotation pole and is unaffected by the daily spin angle. For bodies whose reference frame is not the ecliptic (e.g., moons), the pole is correctly transformed through the body frame into ecliptic space.

**Stars:**
Uses `angularVelocityAtTime()` from the star's rotation model, normalized to a unit direction. For the Sun, this gives a pole tilted ~7.25° from the ecliptic north pole.

**Galaxies and other deep-sky objects (`DeepSky`):**
Uses the object's orientation quaternion (body→world convention) to transform the local Y-axis into world space, giving the symmetry axis perpendicular to the galactic disk.

**Fallback (location, or object with no rotation):**
Falls back to the ecliptic north pole.

## Implementation Notes

> **Note on naming:** In Celestia's source code, rotating the camera around a selected object is called `orbit()` — as in `Observer::orbit()` and `Simulation::orbit()`. This is Celestia's internal term for this camera motion, distinct from the orbital mechanics meaning used in `comet-orbits.md`. The functions added by this feature follow the same convention: `orbitAroundEclipticPole()` and `orbitAroundRotationPole()`.

The world-space rotation axis is converted into the observer's camera space before being passed to the existing `Observer::orbit()`:

```cpp
Vec3d cameraAxis = worldAxis * (~camOrientation).toMatrix3();
Quatf q;
q.setAxisAngle(..., angle);
observer.orbit(selection, q);
```

This reuses the existing `orbit()` math unchanged, ensuring consistent behavior with the normal right-drag camera motion.

Relevant source files:
- `src/celengine/body.h` / `body.cpp` — `Body::getRotationPoleDirection(double tdb)`
- `src/celengine/simulation.h` / `simulation.cpp` — `orbitAroundEclipticPole()`, `orbitAroundRotationPole()`
- `src/celestia/celestiacore.cpp` — modifier key dispatch in `mouseMove()`
