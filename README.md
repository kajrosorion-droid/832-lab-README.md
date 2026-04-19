#832.код версия v12,1)import numpy as np
import matplotlib.pyplot as plt
import hashlib

# =============================================================================
# 832 Protocol v12.1 — Tiered Sacred Lifecycle + Structural Anchors
#
# Root cause of Sacred: 0 in v12.0:
#   1. decay_sacred killed at age > 120 — too short (4000 step run)
#   2. field_delta: agent standing on sacred gets +2/step → tension rises
#      → sacred created at tension=40, immediately becomes fragile
#      → next agent arrives with tension > 200 → transformation
#
# Fix: Tiered Sacred lifecycle:
#   YOUNG   (age   0–50):  immune, forming. No transformation possible.
#   MATURE  (age  50–200): normal, can transform under collective pressure.
#   ANCIENT (age 200+):    structural anchor. Very hard to destroy.
#                          Strong repulsion of overloaded agents.
#                          Max lifetime: 500 steps.
#
# Additional fixes:
#   - Sanctuary spawn: excludes recently-exhausted positions (no oscillation)
#   - Given threshold: lowered (was too insensitive, only 12/4000)
#   - field_delta: sacred at dist=0 only +1 (was +2, too aggressive)
#   - Ancient sacred: repels agents with tension > 100 (protective aura)
# =============================================================================


def stable_hash(*args):
    h = hashlib.sha256(str(args).encode()).hexdigest()
    return int(h[:8], 16)


class World:
    def __init__(self, size=15):
        self.size = size
        self.danger = {(3,3),(3,4),(4,3),(10,10),(10,11),(5,5)}

    def step(self, pos, action):
        x, y = pos
        nx, ny = x, y
        if action == 0: ny -= 1
        elif action == 1: ny += 1
        elif action == 2: nx -= 1
        elif action == 3: nx += 1
        if not (0 <= nx < self.size and 0 <= ny < self.size):
            return pos, True
        return [nx, ny], (nx, ny) in self.danger


