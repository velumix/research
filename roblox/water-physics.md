# Roblox water physics

Verified 2026-11-14 from Roblox Creator docs and DevForum threads. Marked `[read]` where it is my inference rather than documented behaviour.

## Terrain water

- Built-in buoyancy, no scripting. Density < 1 floats, = 1 hangs neutral, > 1 sinks.
- Waterline tracks density: a body at density 0.5 sits about half submerged.
- Default material densities: wood 0.35, plastic/neon/ice float by default. Dense materials sink.
- Welding a low-density part to an assembly makes the whole assembly float.
- **No drag.** Terrain water does not slow you down. You add that yourself.
- Buoyancy reads the surface as a flat plane. Waves are cosmetic and do not move the float behaviour. `[read]`
- **No per-part switch to disable terrain water buoyancy.** Only collision groups or avoiding terrain water. Long-standing open DevForum request as of Oct 2025.

## Part water (hand-rolled)

- Archimedes per sample point: `Fb = rho * V * g`, applied at corners, not one centre force. One centre force gives you a boat that refuses to rock, and a boat that refuses to rock reads as a brick. `[read]`
- Standard hull trick: normal density hull + welded invisible `CanCollide=false` part at density ~0.01 as buoyancy volume. Setting the hull itself to very low density causes violent collisions.
- Buoyancy lifts, it does not level. Need `AlignOrientation` or `BodyGyro` to stay upright, and `VectorForce` opposing velocity for drag.
- Any per-frame force from a read-only height function is memoryless. If the water should *react* — wake, push back, remember the hull passed — you need decaying impulse state on the server, and client and server must share both the clock and the impulse list, or they disagree about where the water is. `[read]`

## Network ownership

- Server-owned assembly is the only clean option for passenger boats.
- Multi-passenger vehicles are a real desync risk under network ownership handoff.
