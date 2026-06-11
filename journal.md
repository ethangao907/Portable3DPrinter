# Portable 3D Printer — Project Journal
I compiled this with AI

Design log for the folding portable printer, January 2026 to present.

---


## Mid-January 2026 — Studying prior art

Pulled apart the chatlogs from Malte Schrader's xPrinter project (2019-2020), the closest existing scissor-lift folding printer.

What I extracted:

- Their nonlinear Z relationship and the Marlin firmware hacks they used (custom GenZ() function in planner.cpp, `zm = sqrt(246.02^2 - (62.5 + z)^2)`).
- Non-uniform ballscrew backlash made auto bed leveling impossible for them. Probing requires direction changes, the backlash changes with fold state, so software compensation could not work. Their proposed fix was a linear Z encoder, never implemented.
- Their housing flexed under the extreme forces of unfolding, the Z motor mount bent slightly, and pivot preload changed with extension.
- Bowden vs direct drive debate in their community. They chose Bowden purely for space. Decided on direct drive for my version. The scissor already adds enough variables (nonlinear motion, backlash, mechanical complexity) without stacking retraction tuning and tube friction on top.
- Diagonal arm mounting for weight distribution was proposed in their logs and never resolved.

## Late January 2026 — Analysis phase

### Scissor kinematics

Derived from scratch. For link length L and base separation d:

- h = sqrt(L² − d²/4)
- dh/dd = −d/(4h) = −1/(2 tan θ)

Mapped mechanical advantage and lift force vs angle. Lift force for mass m is F = mg/(2 tan θ), so motor torque requirement drops as the scissor extends. Conclusions:

- Collapsing below θ ≈ 25-30° is useless (at 20°, 8.5mm of base travel per 1mm of height).
- Extending past 75-80° kills lateral stiffness (at 70°, lateral stiffness is ~40% of collapsed).
- Operating band set at roughly θ = 30-35° collapsed to 65-70° extended, sweet spot 40-60°.
- Link length sized as L = target height / sin(70°), landing around 200mm.
- Considered diagonal bracing wires or carbon rods from base corners to top platform for extended-state stiffness.

### Minimum Z height

100mm is the absolute floor (covers ~80% of typical hobbyist prints), 150mm is comfortable (~95%), 75mm is too limiting (~50%). Aimed above 150mm.

### Z actuator trade study

Three options. Vertical central lead screw (simple kinematics, needs a screw as long as max height), horizontal base lead screw (compact, nonlinear, needs firmware compensation), belt at a corner (compact, geared small motor). Belts were argued hard for (zero backlash with tension, ~50g, 10x faster, quiet). Force math said it didn't matter for capability. NEMA 17 (0.4 Nm) on a 2mm lead gives ~1256N theoretical, ~628N at 50% trapezoidal efficiency, against a ~13N load (bed + mechanism + part), a 48x safety factor. 8mm lead still gives 12x and is 4x faster, so if screws, 8mm lead and no planetary gearbox. The gearbox added 200g, $35, and backlash to solve a problem that didn't exist.

### Extruder trade study

Compared Sprite (280g, heaviest), BIQU H2 (220g, 7:1, 80-90N extrusion force, integrated hotend), Sherpa Mini (90g, recommended for the power and weight budget), Orbiter V2 (140g), LGX Lite (160g), DIY Bowden (lightest, worst quality). Chose the H2 for v1 despite the weight.

### Power architecture

Two architectures considered. Off-the-shelf DC UPS module (turnkey, protections built in, sub-10ms switchover) vs custom ideal-diode OR-ing (LTC4352/LTC4353 controller or TPS2121 power mux, 6S BMS with 4.2V/cell overcharge and 3.0V/cell overdischarge cutoffs, balance, 15-20A overcurrent). Went custom.

Worked through LiPo fundamentals to ground the design. Voltage maps directly to state of charge (4.20V full, 3.30V effectively empty, below 3.0V copper dissolution permanently damages the cell, above 4.2V dendrite formation risk). Capacity 22Ah at 22.2V nominal is ~488Wh. Key realization on charging, the Meanwell PSU set to exactly 25.2V with its 13A current limit performs CC/CV natively. The current limit is the CC phase, the regulated output is the CV phase, current tapers naturally as battery voltage approaches the rail. No additional charge circuitry needed, the BMS handles protection and the ideal diode handles the path. End behavior is laptop-like, plug or unplug mid-print in either direction with no interruption.

### XY mechanism alternatives