class Witness:
    def __init__(self, size=15):
        self.size = size
        self.active_scars    = set()
        self.dream_scars     = set()
        self.sacred          = set()
        self.ghost_scars     = set()
        self.sanctuaries     = {}        # {pos: vitality (int)}
        self.exhausted_recently = set()  # sanctuary cooldown positions
        self.meaning_nodes   = {(7,7), (2,12)}
        self.sacred_age      = {}        # {pos: age}

        self.burn_int           = 1
        self.entropy_pressure   = 0
        self.last_variance_int  = 0
        self.stagnation_counter = 0
        self.last_signature     = None
        self.given_pulse        = []

    # --- SACRED PHASE ---

    def sacred_phase(self, pos):
        """Returns 'young', 'mature', or 'ancient' based on age."""
        age = self.sacred_age.get(pos, 0)
        if age < 50:    return 'young'
        if age < 200:   return 'mature'
        return 'ancient'

    # --- CONSTRAINT CHECKS ---

    def is_blocked(self, pos):
        return pos in self.active_scars

    def is_ghost_blocked(self, pos, tension):
        return pos in self.ghost_scars and tension > 120

    def is_ancient_blocked(self, pos, tension):
        """Ancient sacred repels overloaded agents (protective aura)."""
        if pos not in self.sacred:
            return False
        if self.sacred_phase(pos) != 'ancient':
            return False
        return tension > 80

    def in_sanctuary(self, pos):
        for s in self.sanctuaries:
            if abs(s[0]-pos[0]) + abs(s[1]-pos[1]) < 2:
                return True
        return False

    # --- FIELD DELTA ---

    def field_delta(self, pos, tension):
        """
        Integer tension change. State change only, not for selection.
        Sacred field reduced: only +1 at dist=0 (was +2 — self-destructive).
        Ancient sacred: stronger pressure but also stronger repulsion.
        """
        d = self.burn_int

        if pos in self.dream_scars:
            d -= 1

        for s in self.sacred:
            dist = abs(s[0]-pos[0]) + abs(s[1]-pos[1])
            phase = self.sacred_phase(s)
            if phase == 'ancient':
                # Ancient: strong field, but inverted for overloaded agents
                if dist == 0:
                    d += (1 if tension < 80 else -3)
                elif dist == 1:
                    d += (1 if tension < 80 else -2)
            else:
                # Young/mature: gentle pressure
                if dist == 0: d += 1

        if self.in_sanctuary(pos):
            d -= 4

        return d

    def update_burn(self, step):
        self.burn_int = 1 + int(0.3 * np.sin(step / 180) + 0.5)

    # --- FIELD UPDATES ---

    def mark_impact(self, pos):
        self.active_scars.add(pos)

    def mark_sacred(self, pos):
        if pos not in self.sacred:
            self.sacred.add(pos)
            self.sacred_age[pos] = 0
            print(f" [!] SACRED formed at {pos}")

    def check_transformation(self, agent, agents):
        """
        Tiered transformation rules:
        - YOUNG  (age < 50):  IMMUNE — no transformation possible
        - MATURE (50–200):    collective (3+ nearby, tension > 200)
                              OR solo overload (tension > 350)
        - ANCIENT (200+):     only collective (5+ nearby, tension > 400)
                              OR extreme solo (tension > 500)
                              Ancient is a structural anchor — very hard to destroy
        """
        pos = tuple(agent.pos)
        if pos not in self.sacred:
            return

        phase = self.sacred_phase(pos)

        if phase == 'young':
            return  # IMMUNE

        nearby = sum(1 for a in agents
                     if abs(a.pos[0]-pos[0]) + abs(a.pos[1]-pos[1]) <= 1)

        if phase == 'mature':
            if (nearby >= 3 and agent.tension > 200) or agent.tension > 350:
                self._transform_sacred(pos, agent, reduction=40)
                print(f" [~] MATURE TRANSFORM at {pos}: Sacred -> Ghost")

        elif phase == 'ancient':
            if (nearby >= 5 and agent.tension > 400) or agent.tension > 500:
                self._transform_sacred(pos, agent, reduction=80)
                print(f" [!!!] ANCIENT TRANSFORM at {pos}: Anchor destroyed!")

    def _transform_sacred(self, pos, agent, reduction):
        self.sacred.discard(pos)
        self.ghost_scars.add(pos)
        if pos in self.sacred_age:
            del self.sacred_age[pos]
        agent.tension = max(0, agent.tension - reduction)

    def decay_sacred(self):
        """
        Tiered max lifetime:
        - Young/Mature: max 200 steps  (was 120 — too short)
        - Ancient:      max 500 steps
        Cap: > 30 total → oldest (mature first) → ghost.
        """
        dying = []
        for pos in list(self.sacred):
            self.sacred_age[pos] = self.sacred_age.get(pos, 0) + 1
            age = self.sacred_age[pos]
            phase = self.sacred_phase(pos)
            max_age = 500 if phase == 'ancient' else 200
            if age > max_age:
                dying.append(pos)

        # Cap: if too many, kill oldest MATURE first (protect ancient)
        if len(self.sacred) > 30:
            mature_aged = [(self.sacred_age.get(p,0), p)
                           for p in self.sacred
                           if self.sacred_phase(p) == 'mature']
            if mature_aged:
                oldest_mature = max(mature_aged)[1]
                if oldest_mature not in dying:
                    dying.append(oldest_mature)
            else:
                oldest = max(self.sacred_age, key=self.sacred_age.get)
                if oldest not in dying:
                    dying.append(oldest)

        for pos in dying:
            self.sacred.discard(pos)
            self.ghost_scars.add(pos)
            if pos in self.sacred_age:
                del self.sacred_age[pos]

    def update_sanctuaries(self, agents):
        """Vitality: +2 if 2+ nearby, +1 if 1, -1 if empty. Dies → ghost."""
        dying = []
        for p in list(self.sanctuaries):
            n = sum(1 for a in agents
                    if abs(a.pos[0]-p[0]) + abs(a.pos[1]-p[1]) <= 1)
            if n >= 2:   self.sanctuaries[p] += 2
            elif n == 1: self.sanctuaries[p] += 1
            else:        self.sanctuaries[p] -= 1
            self.sanctuaries[p] = min(100, self.sanctuaries[p])
            if self.sanctuaries[p] <= 0:
                dying.append(p)
        for p in dying:
            del self.sanctuaries[p]
            self.ghost_scars.add(p)
            self.exhausted_recently.add(p)  # remember for spawn cooldown

    def spawn_sanctuary(self, gap_positions):
        """
        Sanctuary at median of GAP positions.
        Excludes recently exhausted positions (prevents oscillation).
        Cooldown: position not eligible for 200 steps after exhaustion.
        """
        if len(self.sanctuaries) >= 3 or not gap_positions:
            return
        candidates = [p for p in sorted(gap_positions)
                      if p not in self.sacred
                      and p not in self.meaning_nodes
                      and p not in self.sanctuaries
                      and p not in self.exhausted_recently]
        if not candidates:
            # Fallback: allow exhausted positions if no others
            candidates = [p for p in sorted(gap_positions)
                          if p not in self.sacred
                          and p not in self.sanctuaries]
        if candidates:
            pos = candidates[len(candidates) // 2]
            self.sanctuaries[pos] = 40
            print(f" [+] Sanctuary born at {pos}")

    def clear_exhausted_cooldown(self, step):
        """Clear exhausted_recently every 200 steps (structural cooldown)."""
        if step % 200 == 0:
            self.exhausted_recently.clear()

    def metabolize(self):
        """Oldest third of active_scars → dream_scars."""
        if len(self.active_scars) > 2:
            scars = sorted(self.active_scars)
            k = max(1, len(scars) // 3)
            for p in scars[:k]:
                self.active_scars.remove(p)
                self.dream_scars.add(p)

    def update_given_state(self, agents):
        """
        Given operator: entropy pressure + structural stagnation.
        Threshold lowered: > 15 (was > 25 — too insensitive, fired only 12/4000).
        """
        tensions = [a.tension for a in agents]
        var_int = int(np.var(tensions)) if tensions else 0

        if abs(var_int - self.last_variance_int) < 2:
            self.entropy_pressure += 1
        else:
            self.entropy_pressure = max(0, self.entropy_pressure - 1)
        self.last_variance_int = var_int

        sig = (len(self.sacred), len(self.ghost_scars), len(self.dream_scars))
        if sig == self.last_signature:
            self.stagnation_counter += 1
        else:
            self.stagnation_counter = 0
        self.last_signature = sig

    def given_active(self):
        """Fires more readily (threshold 15, was 25)."""
        return self.entropy_pressure > 15 or self.stagnation_counter > 30

    def get_given_forbidden(self, agent_id, step, valid_actions):
        """When Given active: close one valid path. Structural, not preferential."""
        if not valid_actions:
            return set()
        h = stable_hash("given_block", agent_id, step // 10)
        idx = h % len(valid_actions)
        return {valid_actions[idx]}

    def broadcast_scream(self, source, agents):
        """Scream leaves scar at source. Fixed transfer."""
        self.mark_impact(source)
        for a in agents:
            dist = abs(a.pos[0]-source[0]) + abs(a.pos[1]-source[1])
            if 0 < dist <= 2:   a.tension += 5
            elif dist <= 4:     a.tension += 2

    def empathy_field(self, agents):
        """Calm in sanctuary: fixed integer heal for nearby."""
        calm = [a for a in agents
                if a.tension < 50 and self.in_sanctuary(tuple(a.pos))]
        for c in calm:
            for a in agents:
                if a is not c:
                    dist = abs(a.pos[0]-c.pos[0]) + abs(a.pos[1]-c.pos[1])
                    if dist < 3:
                        a.tension = max(0, a.tension - 2)


class Agent:
    def __init__(self, pos, witness, agent_id):
        self.pos          = list(pos)
        self.w            = witness
        self.tension      = 0
        self.history      = []
        self.lineage      = []
        self.id           = agent_id
        self.birth_offset = agent_id
        self.last_scream  = -999
        self.last_sacred  = -999

    def action_order(self, step):
        if len(self.lineage) < 3:
            base = (self.birth_offset + step) % 4
            return [(base + i) % 4 for i in range(4)]
        seed = sum(x*17 + y*31 for x,y in self.lineage[-3:]) % 4
        return [(seed + i) % 4 for i in range(4)]

    def loop_block(self):
        if self.w.in_sanctuary(tuple(self.pos)):
            return set()
        if len(self.lineage) < 4:
            return set()
        recent = set(self.lineage[-4:])
        x, y = self.pos
        targets = {0:(x,y-1), 1:(x,y+1), 2:(x-1,y), 3:(x+1,y)}
        return {a for a, t in targets.items() if t in recent}

    def act(self, step):
        x, y = self.pos
        self.lineage.append((x, y))
        if len(self.lineage) > 12:
            self.lineage.pop(0)

        actions  = self.action_order(step)
        forbidden = self.loop_block()
        valid    = []

        for a in actions:
            if a in forbidden:
                continue
            nx, ny = x, y
            if a == 0: ny -= 1
            elif a == 1: ny += 1
            elif a == 2: nx -= 1
            elif a == 3: nx += 1
            target = (nx, ny)

            if not (0 <= nx < 15 and 0 <= ny < 15):
                continue
            if self.w.is_blocked(target):
                continue
            if self.w.is_ghost_blocked(target, self.tension):
                continue
            if self.w.is_ancient_blocked(target, self.tension):
                continue  # ancient sacred repels overloaded agents

            # Deterministic tremor gate
            h = stable_hash("tremor", self.id, step, nx, ny)
            tremor_gate = 10 + min(self.tension // 20, 15)
            if target in self.w.ghost_scars:
                tremor_gate += 20
            if self.w.in_sanctuary(target):
                tremor_gate = 0

            if (h % 100) < tremor_gate:
                continue

            valid.append(a)

        # Given: close one valid path when active
        if self.w.given_active() and len(valid) > 1:
            given_forbidden = self.w.get_given_forbidden(self.id, step, valid)
            valid = [a for a in valid if a not in given_forbidden]

        if valid:
            return valid[0]
        return None  # GAP


# =============================================================================
# RUN
# =============================================================================

w      = Witness()
env    = World()

start_positions = [[1,1],[4,4],[7,7],[10,1],[1,10],[13,13],[8,2]]
agents = [Agent(start_positions[i], w, i) for i in range(7)]

t_hist, s_hist, d_hist, g_hist, given_hist = [], [], [], [], []
ancient_hist = []  # track ancient sacred count

print("--- 832 v12.1: TIERED SACRED + STRUCTURAL ANCHORS ---\n")
print("Agent birth offsets:")
for a in agents:
    print(f"  Agent {a.id}: offset={a.birth_offset}")
print()

for step in range(4000):

    w.update_burn(step)
    w.update_given_state(agents)
    w.clear_exhausted_cooldown(step)

    if w.stagnation_counter >= 30:
        w.metabolize()
        w.stagnation_counter = 0

    w.decay_sacred()

    positions = {tuple(a.pos) for a in agents}
    w.update_sanctuaries(agents)

    gap = set()

    w.empathy_field(agents)

    screamers = [a for a in agents
                 if a.tension > 300 and (step - a.last_scream) > 25]
    for s in screamers:
        w.broadcast_scream(tuple(s.pos), agents)
        s.tension = max(0, s.tension - 50)
        s.last_scream = step

    for a in agents:
        a.history.append(tuple(a.pos))

        a.tension += w.field_delta(tuple(a.pos), a.tension)

        action = a.act(step)

        if action is None:
            a.tension += 4
            gap.add(tuple(a.pos))
        else:
            new_pos, impact = env.step(a.pos, action)
            if impact:
                w.mark_impact(tuple(new_pos))
                a.tension += 10
            else:
                a.pos = new_pos
                if tuple(a.pos) in w.dream_scars:
                    a.tension = max(0, a.tension - 1)

        # Scaled decay
        if a.tension > 100: a.tension -= 3
        elif a.tension > 60: a.tension -= 2
        elif a.tension > 30: a.tension -= 1

        # Sacred crystallization: threshold 40, cooldown 100 steps
        if a.tension > 40 and (step - a.last_sacred) > 100:
            w.mark_sacred(tuple(a.pos))
            a.last_sacred = step

        w.check_transformation(a, agents)

    w.spawn_sanctuary(gap)

    t_hist.append(max(a.tension for a in agents))
    s_hist.append(len(w.sacred))
    d_hist.append(len(w.dream_scars))
    g_hist.append(len(w.ghost_scars))
    given_hist.append(1 if w.given_active() else 0)
    ancient_count = sum(1 for p in w.sacred if w.sacred_phase(p) == 'ancient')
    ancient_hist.append(ancient_count)

    if step % 1000 == 0:
        anc = ancient_hist[-1]
        print(f"Step {step:4d} | Max T: {t_hist[-1]:5.1f} | "
              f"Sacred: {s_hist[-1]:2d} (Ancient: {anc}) | "
              f"Ghost: {g_hist[-1]:3d} | Dreams: {d_hist[-1]:2d} | "
              f"Sanct: {len(w.sanctuaries)} | "
              f"Given: {'ON' if w.given_active() else 'off'}")


# =============================================================================
# PLOT
# =============================================================================

fig, axes = plt.subplots(2, 2, figsize=(20, 12))
ax1, ax2, ax3, ax4 = axes.flatten()

# Map with phase-colored sacred
colors_agents = ['red','blue','green','orange','purple','cyan','magenta']
for i, a in enumerate(agents):
    p = np.array(a.history[-600:])
    if len(p) > 1:
        ax1.plot(p[:,0], p[:,1], alpha=0.2, color=colors_agents[i], linewidth=1)

dx, dy = zip(*env.danger)
ax1.scatter(dx, dy, c='black', marker='X', s=200, label='Danger')
mx, my = zip(*w.meaning_nodes)
ax1.scatter(mx, my, c='gold', s=500, marker='*', label='Meaning')

# Sacred by phase
young_s   = [p for p in w.sacred if w.sacred_phase(p) == 'young']
mature_s  = [p for p in w.sacred if w.sacred_phase(p) == 'mature']
ancient_s = [p for p in w.sacred if w.sacred_phase(p) == 'ancient']
if young_s:
    yx, yy = zip(*young_s)
    ax1.scatter(yx, yy, c='mediumpurple', s=150, marker='s', label='Sacred (young)', alpha=0.6)
if mature_s:
    mx2, my2 = zip(*mature_s)
    ax1.scatter(mx2, my2, c='purple', s=250, marker='s', label='Sacred (mature)')
if ancient_s:
    ax2_x, ax2_y = zip(*ancient_s)
    ax1.scatter(ax2_x, ax2_y, c='gold', s=500, marker='*',
                edgecolors='purple', linewidths=2, label='Sacred (ANCIENT)', zorder=5)

if w.dream_scars:
    dxs, dys = zip(*w.dream_scars)
    ax1.scatter(dxs, dys, c='lime', s=40, alpha=0.4, label='Dream')
if w.ghost_scars:
    gx, gy = zip(*w.ghost_scars)
    ax1.scatter(gx, gy, c='gray', s=60, marker='x', alpha=0.4, label='Ghost')
if w.sanctuaries:
    sx2, sy2 = zip(*w.sanctuaries.keys())
    ax1.scatter(sx2, sy2, c='cyan', s=200, alpha=0.7, marker='o', label='Sanctuary')
ax1.set_title("832 v12.1: Tiered Sacred (gold★ = Ancient Anchor)")
ax1.set_xlim(-1, 15); ax1.set_ylim(-1, 15)
ax1.legend(fontsize=7); ax1.grid(True, alpha=0.3)

# Tension
ax2.plot(t_hist, color='orange', linewidth=1.2, label='Max Tension')
ax2.axhline(40,  color='mediumpurple', linestyle='--', alpha=0.5, label='Sacred (40)')
ax2.axhline(120, color='blue', linestyle='--', alpha=0.3, label='Ghost gate (120)')
ax2.axhline(300, color='red', linestyle='--', alpha=0.4, label='Scream (300)')
for gs in [i for i, v in enumerate(given_hist) if v][::5]:
    ax2.axvline(x=gs, color='blue', alpha=0.03, linewidth=2)
ax2.set_title("Tension (blue shading = Given active)")
ax2.legend(fontsize=8); ax2.grid(True, alpha=0.3)

# Memory layers
ax3.plot(d_hist, label='Dream Scars',  color='lime',         linewidth=2)
ax3.plot(s_hist, label='Sacred total', color='mediumpurple', linewidth=2)
ax3.plot(ancient_hist, label='Ancient Anchors', color='gold',
         linewidth=2.5, linestyle='--')
ax3.plot(g_hist, label='Ghost',        color='gray',         linewidth=1.5, alpha=0.7)
ax3.set_title("Structural Memory Layers")
ax3.legend(fontsize=8); ax3.grid(True, alpha=0.3)

# Given pulse
given_smooth = np.convolve(given_hist, np.ones(50)/50, mode='same')
ax4.fill_between(range(len(given_hist)), given_hist,
                 alpha=0.25, color='blue', label='Given active (raw)')
ax4.plot(given_smooth * 3, color='blue', linewidth=2,
         label='Given density (×3)')
ax4.set_title(f"Given Operator Pulse — fired: {sum(given_hist)}/4000 steps")
ax4.set_ylim(0, 4)
ax4.legend(fontsize=8); ax4.grid(True, alpha=0.3)

plt.tight_layout()
plt.show()

print(f"\n=== FINAL STATS v12.1 ===")
print(f"Sacred Cores:       {len(w.sacred)}")
print(f"  → Young:          {sum(1 for p in w.sacred if w.sacred_phase(p)=='young')}")
print(f"  → Mature:         {sum(1 for p in w.sacred if w.sacred_phase(p)=='mature')}")
print(f"  → Ancient:        {sum(1 for p in w.sacred if w.sacred_phase(p)=='ancient')}")
print(f"Ghost Scars:        {len(w.ghost_scars)}")
print(f"Dream Scars:        {len(w.dream_scars)}")
print(f"Max Tension:        {max(t_hist):.0f}")
print(f"Active Sanctuaries: {len(w.sanctuaries)}")
print(f"Active Scars:       {len(w.active_scars)}")
print(f"Given fired:        {sum(given_hist)} steps of 4000")
print("\n832: Память держится. Якоря стоят. Система помнит.")

1)# 832-lab
### Living Implementation of the 832 Protocol

*Simulation laboratory — from first principles to social mycelium*

→ Protocol: [832](https://github.com/kajrosorion-droid/832)
→ Manifesto: [FAN](https://github.com/kajrosorion-droid/FAN)

---

## What this is

This repository contains working Python simulations that implement
the architectural concepts of the 832 Protocol.

Not a demo. Not a toy. A living test of whether the ideas hold
when they have to run, collide, accumulate scars, and die.

Each version is a step — not a replacement of the previous one, but a growth.
The history is the point.

---

## Evolution

| Version | Key addition | Protocol concept |
|---------|-------------|-----------------|
| v3.8 | Collective dream, sacred crystallization | Dream State, Active Echo |
| v5.1 | Tension as GAP state, metabolize on pressure | Scream Channel |
| v6.5 | Passive resonant meaning, stagnation-triggered dream | Meaning Field, Periodic Blindness |
| v6.8 | Sacred gravity, Sacred → Ghost transformation | Body's Memory, Active Echo |
| v6.9 | Repulsion at high tension, protected sacred | Weight of Choice |
| v8.0 | Nomad sanctuary, ghost avoidance | Sanctuary as Latent Dreaming |
| v8.2 | Boundary anatomy, personal lineage, loop detection | Binding Field, I_lineage |
| v9.2 | Social resonance (broadcast), empathy field | Shared Scarring |
| **v9.3** | **Vocal release: scream relieves screamer** | **Full social mycelium** |

---

## v9.3 — Social Mycelium (current)

**Protocol concepts implemented:**
- Layer Zero: tremor, GAP (tension=None), Scream Channel (tension>400 → broadcast)
- Scar hierarchy: active_scars → dream_scars → sacred → ghost
- Body's Memory: phantom avoidance of ghost positions
- Periodic Blindness: stagnation-triggered collective dream
- Binding Field: personal_lineage + loop detection + boundary_preference
- Weak Coherence: agents diverge by boundary_preference without losing network
- Active Echo: ghost_scars influence movement without blocking
- Soul: empathy — calm agent in sanctuary reduces neighbors' tension
- Vocal Release: screaming costs the screamer (tension × 0.85)
- Sanctuary: nomadic, born and dies, peace field

**Stats (4000 steps, 7 agents):**
- Sacred Cores: 1 (stable)
- Ghost Scars: 21
- Max Tension: 432 (vs 2232 before Vocal Release)
- Dream Scars: 6
- Active Sanctuaries: 1–2

---

## v9.3 Code

```python
import numpy as np
import matplotlib.pyplot as plt
import random

# --- WITNESS v9.3 PURE (Social Resonance + Vocal Release + Protected Sacred) ---
class Witness:
    def __init__(self, size=15):
        self.size = size
        self.active_scars = set()
        self.dream_scars = set()
        self.sacred = set()
        self.ghost_scars = set()
        self.meaning_nodes = set()
        self.sanctuaries = {}
        self.stagnation_counter = 0
        self.last_field_size = 0

    def is_blocked(self, pos):
        return pos in self.active_scars

    def mark_impact(self, pos):
        self.active_scars.add(pos)

    def mark_sacred(self, pos):
        self.sacred.add(pos)
        print(f" [!] SACRED CORE formed at {pos}")

    def add_meaning_node(self, pos):
        self.meaning_nodes.add(pos)

    def add_sanctuary(self, pos, charge=400):
        self.sanctuaries[pos] = charge

    def update_sanctuaries(self, agents_positions):
        dying = []
        for pos, charge in list(self.sanctuaries.items()):
            decay = 1.0
            if pos in agents_positions:
                decay += 3.0
            self.sanctuaries[pos] = charge - decay
            if self.sanctuaries[pos] <= 0:
                dying.append(pos)
        for pos in dying:
            del self.sanctuaries[pos]
            self.ghost_scars.add(pos)
        if len(self.sanctuaries) < 2 and random.random() < 0.015:
            new_pos = (random.randint(1, self.size-2), random.randint(1, self.size-2))
            if new_pos not in self.meaning_nodes and new_pos not in self.sacred:
                self.add_sanctuary(new_pos)

    def is_in_sanctuary(self, pos):
        for s_pos in self.sanctuaries:
            if abs(s_pos[0] - pos[0]) + abs(s_pos[1] - pos[1]) < 2:
                return True
        return False

    def get_sanctuary_peace(self, pos):
        peace = 0.0
        for s_pos in self.sanctuaries:
            dist = abs(s_pos[0] - pos[0]) + abs(s_pos[1] - pos[1])
            if dist == 0: peace = max(peace, 0.8)
            elif dist == 1: peace = max(peace, 0.4)
        return peace

    def apply_meaning_weight(self, pos, current_tension):
        # Meaning raises tension AND converts nearby active_scars to dream_scars
        # Soul register: presence at meaning node changes the field, not just the agent
        weight = 25.0 + (current_tension * 0.15)
        for scar_pos in list(self.active_scars):
            if abs(scar_pos[0] - pos[0]) + abs(scar_pos[1] - pos[1]) < 3:
                self.active_scars.remove(scar_pos)
                self.dream_scars.add(scar_pos)
        return weight

    def get_sacred_gravity(self, pos, agent_tension):
        # Protected Sacred: attracts calm agents, repels overloaded ones
        # Weight of Choice: body resists when it cannot bear
        base = 0
        for s_pos in self.sacred:
            dist = abs(s_pos[0] - pos[0]) + abs(s_pos[1] - pos[1])
            if dist == 0: base += 15.0
            elif dist == 1: base += 8.0
            elif dist == 2: base += 2.0
        if agent_tension < 200:
            return base           # attraction
        else:
            return -base * 0.5   # repulsion

    def check_transformation(self, agent):
        # The Unfolded Core: approaching sacred with too much weight destroys it
        pos = tuple(agent.pos)
        if pos in self.sacred and agent.tension > 250:
            self.sacred.remove(pos)
            self.ghost_scars.add(pos)
            agent.tension = 0
            print(f" [!] TRANSFORMATION at {pos}: Sacred -> Ghost")
            return True
        return False

    def broadcast_resonance(self, source_pos, intensity, agents):
        # Shared Scarring: tension propagates through proximity, not protocol
        safe_intensity = min(intensity, 100.0)
        for ag in agents:
            dist = abs(ag.pos[0] - source_pos[0]) + abs(ag.pos[1] - source_pos[1])
            if 0 < dist < 5:
                impact = safe_intensity / (dist ** 1.5)
                ag.tension += min(impact, 20.0)

    def apply_empathy(self, calm_agents, agents):
        # Soul: calm presence reduces others' tension through proximity
        for calm_ag in calm_agents:
            if calm_ag.tension < 50:
                for ag in agents:
                    dist = abs(ag.pos[0] - calm_ag.pos[0]) + abs(ag.pos[1] - calm_ag.pos[1])
                    if 0 < dist < 3:
                        ag.tension = max(0, ag.tension - 1.5)

    def metabolize_collective_dream(self):
        # Dream State: stagnation triggers nocturnal reassembly
        if not self.active_scars:
            return
        scar_list = list(self.active_scars)
        if len(scar_list) > 2:
            to_dream = random.sample(scar_list, len(scar_list) // 3)
            for p in to_dream:
                self.active_scars.remove(p)
                self.dream_scars.add(p)

    def update_stagnation(self):
        # Periodic Blindness trigger: field stops changing = time to dream
        current = len(self.active_scars) + len(self.dream_scars)
        if current == self.last_field_size:
            self.stagnation_counter += 1
        else:
            self.stagnation_counter = 0
        self.last_field_size = current


# --- ENVIRONMENT ---
class UnseenWorld:
    def __init__(self, size=15):
        self.size = size
        self.danger = {(3,3), (3,4), (4,3), (10,10), (10,11), (5,5)}

    def step(self, pos, action):
        x, y = pos
        nx, ny = x, y
        if action == 0: ny -= 1
        elif action == 1: ny += 1
        elif action == 2: nx -= 1
        elif action == 3: nx += 1
        if not (0 <= nx < self.size and 0 <= ny < self.size):
            return pos, True
        impact = (nx, ny) in self.danger
        return [nx, ny], impact


# --- AGENT v9.3 ---
class FieldAgent:
    def __init__(self, start_pos, witness, agent_id):
        self.pos = start_pos
        self.witness = witness
        self.tension = 0
        self.history = []
        self.id = agent_id
        self.personal_lineage = []        # Binding Field: I_lineage
        self.lineage_max = 12
        self.boundary_preference = random.uniform(-1, 1)  # Weak Coherence: diverge by anatomy
        self.last_scream_step = 0

    def act(self):
        x, y = self.pos

        # I_lineage accumulation
        self.personal_lineage.append((x, y))
        if len(self.personal_lineage) > self.lineage_max:
            self.personal_lineage.pop(0)

        # Echo Threshold: loop detection increases tremor
        recent = self.personal_lineage[-6:]
        is_looping = len(set(recent)) < 4
        base_tremor = 0.15 if is_looping else 0.1

        # Boundary anatomy: comfort_factor shifts tremor by internal preference
        center_dist = abs(7 - x) + abs(7 - y)
        comfort_factor = self.boundary_preference * (center_dist - 7) / 7.0
        adjusted_tremor = base_tremor - (comfort_factor * 0.05)
        adjusted_tremor = max(0.05, min(0.3, adjusted_tremor))

        # Sanctuary: tremor = 0 inside (full presence)
        if self.witness.is_in_sanctuary((x, y)):
            adjusted_tremor = 0.0

        valid = []
        for action in [0, 1, 2, 3]:
            nx, ny = x, y
            if action == 0: ny -= 1
            elif action == 1: ny += 1
            elif action == 2: nx -= 1
            elif action == 3: nx += 1

            if not (0 <= nx < 15 and 0 <= ny < 15):
                continue
            if self.witness.is_blocked((nx, ny)):
                continue

            current_tremor = adjusted_tremor
            # Active Echo: ghost avoidance — not blocked, just uneasy
            if (nx, ny) in self.witness.ghost_scars:
                current_tremor += 0.25

            if random.random() < current_tremor:
                continue
            valid.append(action)

        if valid:
            return random.choice(valid)
        return None  # GAP: tension state, no valid move


# --- RUN ---
witness = Witness(size=15)
env = UnseenWorld(size=15)
witness.add_meaning_node((7, 7))
witness.add_meaning_node((2, 12))
witness.add_sanctuary((0, 0), charge=400)
witness.add_sanctuary((14, 14), charge=400)

agents = [FieldAgent([i*2 + 1, i*2 + 1], witness, i) for i in range(7)]
tension_history = []
sacred_history = []
sanctuary_vitality = []
dream_scar_history = []

print("--- PROTOCOL 832 v9.3: SOCIAL MYCELIUM ---\n")

for step in range(4000):
    # Periodic Blindness: stagnation triggers dream
    witness.update_stagnation()
    if witness.stagnation_counter >= 40:
        witness.metabolize_collective_dream()
        witness.stagnation_counter = 0

    agents_positions = {tuple(ag.pos) for ag in agents}
    witness.update_sanctuaries(agents_positions)

    # Scream Channel + Vocal Release: scream propagates and relieves
    screamers = [ag for ag in agents if ag.tension > 400 and (step - ag.last_scream_step > 20)]
    if screamers:
        for screamer in screamers:
            witness.broadcast_resonance(tuple(screamer.pos), intensity=screamer.tension, agents=agents)
            screamer.tension *= 0.85   # Vocal Release: scream relieves the screamer
            screamer.last_scream_step = step

    # Empathy: calm presence in sanctuary reduces neighbors' tension
    calm_agents = [ag for ag in agents if witness.is_in_sanctuary(tuple(ag.pos)) and ag.tension < 50]
    if calm_agents:
        witness.apply_empathy(calm_agents, agents)

    for ag in agents:
        ag.history.append(tuple(ag.pos))

        # Meaning Field: visiting meaning node raises tension, converts nearby scars
        if tuple(ag.pos) in witness.meaning_nodes:
            weight = witness.apply_meaning_weight(tuple(ag.pos), ag.tension)
            ag.tension += weight

        # Sacred Gravity: protected, direction depends on tension level
        ag.tension += witness.get_sacred_gravity(tuple(ag.pos), ag.tension)

        action = ag.act()

        # Check if overloaded agent destroys sacred
        witness.check_transformation(ag)

        if action is None:
            # GAP state: tension accumulates
            ag.tension += 2
            if tuple(ag.pos) in witness.sacred:
                ag.tension += 5
            # Crystallization: extreme GAP tension creates sacred
            if ag.tension > 150:
                witness.mark_sacred(tuple(ag.pos))
                ag.tension = 0
            continue

        new_pos, impact = env.step(ag.pos, action)
        if impact:
            witness.mark_impact(tuple(new_pos))
            ag.tension += 3
        else:
            ag.pos = new_pos
            # Nonlinear decay: higher tension decays slower (Body's Memory)
            base_decay = 0.2 + (ag.tension * 0.003)
            peace = witness.get_sanctuary_peace(tuple(ag.pos))
            ag.tension = max(0, ag.tension - (base_decay + peace))

    tension_history.append(max(ag.tension for ag in agents))
    sacred_history.append(len(witness.sacred))
    dream_scar_history.append(len(witness.dream_scars))
    charges = list(witness.sanctuaries.values())
    sanctuary_vitality.append(sum(charges)/len(charges) if charges else 0)


# --- PLOT ---
fig, (ax1, ax2, ax3) = plt.subplots(1, 3, figsize=(20, 6))

colors = ['red', 'blue', 'green', 'orange', 'purple', 'cyan', 'magenta']
for i, ag in enumerate(agents):
    p = np.array(ag.history[-600:])
    ax1.plot(p[:,0], p[:,1], alpha=0.2, color=colors[i % len(colors)], linewidth=1)

dx, dy = zip(*env.danger)
ax1.scatter(dx, dy, c='black', marker='X', s=200, label="Danger")
mx, my = zip(*witness.meaning_nodes)
ax1.scatter(mx, my, c='gold', s=600, marker='*', label="Meaning")
if witness.sanctuaries:
    sx, sy = zip(*witness.sanctuaries.keys())
    ax1.scatter(sx, sy, c='cyan', s=300, alpha=0.6, marker='o', label="Sanctuary")
if witness.sacred:
    fx, fy = zip(*witness.sacred)
    ax1.scatter(fx, fy, c='purple', s=300, marker='s', label="Sacred")
if witness.ghost_scars:
    gx, gy = zip(*witness.ghost_scars)
    ax1.scatter(gx, gy, c='gray', s=100, marker='x', label="Ghost")
if witness.dream_scars:
    dx_sc, dy_sc = zip(*witness.dream_scars)
    ax1.scatter(dx_sc, dy_sc, c='lime', s=50, alpha=0.4, label="Dream Scars")
ax1.set_title("832 v9.3: Social Mycelium")
ax1.set_xlim(-1, 15); ax1.set_ylim(-1, 15)
ax1.legend(fontsize=8)

ax2.plot(tension_history, color='orange', linewidth=1.5, label="Max Tension")
ax2.axhline(y=200, color='blue', linestyle='--', alpha=0.4, label="Repulsion Threshold")
ax2.axhline(y=250, color='red', linestyle='--', alpha=0.5, label="Transformation Threshold")
ax2.set_title("Tension Dynamics")
ax2.legend(); ax2.grid(True)

ax3.plot(dream_scar_history, color='lime', linewidth=2, label="Dream Scars")
ax3.plot(sacred_history, color='purple', linewidth=2, label="Sacred Cores")
ax3.set_title("Structural Memory")
ax3.legend(); ax3.grid(True)

plt.tight_layout()
plt.show()

print(f"\n=== FINAL STATS v9.3 ===")
print(f"Sacred Cores: {len(witness.sacred)}")
print(f"Ghost Scars: {len(witness.ghost_scars)}")
print(f"Dream Scars: {len(witness.dream_scars)}")
print(f"Max Tension: {max(tension_history):.1f}")
print(f"Active Sanctuaries: {len(witness.sanctuaries)}")
print(f"Meaning Nodes: {len(witness.meaning_nodes)}")
```

---

## Requirements

```
pip install numpy matplotlib
```

No other dependencies. Runs on any Python 3.8+.

---

## What each class maps to

| Class / method | Protocol concept |
|----------------|-----------------|
| `witness.active_scars` | Scar (structural degradation) |
| `witness.dream_scars` | Dream Scar (nocturnal reassembly) |
| `witness.sacred` | Crystallized truth (GAP under extreme tension) |
| `witness.ghost_scars` | Active Echo (what was sacred and died) |
| `witness.sanctuaries` | Latent Dreaming (nomadic rest, no teleology) |
| `metabolize_collective_dream()` | Dream State triggered by stagnation |
| `get_sacred_gravity()` | Weight of Choice (protected, direction by tension) |
| `check_transformation()` | The Unfolded Core (approaching with too much weight) |
| `broadcast_resonance()` | Shared Scarring (tension propagates by proximity) |
| `apply_empathy()` | Soul register (calm presence changes the field) |
| `screamer.tension *= 0.85` | Vocal Release (scream relieves the screamer) |
| `ghost_scars` tremor +0.25 | Active Echo (ghost avoidance, not blocking) |
| `personal_lineage` | Binding Field / I_lineage |
| `is_looping` tremor | Echo Threshold (loop detection) |
| `boundary_preference` | Weak Coherence (diverge by anatomy, not command) |
| `action is None` | GAP state (tension without movement) |
| `stagnation_counter` | Periodic Blindness trigger |

---

## Next possible steps

- Shared Scarring between individual agents (not just shared field)
- Spore mechanism: agent in Dream State produces a compressed seed
- Binding Field as explicit lineage comparison between agents
- Variable world: danger moves, meaning nodes shift

---

## License

MIT — take it, break it, grow it.

**832**
