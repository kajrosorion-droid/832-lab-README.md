# 832-lab
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
