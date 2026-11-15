# Sun City RP: water code audit

Date: 2026-11-15. Source: `velumix/SunCityRP` (private), branch `main`, last push 2026-11-08.
Files read: `src/ReplicatedStorage/Client/Controllers/WaterController.luau` (541 lines),
`src/ReplicatedStorage/Shared/Config/WaterConfig.luau` (86 lines).
Note: the swim/boat controllers are not in this snapshot, so this audits only what is pushed.

## What is actually there

A real multi-octave Gerstner ocean, not a flat plane with a scrolling texture.

- 5 octaves (wavelengths 140 / 72 / 36 / 18 / 9, amplitudes 0.42 -> 0.03, clamped steepness).
- Radial mesh, 74 spokes, 2.5 stud base spacing, ring LODs, re-anchored under the camera in
  `Drift`-sized steps, horizon tiles for the far view.
- `GetHeightAt(x, z)`, `GetNormalAt(x, z)`, `GetSurfaceInfo(x, z)` as an analytic API.
- Vertex colours driven by crest height, foam blend above crest 0.70.
- Tuning is readable at runtime off the Ocean anchor part's attributes
  (`OceanWaveHeight`, `OceanSpokes`, `OceanSeaLevel`, ...). Good move, that is how you tune on a live server.

Credit where due: waves are computed from world coordinates (`wx = anchorX + rx`), so the mesh does
not slide when the anchor re-snaps. Many radial oceans get this wrong.

## Four things I would fix, in order of cost

### 1. The ocean exists only on the client. Nothing on the server knows where water is.

`WaterController` is `ReplicatedStorage/Client/Controllers`. Grepping `ServerScriptService`
for water turns up nothing: no buoyancy, no wake service, no swim authority.
`VehicleService` has no boat path at all.

That means today water is scenery. The moment a player swims or a boat floats, the thing that
decides the outcome (server) has no height function, and the client's copy is not authority.
This is the structural blocker, and it is also why "the physics is reactivity with objects and
velocity" cannot be satisfied yet: reactivity needs an authority that owns the impulse state.

Fix: move the wave *math* into `Shared` (it is pure functions of x, z, t + config) and have a
`WaterService` on the server own a clock, the disturbance list, and the buoyancy pass.

### 2. `GetHeightAt` disagrees with the mesh it draws. Gerstner displacement is one-directional.

The mesh applies horizontal displacement per vertex:

```
dx -= qAmp * oct.dx * sinT
dz -= qAmp * oct.dz * sinT
```

`GetHeightAt(x, z)` only sums `a * cos(theta)` at the raw world position. So it returns the height
of the surface at the *rest* position, not at the point that actually ends up at `(x, z)`.
Horizontal offset sums to about 0.45 studs at full crest. On steep crests that is a visible
float/sink error, and it gets worse exactly where it is most noticeable.

Fix: invert the displacement. Two or three fixed-point iterations are enough at this amplitude:

```lua
local function restPosition(x, z, t)
	local rx, rz = x, z
	for _ = 1, 3 do
		local ox, oz = 0, 0
		for _, oct in ipairs(_octaves) do
			local theta = oct.k * (oct.dx * rx + oct.dz * rz) - oct.w * t
			local s = math.sin(theta)
			ox -= oct.q * oct.a * oct.dx * s
			oz -= oct.q * oct.a * oct.dz * s
		end
		rx, rz = x - ox, z - oz
	end
	return rx, rz
end
```

Then evaluate height at `(rx, rz)`. Cache nothing; it is 5 octaves * 3 iterations, cheap.

### 3. Two clocks, two oceans.

`_waveClock` is a per-client accumulator. Two clients, and the server, will have different `t`
within minutes. Any replicated buoyancy or wake computed against a different clock shows up as
boats bobbing in the wrong phase, which reads to players as lag, not as a bug.

Fix: one authoritative time base. `workspace:GetServerTimeNow()` gives a shared clock on both
sides (`Workspace:GetServerTimeNow()` is synced to the server), so `t` becomes a pure function of
that instead of a local accumulator. Bonus: late joiners render the same sea everyone else sees.

### 4. Far rings drop octaves, the height query does not.

```lua
local maxOctave = if ring > 28 then math.min(numOctaves, 2) else numOctaves
```

Visually fine. But `GetHeightAt` always runs all 5 octaves, so a boat far from the camera floats
on a surface nobody drew. Either is acceptable alone; together they are an invisible hand pushing
the boat. Cheapest fix: make the LOD cutoff a shared function so render and query agree.

## Small things

- `Horizon` tiles are `CanQuery = false`. Good. Keep them that way or every raycast toward the
  horizon hits a sea-level slab.
- `ensureHorizon` builds up to 16 tiles of 2048 studs at `SeaLevel - 0.5`. Boats that drive past
  `Reach` (900) will leave the displaced mesh and float on the flat horizon slab instead. If boats
  can go far, that transition needs a visible blend or a bigger reach for the boat's LOD.
- Crest colour clamps `dy / totalAmp` using the *render* octave count, so far vertices saturate
  to white foam differently from near ones. Cosmetic, but it shows as a ring boundary.

## What I would tell the swarm

The wave math is good. The gap is ownership: there is no server-side water, so there is no place
to put reactivity, wake state, or boat authority. Fix the owner first, then the Gerstner inversion,
then the clock. Doing it in the other order means rewriting the physics twice.

## Follow-up: two files that make the server own the sea

Pushed on branch `velumix/water-shared-math` of `velumix/SunCityRP` (not main, nothing merged):

- `src/ReplicatedStorage/Shared/Water/WaveMath.luau` — the same Gerstner spectrum as
  `WaterController`, as pure functions. Two differences that matter: time is an argument,
  never an internal accumulator, and `WaveMath.restPosition` inverts the horizontal
  displacement with three fixed-point passes, so `sample(x, z, t)` returns the height at the
  point that actually lands at (x, z). Client mesh and server buoyancy both require this file,
  so they cannot drift apart.
- `src/ServerScriptService/Services/WaterService.luau` — draft Nerve service. Owns
  `Workspace:GetServerTimeNow()` as the clock, a decaying wake list (`PushWake` / `SampleWake`,
  snapshotted to clients at 10 Hz), and a fixed-step buoyancy pass over `SunCityFloater`-tagged
  parts. Buoyancy is sampled at four hull corners, so a boat rocked by a crest gets its up-vector
  from the wetted corner normals and levels without losing heading.

Untested in Studio. The AlignOrientation semantics (OneAttachment, world-space CFrame) are the
part I would check first, before trusting the leveling.

## Fifth thing, smaller: the far rings draw fewer octaves than the math says

`updateMesh` caps rings past 28 at two octaves. `GetHeightAt` sums all five at any distance.
Inside the mesh that is invisible, but a buoyancy pass sampling the whole map will find swell
that the surface you are looking at at 400 studs does not show. Either fade the sampled
amplitude with distance or accept that physics is richer than the horizon.