Reviewed the CoreH-bot paper (planar single-belt mechanism, 14 pulleys, 4 motors, same Jacobian as CoreXY, balanced torques like CoreXY without 3D belt routing, 250mm/s at ~0.1mm tolerance in their prototype). Considered for compactness and power efficiency. Stayed with CoreXY.

### CAD methodology

Layout sketches and spatial budgets before any geometry. Hardware defines positions, designed parts fill the gaps. Clearance tables (3mm rotating parts, 2mm belts, 10-20mm around the hotend, 10mm wiring, fastener access for hex keys, 2-5mm PCB to frame, 10-20mm battery ventilation). Interference detection run at every extreme position after every added part. Motion studies of the scissor across full Z to catch short cables and collisions before building. Phase order, hardware placement, frame, motion system, toolhead, electronics, cosmetics. Each part made as big as possible without colliding with others. When a new part wouldn't fit, space would be carved out of existing parts to make room, requiring considerable strength analysis.

Misc from this period, 90° bend duct losses on the blower fan, battery bay ventilation requirements.

---

## February–March 2026 — V1 CAD and fabrication

Most CAD happened here, less documented research and calculations. Notably, I fit 3 linear rails, two axial + thrust bearing preload stacks, and a CoreXY cable routing into a 25x80mm cross section. Parts reached up to V7.

The packaging process that emerged, set the baseline envelope first (300×300×80mm fold, scissor lift, 10h battery target, direct drive toolhead), calculate from constraints to a real parts list, import parts into CAD and arrange until a configuration fits, then design the printed "glue" brackets that hold the off-the-shelf parts in that arrangement.

### March 30 — Dyneema pulley design

- Grooved vs smooth idlers. Grooved wins for round slippery cable. U-groove with radius 5-10% larger than cable radius, V-grooves wedge and wear.
- A true helical groove on a cylinder keeps constant effective radius, the ramp/cam concern only matters at crossover regions and is negligible at small pitch.
- For idlers with axially offset entry/exit, tilting the pulley a degree or two (arctan(Δz/2L)) keeps the cable in a single circumferential groove. No helix needed.
- For the multi-wrap drive capstan, a smooth drum was chosen over a helical groove. The helix forms naturally from approach angle, constant radius everywhere, at the cost of possible axial walking.
- Wrap count sized with the capstan equation using a conservative Dyneema-on-PLA COF of 0.05-0.10 against a worst-case inertial load of F = ma ≈ 0.25kg × 3 m/s² = 0.75N at a modest 3000 mm/s² CoreXY acceleration.

### April 2 — Drive pulley sizing and routing

- Drum diameter set to 80/π mm so circumference is exactly 80mm and steps/mm divides cleanly. Klipper doesn't strictly need integer steps/mm, but it avoids rounding accumulation and makes calibration math clean.
- Effective wrap diameter includes one cable diameter on top of the groove minor diameter, and changes slightly under tension.
- Checked whether a wrap-angle difference between the two CoreXY paths matters (3 wraps vs 3 wraps + 10° at 10N). Capstan ratio difference ~1%, slip impossible at that wrap count. The real path-matching concern is stretch consistency between unequal path lengths, not wrap angle.
- Routing geometry decision, linear height increase across a 2-segment span requires one bearing tilted in both x and y.

### April 4 — Nonlinear Z in Klipper, v1 spec complete

Four approaches evaluated for the scissor's nonlinear motor-to-nozzle mapping:

1. Custom kinematics module in `klippy/kinematics/` (the correct answer). Base on corexy.py, override only the Z portion with forward/inverse scissor kinematics.
2. Lookup table via gcode_macro. Jinja2 has no asin, would need polynomial approximation or discrete interpolation. Fragile.
3. z_thermal_adjust abuse. Wrong tool, it's an additive thermal correction.
4. Dynamic step_distance scaling. Breaks homing.

Key derivations needed, exact screw-travel-to-θ relationship, where Z=0 sits on the curve (stay away from the collapsed singularity), and the θ range that keeps step resolution usable.

v1 spec as fully written at this point:

