# Notes: the original arm rig (before the full-body upgrade)

These notes describe `index.html` as it was before the full-body work (commit `32f8ce7`). They are a map of what is
reused, so the new code can wrap the arm and hand systems without rewriting them.

## Frame and proportions
- Units are metres. In the bind pose the left arm is a T-pose along +X, palm down (-Y), thumb forward (+Z).
- Upper arm `L1 = 0.30`, forearm `L2 = 0.26`, wrist at `WX = 0.56`, palm end `PE = WX + 0.083`.
- The elbow hinge is local Y; flexion swings the forearm toward +Z.

## Skeleton (38 bones)
- `shoulder` (root, translation only), `upperArm`, `elbowHalf` (helper that turns half the elbow angle),
  `forearm` + `twist1..3` (pronation split in thirds), `hand`.
- Five fingers × (3 bones + 3 half-angle helpers). The thumb's first bone is the metacarpal, buried in the thenar pad.
- `FINGERS[]` holds per-finger data: base position, rest quaternion, bone lengths, radius keys `rk`,
  `spreadSign` (which way +Y spread moves away from the neighbours) and `fanFix`.

## Skin
- **Arm tube**: `addTube()` sweeps a superellipse cross-section (`HY`, `HZ`, exponent `NN` keyed along x with a
  cubic Hermite `key()`), with Gaussian muscle `BUMPS` (deltoid, biceps, triceps, olecranon, brachioradialis, …).
  The palm is the far end of the same tube; the knuckle line is curved and the metacarpal heads are bumps.
- **Painted weights** (`armWeights`): around the elbow the blend width depends on the angle round the limb:
  ~1 cm on the inside (nearly rigid segments that fold along a crease) and ~7.5 cm on the outside (smooth stretch).
  The middle band goes to the `elbowHalf` helper. The forearm blends through `forearm → twist1 → twist2 → twist3`,
  then into `hand` at the wrist.
- **Hand SDF**: palm (superellipse matching the tube), five finger capsules with knuckle and pad bumps, combined with
  a polynomial smooth-min (`smin`, wider for the thumb web). Meshed with **surface nets** (one vertex per sign-changing
  cell, one quad per crossing edge) at 1.3 mm. Each vertex mixes the palm weights and each nearby finger's
  weights (`fingerWeights`: hat functions over the 3 joints with helper bands) by `exp(-distance/3.5 mm)`.
- **Morph targets**: 0 = biceps contraction, 1 = forearm tension; driven by elbow flex and hand tension springs.
- `elbowHalf.scale.z` is stretched by the mitre factor `1/cos(θ/2)` so the elbow keeps its volume.
- Nails are small flattened spheres parented to the distal bones.

## IK (`update`)
- Analytic two-bone IK in the arm's parent frame: law of cosines for the elbow; the pole spring sets the elbow plane.
- Forearm twist: roll about the forearm until the palm faces `P.palm`, clamped to [-1.4, 1.5] rad, split /3 over the
  twist bones. Wrist flex/deviation via Euler `YZX` on `hand`.
- The shoulder origin (`shS`) leans 3 cm toward long reaches.

## Spring layer
- Every channel is a critically-damped-ish spring `{x, v, t, w, z}` (semi-implicit Euler, sub-stepped at ≤6 ms).
- Vector springs (`VS`) for the IK target, pole, palm direction, shoulder offset, ball.
- Secondary motion: wrist velocity in the forearm frame (`vf`) drags the wrist flex/deviation and finger curls.
- Finger springs: the distal joints and outer fingers are softer, so a curl ripples knuckle → tip, index → pinky.

## Finger contact solver
- Each finger bone is a capsule (`F.sr` radii). Colliders: palm box, ball (`ORB_R`), table plane, and for the thumb
  the other fingers (segment–segment distance).
- `resolveFinger`: keeps last frame's pose; if penetrating, bisects the smallest uncurl that clears; otherwise each
  joint knuckle-first steps toward its target (≤ 12 rad/s) and stops at contact (bisection). If a finger is pinned
  against the table even when open, the whole hand is lifted (`needLift` → `pushUp`).
- The resolved pose is low-passed (`S.sm`) before being written to bones and helpers.

## Hand behaviour tuning
- Two-stage fist (`fistBlend`): OPEN → BEAK (knuckles fold, tips gather) → FIST.
- Thumb lags: swings to `ready` while fingers close, folds (`fold`) a beat later and lands as the index finishes.
- Shared tendons: each finger's target mixes its neighbours' (ring finger most coupled).
- Relaxed curl (`RELAX`) deepens toward the pinky (ulnar bias).

## Behaviours
- `MODES` write a desired pose `P` (target, pole, palm direction, wrist, finger curls/spread, thumb roll, tension);
  the springs and solvers do the rest. Modes: reach & grab, pick & place, finger wave, wave, clench, finger roll, rest.

## Rendering and extras
- MeshPhysicalMaterial skin with sheen, HDR environment from a CDN, PCF soft shadows, procedural wood table.
- Views: skin, X-ray (bone octahedra), weights (vertex colours per bone).
- `direct()` renders a scripted two-panel showcase frame by frame and posts frames to `record-server.js`.

## Plan for the full body
- The arm and hand code moves into a factory, `makeArm(side)`, that runs the original code once per side. The
  geometry is built once in the canonical (left-arm) frame and copied for the right side through a mirror.
- Each arm solves in its own **arm space**: origin at the shoulder joint, axes from the chest, mirrored in X for the
  right side. Inside that space the original IK, twist, wrist and finger contact code run unchanged.
- The arm root's matrix is written directly (it may hold a reflection), so the skin of both arms comes from the same
  painted weights.