- 180×200×175mm build volume in a 300×300×80mm folded envelope, ~85% volumetric efficiency.
- Nested scissor arms, every pivot preloaded with a thrust + axial (deep groove) bearing combination clamped by bolt and nut. The stationary bottom pivot pair, the most critical joints, use 50mm × 6mm shoulder bolts in compliant printed clamps.
- One MGN9 rail nests into the scissor arm itself, so three MGN9 rails, two preloaded pivot assemblies, and the full CoreXY cable routing coexist in an 80×25mm cross section.
- CoreXY driven by Dyneema instead of GT2. The cable crosses itself multiple times, threads under mounting plates, tensioned by two adjustable rear screw tensioners. Dyneema twists out of plane between pulleys, which belts cannot, and is 20-40x stiffer per cross section, worst-case position error ~0.08mm over full travel.
- BIQU H2 direct drive, 4020 blower for part cooling, BLTouch at 0mm X / 24mm Y offset to minimize lost bed mesh area.
- 6S 22Ah LiPo + slim 400W Meanwell 24V PSU through the ideal-diode OR-ing circuit. Simultaneous wall printing and charging, seamless switchover both directions. ~50W system draw with no heated bed, 8-10 hour theoretical runtime.
- Klipper on a Raspberry Pi Zero 2W over WiFi (later Pi 4 in the built machine), BTT SKR Mini E3 V3, BTT Mini 12864 display.
- Every printed component fits a 220×220mm bed, no machined parts.

Also worked out a battery level display hack, expose the pack voltage through a fake `[temperature_sensor]` via `[adc_temperature]` scaling so Mainsail shows it in the top bar, or a custom KlipperScreen panel.

### April 20 — Capstan walking

The major v1 failure mode. Under rotation the Dyneema migrates axially along the smooth drum and bunches at one end until the system jams. Rope clumped, wraps locked, the carriage stalled. The lesson, cable behavior on small drums with mismatched inlet/outlet heights is a contact mechanics problem, not a geometry problem.

Solutions considered, helical grooves (ruled out, unlimited bidirectional rotation and no axial room), fleet angle correction via shaft tilt, fairlead pulleys, Jake Read's double idler, translating drum. Converged on a self-tailing winch mechanism from sailing hardware as the most promising structural fix. The self-tailing guide physically forces the rope back to a fixed axial exit position every revolution, eliminating the walk instead of slowing it.

Quantitative pass:

- 80/π mm drum, worst-case diagonal move sqrt(200² + 175²) ≈ 266mm = 3.3 drum rotations.
- Effective rope modulus E ≈ 40 GPa for braided Dyneema (raw UHMWPE fiber is 100-130 GPa, discounted for braid angle, bedding-in, and ~65% packing).
- Per-diameter results over the 1030mm path with anchor distances of 65mm (input side) and 140mm (output side):

| Diameter | D/d | Walk per full move | Fleet-angle violation | Stiffness | Tension bump |
|---|---|---|---|---|---|
| 2.0mm | 12.7 (6.4 on small idlers) | 6.6mm | 0.52mm | ~88 N/mm | 46 N |
| 0.8mm | 31.8 | 2.64mm | 0.08mm | 13.6 N/mm | 1.1 N |
| 0.5mm | 51 | 1.65mm | 0.031mm | 5.0 N/mm | 0.16 N |
| 0.3mm | 85 | 0.99mm | 0.011mm | 1.8 N/mm | 0.02 N |

- 2mm is badly oversized on every axis. 0.5mm is the sweet spot, breaking strength 350-500N against 20-30N peak loads, 15-25x safety factor. 0.3mm works but knots and splices (30-50% strength loss) start to matter at 5-12x.
- Sourcing, high-quality 8-carrier braided fishing line (Sufix 832, PowerPro Super 8 Slick) is functionally identical to marketed Dyneema rope at a fraction of the cost, 80-100 lb test corresponds to the target diameter.
- Perspective on tension error sources, acceleration loads (~2N at 5000 mm/s² with a 400g head), creep bed-in (10-20% preload loss over the first days, slow creep after), enclosure temperature, and capstan friction at each direction change are all 10-100x larger than fleet-angle effects. Fleet-angle stretch and the positioning violation are the same 0.08mm viewed two ways, not additive.

---

## May 2026

### May 8 — Electronics bring-up, nonlinear Z round two

Motor spin tests. Second pass on the kinematics with new physics caught:

- Resolution at the high-gain end. Near collapse, dz per microstep = 2L·cos(θ)·(2π/steps_per_rev)/gear_ratio. For 2L = 200mm, 1600 steps/rev, no gearing, ~0.78mm per microstep at the bottom, 4x a typical layer height. Either gear down ~30:1 or keep the printable range above θ = 30°, where the gain variation across the range is a manageable ~5x (cos30/cos80).
- Velocity limits. Motor velocity = toolhead velocity / gain, so the motor flies at the low-gain top of travel. max_z_velocity is set by worst case, homing and probing speeds need attention.
- Implementation path confirmed, custom kinematics file with delta.py as the reference (delta Z is also nonlinear with tower position), a C helper through `setup_itersolve` in `klippy/chelper/`. The win over gcode remapping or post-processing, bed_mesh, probing, and axis_twist_compensation all operate in nozzle Z space and keep working.

### May 9-15 — V1 status, captured mid-build

Status at this point, mechanical architecture fully designed, CAD complete, several structural parts printed, upper gantry partially assembled. The scissor pivot system was the most-iterated section, getting scissor stiffness right without machined components took multiple revisions. The active technical problem was XY cable routing at the spool level, with thinner cable (~0.8mm), level-wind guidance, and GT2 on selected paths under evaluation.

Competitive landscape as mapped, Positron V3 is the nearest prior art but runs a 200W external PSU. Prior battery attempts (a 2020 student DVD-drive build, TOME, Pbag) failed on ~3 hour runtimes, tiny volumes, or never reaching reproducible form. Reference desktop machines (A1, MK4, Ender 3) pack ~220×220×250 build volumes into footprints roughly 4x the print area, mostly frame, motors, and dead air. Positron hits ~95% volumetric efficiency but is mains-tethered.

### May 18 — The V2 fork

Decision, get v1 to first-print state in 1-2 weeks for closure, then pivot to v2 CAD. v1 lessons feed forward, iteration stories beat clean pivots.

Dual side lead screws confirmed as the central v2 architecture change, replacing the single central screw:

- The central screw dictated the entire superstructure. The bottom bridge plate had to tie both sides together, which forced the surrounding frames. Removing it collapses the dependency chain and opens a central cavity big enough for a full spool. Shortening the rear pivot bolts opens further space.
- Eliminates single-point gantry racking (Voron 2.4 / RatRig precedent).
- Independent motors over a belt-synced pair, Klipper Z_TILT_ADJUST is too valuable to give up, especially on a machine that gets folded and redeployed where the frame may not unfold to exact tolerance every time.
- Screws placed along the scissor pivot axis so they fold into the same plane as the scissors. Screw length cannot exceed folded height, which couples build height to folded thickness.
- Lead screws with anti-backlash nuts over ball screws (cheaper, quieter, alignment-abuse tolerant), T8x8 standard.
- Electronics relocate behind the filament roll. Filament path must not cross the PCB, board must stay accessible without removing the spool.

The 90° toolhead problem named explicitly. An H2 cannot be rotated, its motor and gear geometry assume vertical filament drop. Paths considered, commission the open-source Positron 90° hotend + horizontal extruder (fastest, proven), design a custom horizontal extruder around the Positron hotend (~4 weeks CAD + machining), or full scratch design (3-6 months, kills the timeline).

Mod-a-Positron vs ground-up analysis. Modding fails on three fronts, no internal volume exists for a 500g spool bay (forces a frame redesign, not a mod), PSU sizing (~360W needed with a 150W bed + 100W chamber heater vs a ~150-200W stock bay), and electronics must relocate outside the heated zone (NEMA 17s lose torque above ~80°C ambient, drivers definitely move). Alternatives, build a stock Positron as a test mule to validate the thermal model, or fork the CC-BY-SA CAD and spend the design budget on the differentiators. Estimate 3-5 months from a fork vs 5-7 ground-up.

Proton analysis (the larger machine from the Positron team). It validated the battery de-prioritization, Proton uses USB-C PD as failover only with mains primary and kills one of two bed zones in battery mode. The same conclusion reached independently here, battery is a transport feature between mains locations, not a primary mode. Three differentiators confirmed unclaimed, integrated folding enclosure (Proton punts, suggesting a collapsible painting tent), internal filament storage (their Gluon extruder is about extruder compactness, not spool storage), and tool battery support (USB-C PD caps at 100-140W and most PD banks don't deliver it). Proton's strain-gauge nozzle-tap leveling is now table stakes and was added to the v2 spec.

The v2 repositioning, stated plainly. v1's logic was battery + low power, therefore no heated bed. v2 inverts it. Drop the onboard battery entirely (a liability, and true off-grid printing is rare in practice), add a heated bed and a folding bellows enclosure targeting a 50-60°C chamber for ABS/ASA/nylon, and take power from a front-mounted power tool battery (Milwaukee M18 / DeWalt) replacing the screen, 2-4 hour transport-mode printing (PLA only, no chamber heat), mains-powered charging in a future version. No longer an off-grid machine, a foldable enclosed transportable printer that runs on job-site batteries and prints engineering plastics.

Remaining v2 work as listed, dual side lead screws with software Z-tilt, folding bellows enclosure (50-60°C), heated bed sized for chamber and power budget, tool battery mounting blocks, custom 90° hotend with rear cable routing (kills the floppy top conduit), 3mm GT2 belt drive, HBot first with CoreXY fallback if racking forces are too high.

Open TBDs flagged, build volume target (drives enclosure, bed, spool clearance, PSU), 500g vs 1kg spool, folded dimension target, bed surface (settled toward PEI flex steel, dual heating zones, under-bed insulation), custom integrated board vs Pi + SKR.

Summer timeline, weeks 1-2 v1 first print + closure video, weeks 3-6 v2 CAD + commission toolhead, weeks 7-10 build and iterate, weeks 11-13 launch content + waitlist, kit fulfillment spring 2027.

### May 23 — Viability check

Demand validation friction ladder, email waitlist, paid IP sales, refundable deposits, small accessory sales. Revenue bracketed honestly, $25-50k bear to $300-700k+ bull depending on quality, distribution, and reseller partnerships. Questioned the effort-to-revenue ratio against simpler consumer products, the structural answer is TAM, purchase frequency, and decision friction differ, and the project's real value is career capital and optionality. Disposed of several kilograms of failed PLA parts from the v1 build.

---

## June 2026

### June 2 — Bellows physics, leveling validity

- For a 40-60°C FDM chamber the dominant heat loss is convective/air leakage, not radiation. Bellows material spec therefore, air-impermeable, heat-tolerant to ~60°C, fold-capable without cracking. Primary candidates, TPU film (transparent) or TPU-coated polyester fabric (opaque). Silicone-coated fiberglass considered earlier.
- Three-point leveling validity, two corners plus one midpoint is fully valid. Equilateral is the theoretical optimum but the performance curve is very flat. The real failure modes are skinny/near-collinear triangles and mounts that don't coincide with actual structural support points.
- Confirmed for travel, the printer (no battery), bench PSU, wires, and boards are all permitted carry-on across US/Canada/international security.

### June 8 — Scissor layout final analysis, repo

Four bed-support configurations ranked with stiffness as the sole criterion:

1. Four-edge scissors, synchronized. Stiffest, but statically indeterminate. A rigid plate has exactly three out-of-plane DOF (heave, pitch, roll), a fourth support is redundant. Load distribution then depends on relative support heights, and with lead screw lash, joint tolerance, and thermal growth the four supports will not match, so the bed either rocks (four-legged-table wobble) or gets bent as supports fight. Four driven points over-define the leveling plane, Z_TILT_ADJUST assumes an exactly-constrained set. On a machine that folds and redeploys, every redeployment resettles the four supports differently. Justified only at bed sizes where three-point sag is real.
2. Three-point (two scissors + passive diagonal, or three scissors). Fully constrains all out-of-plane DOF with zero redundancy, exact-constraint leveling, no fighting. The winner for a small portable bed. Caveat, a passive support must be low-friction in Z (linear bearing or rail, not a sliding fit) or it binds against the lead screw leveling.
3. Opposite-side two. One directly supported tilt axis, one soft joint-compliance-limited axis.
4. Adjacent-L unsupported. Worst, full-diagonal cantilever sag.

Side conclusion, if bed flatness under load is the worry, stiffen the plate (thickness, ribbing, material) rather than add a fourth lift. It helps every layout without importing over-constraint.

Repo published at github.com/ethangao907/Portable3DPrinter. Firmware/software under GPLv3, hardware design files under CERN-OHL-S v2 (added manually as LICENSE-hardware). Design files live in the `Scissor 3D Printer` folder.

Current v2 hardware as of this entry, "Foldup 3D printer" assembly in Onshape (668 instances), dual lead-screw Z, Dragon Burner V8 toolhead, Sherpa Micro extruder, SKR Mini controller, heated bellows enclosure for engineering plastics.

---

## V1 final configuration (as built)

- Scissor-lift Z, central lead screw, Klipper nonlinear Z mapping
- Dyneema CoreXY, rear screw tensioners
- BTT SKR Mini E3 V3 + Raspberry Pi 4, 24V bus
- Folds to ~300×300×80mm, ~180×200×175mm build volume
- No enclosure, no heated bed
- Deploys, homes, and prints

## V2 direction (active)

- Dual independent side lead screws, Z_TILT_ADJUST, central filament bay (500g internal, 1kg external optional)
- Folding bellows enclosure, 50-60°C chamber, ~80W chamber heater
- Heated bed, PEI flex steel, dual zones, under-bed insulation
- Tool battery mounting blocks front, onboard battery deleted
- Custom 90° hotend, rear cable routing
- 3mm GT2 belt, HBot first, CoreXY fallback
- Strain-gauge nozzle-tap leveling
- Three-point bed support
