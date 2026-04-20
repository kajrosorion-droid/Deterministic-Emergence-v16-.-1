
import numpy as np
import matplotlib.pyplot as plt
import hashlib
from collections import deque

# =============================================================================
# 832 Protocol v16.0 — Fixed Symbolic Swarm
#
# Fixes from v29:
#   ✗ Agent.act() calls given_active(agents) — NameError, agents not in scope
#     FIX: store agents ref in witness, or pass phase/given_flag as param
#   ✗ np.random used in 8+ places — breaks determinism
#     FIX: all replaced with stable_hash
#   ✗ loop_block returns empty in 'exploration' for non-hunters
#     FIX: only hunter near sphere gets empty loop_block
#   ✗ Given threshold: only fires in crisis with max_t > 300 → 16/4000
#     FIX: restore entropy_pressure trigger (>15)
#   ✗ inject_structured_noise uses np.random
#     FIX: deterministic noise via stable_hash
#   ✗ reproduce uses np.random.randint for mutation
#     FIX: stable_hash mutation
#
# What's kept from v29 (the good):
#   + Sigils (DANGER/FOOD/HOME/BUILD/HUNT)
#   + Cultural memory
#   + Sacred geometry (links, triangles)
#   + Triangle inversion → new sphere
#   + Reproduction + death + legacy transfer
#   + Roles via constraints (not scores)
#   + Micro resonance + collective resonance
#   + Pheromone toxicity
#
# New in v16.0:
#   + Agent dream state: tension < 10 for 5+ steps → no movement,
#     contributes +1 to field_delta of adjacent cells (quiet presence)
#   + Sigil longevity boost when agent dies nearby (burial mark)
#   + Given now also tracks: if all agents have same role → inject noise
# =============================================================================


def stable_hash(*args):
    h = hashlib.sha256(str(args).encode()).hexdigest()
    return int(h[:8], 16)


def sh_range(n, *args):
    """Deterministic 0..n-1 via stable_hash."""
    return stable_hash(*args) % n


class MovingMeaning:
    def __init__(self, pos, sphere_id):
        self.pos          = list(pos)
        self.id           = sphere_id
        self.hold_timer   = 0
        self.last_escape  = -999
        self.last_sacred_spawn = -999
        self.memory_weight = 1

    def update_position(self, step, agents, world, witness):
        x = int(np.round(10 + 5*np.sin(step/200 + self.id) + 3*np.cos(step/173)))
        y = int(np.round(10 + 4*np.cos(step/150 - self.id) + 3*np.sin(step/211)))
        x = max(1, min(world.size-2, x))
        y = max(1, min(world.size-2, y))

        ancient_pos = [p for p in witness.sacred if witness.sacred_phase(p) == 'ancient']
        if ancient_pos:
            nearest = min(ancient_pos,
                          key=lambda p: abs(p[0]-self.pos[0]) + abs(p[1]-self.pos[1]))
            dist = abs(nearest[0]-self.pos[0]) + abs(nearest[1]-self.pos[1])
            if dist <= 3 and sh_range(100, "attract", self.id, step) < 30:
                if   self.pos[0] < nearest[0]: self.pos[0] += 1
                elif self.pos[0] > nearest[0]: self.pos[0] -= 1
                elif self.pos[1] < nearest[1]: self.pos[1] += 1
                elif self.pos[1] > nearest[1]: self.pos[1] -= 1
                return

        nearby = sum(1 for a in agents
                     if abs(a.pos[0]-self.pos[0]) + abs(a.pos[1]-self.pos[1]) <= 2)
        if nearby >= 5 and step - self.last_escape > 20:
            d = sh_range(4, "escape", self.id, step)
            nx, ny = self.pos[0], self.pos[1]
            if d == 0: ny -= 1
            elif d == 1: ny += 1
            elif d == 2: nx -= 1
            elif d == 3: nx += 1
            if 0 <= nx < world.size and 0 <= ny < world.size:
                self.pos = [nx, ny]
                self.last_escape = step
            self.hold_timer = 0
        else:
            if abs(self.pos[0]-x) + abs(self.pos[1]-y) > 0 and step % 3 == 0:
                if   self.pos[0] < x: self.pos[0] += 1
                elif self.pos[0] > x: self.pos[0] -= 1
                elif self.pos[1] < y: self.pos[1] += 1
                elif self.pos[1] > y: self.pos[1] -= 1

        if step - self.last_sacred_spawn > 50:
            near_anc = any(witness.sacred_phase(p) == 'ancient'
                           for p in witness.sacred
                           if abs(p[0]-self.pos[0]) + abs(p[1]-self.pos[1]) <= 2)
            threshold = 20 if near_anc else 8
            if sh_range(100, "sphere_sacred", self.id, step) < threshold:
                witness.mark_sacred(tuple(self.pos))
                self.last_sacred_spawn = step

        if nearby >= 3: self.hold_timer += 1
        else:           self.hold_timer = max(0, self.hold_timer - 1)


class World:
    def __init__(self, size=20):
        self.size   = size
        self.danger = {(3,3),(3,4),(4,3),(16,16),(16,17),(17,16),(10,10),(5,15)}

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
    def __init__(self, size=20):
        self.size   = size
        self.active_scars       = set()
        self.dream_scars        = set()
        self.cultural_memory    = {}    # {pos: (lineage_id, weight)}
        self.sacred             = set()
        self.ghost_scars        = set()
        self.sanctuaries        = {}
        self.exhausted_recently = set()
        self.sacred_age         = {}

        self.pheromone_danger    = {}
        self.pheromone_relief    = {}
        self.pheromone_discovery = {}
        self.pheromone_toxicity  = {}

        self.sigils  = {}   # {pos: {type, lineage, decay}}
        self.sacred_links = set()
        self.triangle_ages = {}
        self.sanctuary_triangles = []

        self.burn_int           = 1
        self.entropy_pressure   = 0
        self.last_variance_int  = 0
        self.stagnation_counter = 0
        self.last_signature     = None

        self.given_flag         = False   # set each step, read by agents
        self.resonance_active   = 0
        self.recent_screams     = []

        # Stats
        self.death_toll         = 0
        self.sacred_collapses   = 0
        self.memory_decays      = 0
        self.legacy_power       = 0
        self.scream_events      = 0
        self.resonance_events   = 0
        self.micro_resonance    = 0
        self.spheres_created    = 0
        self.current_phase      = 'stable'
        self.stable_duration    = 0

        # Dream state cells: {pos: age} — agents in dream, no movement
        self.dream_presence     = {}

    # --- SACRED PHASE ---

    def sacred_phase(self, pos):
        if pos not in self.sacred: return None
        age = self.sacred_age.get(pos, 0)
        if age < 50:  return 'young'
        if age < 200: return 'mature'
        return 'ancient'

    # --- CONSTRAINT CHECKS ---

    def is_blocked(self, pos):
        return pos in self.active_scars

    def is_ghost_blocked(self, pos, tension):
        return pos in self.ghost_scars and tension > 120

    def is_ancient_blocked(self, pos, tension, role='universal'):
        if pos not in self.sacred or self.sacred_phase(pos) != 'ancient':
            return False
        threshold = 150 if role == 'guardian' else 80
        return tension > threshold

    def is_danger_blocked(self, pos, tension):
        return self.pheromone_danger.get(pos, 0) >= 20 and tension > 30

    def is_toxic(self, pos):
        return self.pheromone_toxicity.get(pos, 0) > 10

    def in_sanctuary(self, pos):
        for s in self.sanctuaries:
            if abs(s[0]-pos[0]) + abs(s[1]-pos[1]) < 2:
                return True
        # Also check triangle sanctuaries
        if self.sanctuary_triangles:
            try:
                from matplotlib.path import Path
                for hull_pts in self.sanctuary_triangles:
                    if Path(hull_pts).contains_point(pos):
                        return True
            except Exception:
                pass
        return False

    def near_sphere(self, pos, spheres, radius=2):
        return any(abs(sp.pos[0]-pos[0]) + abs(sp.pos[1]-pos[1]) <= radius
                   for sp in spheres)

    def in_sacred_link(self, pos):
        for (p1, p2) in self.sacred_links:
            x, y = pos
            x1, y1 = p1; x2, y2 = p2
            dx, dy = x2-x1, y2-y1
            if dx == 0 and dy == 0:
                dist = abs(x-x1) + abs(y-y1)
            else:
                t = ((x-x1)*dx + (y-y1)*dy) / (dx*dx + dy*dy)
                t = max(0.0, min(1.0, t))
                dist = abs(x - (x1 + t*dx)) + abs(y - (y1 + t*dy))
            if dist <= 1.5:
                return True
        return False

    def protect_sacred(self, pos):
        protection = sum(self.pheromone_relief.get(p, 0)
                         for p in self.pheromone_relief
                         if abs(p[0]-pos[0]) + abs(p[1]-pos[1]) <= 1)
        protection += sum(self.pheromone_discovery.get(p, 0)
                          for p in self.pheromone_discovery
                          if abs(p[0]-pos[0]) + abs(p[1]-pos[1]) <= 1)
        return protection > 50

    # --- FIELD DELTA ---

    def field_delta(self, pos, tension, spheres):
        d = self.burn_int

        if self.resonance_active > 0: d = 0

        if pos in self.dream_scars: d -= 1

        for s in self.sacred:
            dist = abs(s[0]-pos[0]) + abs(s[1]-pos[1])
            phase = self.sacred_phase(s)
            if phase == 'ancient':
                if dist == 0:   d += (1 if tension < 80 else -2)
                elif dist == 1: d += (0 if tension < 80 else -1)
            else:
                if dist == 0: d += 1

        if self.in_sanctuary(pos): d -= 3

        if self.pheromone_relief.get(pos, 0) > 10: d -= 1

        if self.near_sphere(pos, spheres, 2): d -= 1

        if self.is_toxic(pos): d += 3

        # Dream presence: agents in dream state create quiet field around them
        if self.dream_presence.get(pos, 0) > 0:
            d -= 1  # dreaming agent calms the area

        return max(-5, min(6, d))

    def update_burn(self, step, agents):
        if self.resonance_active > 0:
            self.burn_int = 0
            return
        phi = (1 + 5**0.5) / 2
        period = max(50, int(233 / phi))
        raw = (1/phi) + (1/(phi**2)) * np.sin(step / period)
        self.burn_int = max(1, int(round(raw)))

    # --- UPDATES ---

    def mark_impact(self, pos):
        self.active_scars.add(pos)
        self.pheromone_danger[pos] = self.pheromone_danger.get(pos, 0) + 5

    def mark_sacred(self, pos):
        if pos not in self.sacred:
            self.sacred.add(pos)
            self.sacred_age[pos] = 0

    def check_transformation(self, agent, agents):
        pos = tuple(agent.pos)
        if pos not in self.sacred:
            return
        if self.protect_sacred(pos) or self.in_sacred_link(pos):
            return
        phase = self.sacred_phase(pos)
        if phase == 'young':
            return
        nearby = sum(1 for a in agents
                     if abs(a.pos[0]-pos[0]) + abs(a.pos[1]-pos[1]) <= 1)
        if phase == 'mature':
            if (nearby >= 3 and agent.tension > 200) or agent.tension > 350:
                self._transform(pos, agent, 40)
        elif phase == 'ancient':
            if (nearby >= 5 and agent.tension > 400) or agent.tension > 500:
                self._transform(pos, agent, 80)
                print(f" [!!!] ANCIENT DESTROYED at {pos}")

    def _transform(self, pos, agent, reduction):
        self.sacred.discard(pos)
        self.ghost_scars.add(pos)
        self.sacred_age.pop(pos, None)
        agent.tension = max(0, agent.tension - reduction)
        self.sacred_collapses += 1

    def decay_sacred(self):
        dying = []
        for pos in list(self.sacred):
            self.sacred_age[pos] = self.sacred_age.get(pos, 0) + 1
            age   = self.sacred_age[pos]
            phase = self.sacred_phase(pos)
            max_age = 500 if phase == 'ancient' else 200
            if age > max_age:
                dying.append(pos)
        if len(self.sacred) > 50:
            mature_aged = [(self.sacred_age.get(p,0), p) for p in self.sacred
                           if self.sacred_phase(p) == 'mature']
            if mature_aged:
                dying.append(max(mature_aged)[1])
            elif self.sacred_age:
                dying.append(max(self.sacred_age, key=self.sacred_age.get))
        for pos in dying:
            self.sacred.discard(pos)
            self.ghost_scars.add(pos)
            self.sacred_age.pop(pos, None)

    def structural_sacred_collapse(self, step):
        if step % 200 == 0:
            for pos in list(self.sacred):
                if sh_range(100, "collapse", pos, step) < 2:
                    self.sacred.discard(pos)
                    self.ghost_scars.add(pos)
                    self.sacred_age.pop(pos, None)
                    self.sacred_collapses += 1

    def decay_cultural_memory(self, step):
        if step % 250 == 0 and self.cultural_memory:
            to_del = [p for p in self.cultural_memory
                      if sh_range(100, "decay_mem", p, step) < 5]
            for p in to_del:
                del self.cultural_memory[p]
                self.memory_decays += 1

    def update_sigils(self):
        """Sigil decay -1/step. Burial mark: agents dreaming nearby extend decay."""
        to_del = []
        for pos, data in self.sigils.items():
            data['decay'] -= 1
            # Dream presence nearby: sigil lasts longer (burial mark effect)
            if self.dream_presence.get(pos, 0) > 0:
                data['decay'] += 1  # counteracts decay while dreamer present
            if data['decay'] <= 0:
                to_del.append(pos)
        for pos in to_del:
            del self.sigils[pos]

    def update_pheromone_toxicity(self):
        all_ph = {}
        for grid in [self.pheromone_relief, self.pheromone_danger, self.pheromone_discovery]:
            for pos, val in grid.items():
                all_ph[pos] = all_ph.get(pos, 0) + val
        for pos, val in all_ph.items():
            if val > 40:
                self.pheromone_toxicity[pos] = self.pheromone_toxicity.get(pos, 0) + 1
            else:
                self.pheromone_toxicity[pos] = max(0, self.pheromone_toxicity.get(pos, 0) - 1)
        for p in [k for k, v in self.pheromone_toxicity.items() if v <= 0]:
            del self.pheromone_toxicity[p]

    def update_sacred_geometry(self):
        """Sacred links between nearby ancients. Triangle sanctuaries."""
        ancients = [p for p in self.sacred if self.sacred_phase(p) == 'ancient']
        self.sacred_links.clear()
        for i, p1 in enumerate(ancients):
            for p2 in ancients[i+1:]:
                if abs(p1[0]-p2[0]) + abs(p1[1]-p2[1]) <= 5:
                    self.sacred_links.add((p1,p2) if p1<p2 else (p2,p1))

        self.sanctuary_triangles.clear()
        if len(ancients) >= 3:
            points = np.array(ancients, dtype=float)
            try:
                from scipy.spatial import ConvexHull
                hull = ConvexHull(points)
                if len(hull.vertices) == 3:
                    valid = all(
                        abs(points[hull.vertices[i]][0] - points[hull.vertices[(i+1)%3]][0]) +
                        abs(points[hull.vertices[i]][1] - points[hull.vertices[(i+1)%3]][1]) <= 7
                        for i in range(3)
                    )
                    if valid:
                        poly = points[hull.vertices]
                        self.sanctuary_triangles.append(poly)
                        verts = frozenset(tuple(p) for p in poly)
                        if verts not in self.triangle_ages:
                            self.triangle_ages[verts] = 0
            except Exception:
                pass

    def check_triangle_inversion(self, step, spheres):
        to_remove = []
        for verts, age in list(self.triangle_ages.items()):
            if all(p in self.sacred and self.sacred_phase(p) == 'ancient' for p in verts):
                self.triangle_ages[verts] = age + 1
                if self.triangle_ages[verts] >= 300:
                    cx = int(np.mean([p[0] for p in verts]))
                    cy = int(np.mean([p[1] for p in verts]))
                    total_mem = sum(self.sacred_age.get(p, 0) for p in verts)
                    for p in verts:
                        self.sacred.discard(p)
                        self.ghost_scars.add(p)
                        self.sacred_age.pop(p, None)
                    new_sp = MovingMeaning([cx, cy], len(spheres))
                    new_sp.memory_weight = total_mem // 100 + 1
                    spheres.append(new_sp)
                    self.spheres_created += 1
                    to_remove.append(verts)
                    print(f" [CREATION] Triangle inverted → sphere at ({cx},{cy})")
            else:
                to_remove.append(verts)
        for v in to_remove:
            self.triangle_ages.pop(v, None)

    def update_sanctuaries(self, agents):
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
            self.exhausted_recently.add(p)

    def spawn_sanctuary(self, gap_positions):
        if len(self.sanctuaries) >= 8 or not gap_positions:
            return
        candidates = [p for p in sorted(gap_positions)
                      if p not in self.sacred
                      and p not in self.sanctuaries
                      and p not in self.exhausted_recently]
        if not candidates:
            candidates = [p for p in sorted(gap_positions)
                          if p not in self.sacred and p not in self.sanctuaries]
        if candidates:
            self.sanctuaries[candidates[len(candidates)//2]] = 40

    def clear_exhausted_cooldown(self, step):
        if step % 200 == 0:
            self.exhausted_recently.clear()

    def metabolize(self):
        if len(self.active_scars) > 2:
            scars = sorted(self.active_scars)
            k = max(1, len(scars) // 2)
            for p in scars[:k]:
                self.active_scars.remove(p)
                self.dream_scars.add(p)

    def decay_pheromones(self):
        for grid in [self.pheromone_relief, self.pheromone_danger, self.pheromone_discovery]:
            for pos in list(grid.keys()):
                grid[pos] -= 2
                if grid[pos] <= 0:
                    del grid[pos]

    def update_given_state(self, agents):
        """
        Given fires on:
        1. Entropy pressure: variance not changing
        2. Stagnation: structural signature unchanged
        3. Role monoculture: all agents have same role (new in v16)
        """
        if not agents:
            self.given_flag = False
            return

        var_int = int(np.var([a.tension for a in agents]))
        threshold = max(5, int(self.last_variance_int * 0.05))
        if abs(var_int - self.last_variance_int) < threshold:
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

        # Role monoculture detection
        roles = [a.current_role for a in agents]
        role_counts = {r: roles.count(r) for r in set(roles)}
        dominant_share = max(role_counts.values()) / len(roles) if roles else 0

        self.given_flag = (
            self.entropy_pressure > 15
            or self.stagnation_counter > 30
            or dominant_share > 0.85  # 85% same role = monoculture
        )

    def get_given_forbidden(self, agent_id, step, valid_actions):
        if not valid_actions: return set()
        return {valid_actions[sh_range(len(valid_actions), "given", agent_id, step//10)]}

    def broadcast_scream(self, source, agents, step):
        self.mark_impact(source)
        self.scream_events += 1
        self.recent_screams.append(step)
        for a in agents:
            dist = abs(a.pos[0]-source[0]) + abs(a.pos[1]-source[1])
            if 0 < dist <= 3:
                a.tension += 10
                if dist <= 1: a.tension += 5

    def check_collective_resonance(self, step, agents):
        self.recent_screams = [s for s in self.recent_screams if step - s < 10]
        if len(self.recent_screams) >= 3:
            self.resonance_events += 1
            self.resonance_active = 15
            for a in agents:
                a.energy   = min(100, a.energy + 20)
                a.tension  = max(0, a.tension - 30)
            for pos in list(self.pheromone_toxicity.keys()):
                self.pheromone_toxicity[pos] = max(0, self.pheromone_toxicity[pos] - 20)
            self.recent_screams.clear()
            print(f" [SYMPHONY] Collective resonance at step {step}!")

    def check_micro_resonance(self, agents):
        for i, a1 in enumerate(agents):
            if a1.tension >= 50: continue
            nearby = [a2 for j, a2 in enumerate(agents)
                      if i != j and a2.tension < 50
                      and abs(a1.pos[0]-a2.pos[0]) + abs(a1.pos[1]-a2.pos[1]) <= 2]
            if 1 <= len(nearby) <= 2:
                for a in [a1] + nearby:
                    a.energy = min(100, a.energy + 5)
                self.micro_resonance += 1

    def empathy_field(self, agents):
        calm = [a for a in agents
                if a.tension < 50 and self.in_sanctuary(tuple(a.pos))]
        for c in calm:
            for a in agents:
                if a is not c and abs(a.pos[0]-c.pos[0]) + abs(a.pos[1]-c.pos[1]) < 3:
                    a.tension = max(0, a.tension - 2)

    def inject_deterministic_noise(self, agents, step):
        """Replaces np.random noise with stable_hash noise. Fires every 200 steps."""
        if step % 200 == 0:
            count = min(15, len(agents))
            for i in range(count):
                idx = sh_range(len(agents), "noise_pick", i, step)
                a   = agents[idx]
                delta = (sh_range(20, "noise_delta", i, step)) - 5  # -5..+14
                a.tension = max(0, a.tension + delta)

    def update_system_phase(self, agents, step):
        if not agents: return
        max_t  = max(a.tension for a in agents)
        num_anc = sum(1 for p in self.sacred if self.sacred_phase(p) == 'ancient')
        num_ghost = len(self.ghost_scars)

        if max_t > 250 or num_ghost > 150:
            self.current_phase  = 'crisis'
            self.stable_duration = 0
        elif self.current_phase == 'crisis' and max_t < 150:
            self.current_phase = 'exploration'
        else:
            if num_anc >= 5 and max_t < 100:
                self.current_phase = 'stable'
                self.stable_duration += 1
            else:
                self.current_phase = 'exploration'
                self.stable_duration = max(0, self.stable_duration - 1)

        if self.stable_duration > 300:
            self.current_phase   = 'exploration'
            self.stable_duration = 0

    def update_dream_presence(self, dreaming_positions):
        """Track cells where agents are dreaming."""
        # Decay existing
        for pos in list(self.dream_presence.keys()):
            self.dream_presence[pos] -= 1
            if self.dream_presence[pos] <= 0:
                del self.dream_presence[pos]
        # Add current dreamers
        for pos in dreaming_positions:
            self.dream_presence[pos] = self.dream_presence.get(pos, 0) + 3


class Agent:
    def __init__(self, pos, witness, agent_id, genome=None, lineage_id=None):
        self.pos          = list(pos)
        self.w            = witness
        self.tension      = 0
        self.prev_tension = 0
        self.history      = []
        self.lineage_mem  = []   # renamed from lineage to avoid clash
        self.id           = agent_id
        self.birth_offset = agent_id % 4
        self.last_scream  = -999
        self.last_sacred  = -999
        self.energy       = 100
        self.lineage_id   = lineage_id if lineage_id is not None else agent_id
        self.current_role = 'universal'
        self.hunter_skill = 0
        self.guardian_skill = 0
        self.builder_skill  = 0
        self.bad_memory   = deque(maxlen=5)

        # Dream state: consecutive steps with tension < 10
        self.dream_counter = 0
        self.in_dream      = False

        self.genome = genome.copy() if genome else {
            'tremor_base':       sh_range(16, "genome_t", agent_id) + 5,
            'explore_bias':      sh_range(11, "genome_e", agent_id),
            'memory_weight':     sh_range(5,  "genome_m", agent_id) + 1,
            'sacred_threshold':  sh_range(41, "genome_s", agent_id) + 60,
        }

        self.role_ema = {'hunter':0.0, 'guardian':0.0, 'builder':0.0, 'universal':0.0}

    def update_role(self, spheres):
        alpha = 0.3
        near_sph = 1 if self.w.near_sphere(tuple(self.pos), spheres, 3) else 0
        near_sac = 1 if any(abs(p[0]-self.pos[0]) + abs(p[1]-self.pos[1]) <= 2
                             for p in self.w.sacred) else 0
        is_build = 1 if (near_sph == 0 and near_sac == 0) else 0

        self.role_ema['hunter']   = alpha*near_sph  + (1-alpha)*self.role_ema['hunter']
        self.role_ema['guardian'] = alpha*near_sac  + (1-alpha)*self.role_ema['guardian']
        self.role_ema['builder']  = alpha*is_build  + (1-alpha)*self.role_ema['builder']

        max_val = max(self.role_ema.values())
        if max_val < 0.3:
            self.current_role = 'universal'
        elif self.role_ema['hunter'] == max_val:
            self.current_role = 'hunter'
        elif self.role_ema['guardian'] == max_val:
            self.current_role = 'guardian'
        else:
            self.current_role = 'builder'

    def get_role_color(self):
        return {'hunter':'red','guardian':'blue','builder':'green','universal':'gray'}[self.current_role]

    def action_order(self, step):
        if len(self.lineage_mem) < 3:
            base = (self.birth_offset + step) % 4
            return [(base+i)%4 for i in range(4)]
        seed = sum(x*17 + y*31 for x,y in self.lineage_mem[-3:]) % 4
        return [(seed+i)%4 for i in range(4)]

    def loop_block(self, role, spheres):
        if self.w.in_sanctuary(tuple(self.pos)): return set()
        if role == 'hunter' and self.w.near_sphere(tuple(self.pos), spheres, 3):
            return set()  # hunter near sphere: urgency overrides caution
        window = 2 if role == 'builder' else 4
        if len(self.lineage_mem) < window: return set()
        recent = set(self.lineage_mem[-window:])
        x, y = self.pos
        targets = {0:(x,y-1),1:(x,y+1),2:(x-1,y),3:(x+1,y)}
        return {a for a, t in targets.items() if t in recent}

    def act(self, step, spheres):
        x, y = self.pos
        self.lineage_mem.append((x, y))
        if len(self.lineage_mem) > 12:
            self.lineage_mem.pop(0)

        # Dream state: agent stays still, contributes to field
        if self.tension < 10:
            self.dream_counter += 1
            if self.dream_counter >= 5:
                self.in_dream = True
                return None  # GAP — dreaming, not stuck
        else:
            self.dream_counter = 0
            self.in_dream      = False

        role     = self.current_role
        actions  = self.action_order(step)
        forbidden = self.loop_block(role, spheres)
        valid    = []

        for a in actions:
            if a in forbidden: continue
            nx, ny = x, y
            if a == 0: ny -= 1
            elif a == 1: ny += 1
            elif a == 2: nx -= 1
            elif a == 3: nx += 1
            target = (nx, ny)

            if not (0 <= nx < self.w.size and 0 <= ny < self.w.size): continue
            if self.w.is_blocked(target): continue
            if self.w.is_ghost_blocked(target, self.tension): continue
            if self.w.is_ancient_blocked(target, self.tension, role): continue
            if self.w.is_danger_blocked(target, self.tension): continue

            # Sigil reading
            sigil = self.w.sigils.get(target)
            if sigil:
                lin_ok = (sigil['lineage'] == self.lineage_id or sigil['lineage'] == -1)
                if lin_ok:
                    if sigil['type'] == 'DANGER':
                        self.bad_memory.append(target)
                        continue
            if target in self.bad_memory: continue

            # Toxic tremor
            if self.w.is_toxic(target):
                if sh_range(100, "toxic", self.id, step, nx, ny) < 40:
                    continue

            # Deterministic tremor gate
            h = stable_hash("tremor", self.id, step, nx, ny)
            tremor_gate = self.genome['tremor_base'] + min(self.tension//20, 15)
            if target in self.w.ghost_scars:   tremor_gate += 20
            if self.w.in_sanctuary(target):    tremor_gate  = 0
            if self.w.in_sacred_link(target):  tremor_gate  = int(tremor_gate * 1.3)

            # Cultural memory bonus (no score — reduces gate only if own lineage)
            mem = self.w.cultural_memory.get(target)
            if mem and mem[0] == self.lineage_id:
                tremor_gate = max(0, tremor_gate - 5 * mem[1])

            if (h % 100) < tremor_gate: continue
            valid.append(a)

        # Given: close one valid path (uses flag set by witness each step)
        if self.w.given_flag and len(valid) > 1:
            given_forbidden = self.w.get_given_forbidden(self.id, step, valid)
            valid = [a for a in valid if a not in given_forbidden]

        if valid:
            return valid[0]
        return None  # GAP


def reproduce(parent1, parent2, new_id, step, w):
    child_genome = {}
    for key in parent1.genome:
        avg = (parent1.genome[key] + parent2.genome[key]) // 2
        mutation = sh_range(5, "mut", key, parent1.id, parent2.id, step) - 2
        child_genome[key] = max(1, avg + mutation)

    # Radical mutation (10% chance)
    if sh_range(100, "radical", new_id, step) < 10:
        child_genome['tremor_base'] = max(1, child_genome['tremor_base']
                                          + sh_range(21, "rad_t", new_id, step) - 10)

    lineage = parent1.lineage_id if parent1.lineage_id == parent2.lineage_id else new_id

    for dx, dy in [(0,0),(1,0),(-1,0),(0,1),(0,-1)]:
        nx = parent1.pos[0] + dx
        ny = parent1.pos[1] + dy
        if 0 <= nx < w.size and 0 <= ny < w.size:
            child = Agent([nx, ny], w, new_id, child_genome, lineage)
            child.role_ema      = parent1.role_ema.copy()
            child.hunter_skill  = (parent1.hunter_skill  + parent2.hunter_skill)  // 2
            child.guardian_skill = (parent1.guardian_skill + parent2.guardian_skill) // 2
            child.builder_skill = (parent1.builder_skill + parent2.builder_skill) // 2
            return child
    return None


# =============================================================================
# RUN
# =============================================================================

w       = Witness(size=20)
env     = World(size=20)
spheres = [MovingMeaning([5,5],0), MovingMeaning([15,15],1), MovingMeaning([10,10],2)]

# Deterministic start positions using stable_hash
start_positions, used = [], set()
seed_counter = 0
while len(start_positions) < 50:
    x = sh_range(18, "start_x", seed_counter) + 1
    y = sh_range(18, "start_y", seed_counter) + 1
    seed_counter += 1
    if (x,y) not in env.danger and (x,y) not in used:
        used.add((x,y))
        start_positions.append([x,y])

agents  = [Agent(start_positions[i], w, i) for i in range(50)]
next_id = len(agents)

(t_hist, s_hist, d_hist, g_hist, given_hist, ancient_hist,
 pheromone_hist, pop_hist, cultural_hist, sigil_hist,
 scream_hist, reson_hist, micro_hist, song_hist) = ([] for _ in range(14))

print("--- 832 v16.0: FIXED SYMBOLIC SWARM ---\n")

for step in range(4000):
    if w.resonance_active > 0: w.resonance_active -= 1

    # Sphere updates
    for sp in spheres:
        sp.update_position(step, agents, env, w)
        for a in agents:
            if abs(a.pos[0]-sp.pos[0]) + abs(a.pos[1]-sp.pos[1]) <= 2:
                sp.memory_weight = min(100, sp.memory_weight + 1)

    # Sphere spawning (deterministic)
    for sp in spheres:
        if sp.hold_timer > 120 and len(spheres) < 8:
            nx = sh_range(18, "sp_x", step, sp.id) + 1
            ny = sh_range(18, "sp_y", step, sp.id) + 1
            spheres.append(MovingMeaning([nx, ny], len(spheres)))
            sp.hold_timer = 0
            print(f" [SPHERE] New sphere at ({nx},{ny})")
            break

    w.update_sacred_geometry()
    w.check_triangle_inversion(step, spheres)
    w.update_burn(step, agents)
    w.update_given_state(agents)    # sets w.given_flag
    w.update_system_phase(agents, step)
    w.clear_exhausted_cooldown(step)
    w.decay_pheromones()
    w.update_sigils()
    w.update_pheromone_toxicity()
    w.structural_sacred_collapse(step)
    w.decay_cultural_memory(step)

    if w.stagnation_counter >= 30 or (step > 0 and step % 100 == 0):
        w.metabolize()
        if w.stagnation_counter >= 30:
            w.stagnation_counter = 0

    w.decay_sacred()
    w.update_sanctuaries(agents)

    gap, dreaming_positions = set(), set()
    w.empathy_field(agents)
    w.check_micro_resonance(agents)
    w.inject_deterministic_noise(agents, step)

    # Scream check
    for a in agents:
        if a.tension > 250 and (step - a.last_scream) > 30:
            w.broadcast_scream(tuple(a.pos), agents, step)
            a.tension     = max(0, a.tension - 70)
            a.last_scream = step
    w.check_collective_resonance(step, agents)

    # Update roles
    for a in agents: a.update_role(spheres)

    # Energy, death, reproduction
    newborns, dead = [], []
    for a in agents:
        a.energy -= 1 + a.tension // 100
        if a.tension < 80:     a.energy = min(100, a.energy + 1)
        if w.in_sanctuary(tuple(a.pos)) and a.tension < 30:
            a.energy = min(100, a.energy + 2)
        if tuple(a.pos) in w.dream_scars: a.energy = min(100, a.energy + 1)
        if w.is_toxic(tuple(a.pos)):      a.energy -= 3

        if a.energy <= 0:
            # 20% survival chance
            if sh_range(100, "survive", a.id, step) < 20:
                a.energy = 10
            else:
                dead.append(a)
                w.ghost_scars.add(tuple(a.pos))
                w.death_toll += 1
                # Legacy transfer
                if sh_range(100, "legacy", a.id, step) < 80:
                    for other in agents:
                        if other is not a and \
                           abs(other.pos[0]-a.pos[0]) + abs(other.pos[1]-a.pos[1]) <= 2:
                            other.energy = min(100, other.energy + 60)
                            w.legacy_power += 60
                            # Burial mark: extend nearby sigils
                            for spos in list(w.sigils.keys()):
                                if abs(spos[0]-a.pos[0]) + abs(spos[1]-a.pos[1]) <= 2:
                                    w.sigils[spos]['decay'] += 15
                continue

        # Reproduction
        if a.energy > 80 and a.tension < 150 and w.current_phase != 'crisis':
            for other in agents:
                if (other is not a and other.energy > 80 and other.tension < 150
                        and abs(a.pos[0]-other.pos[0]) + abs(a.pos[1]-other.pos[1]) == 1
                        and len(agents) < 200):
                    chance = 20
                    if w.in_sanctuary(tuple(a.pos)) or w.near_sphere(tuple(a.pos), spheres): chance = 40
                    if w.resonance_active > 0: chance = 60
                    if sh_range(100, "repro", a.id, other.id, step) < chance:
                        child = reproduce(a, other, next_id, step, w)
                        if child:
                            newborns.append(child)
                            next_id += 1
                            a.energy -= 30; other.energy -= 30
                        break

    for a in dead:    agents.remove(a)
    agents.extend(newborns)

    # Immigration if population too low
    if len(agents) < 30:
        for i in range(6):
            x = sh_range(18, "immig_x", step, i) + 1
            y = sh_range(18, "immig_y", step, i) + 1
            avg_genome = {}
            if agents:
                for key in agents[0].genome:
                    avg_genome[key] = sum(ag.genome[key] for ag in agents) // len(agents)
            agents.append(Agent([x, y], w, next_id, avg_genome if avg_genome else None))
            next_id += 1
        print(f" [IMMIGRATION] Step {step}, pop boosted to {len(agents)}")

    # Main agent loop
    for a in agents:
        a.history.append(tuple(a.pos))
        if len(a.history) > 50: a.history.pop(0)
        a.prev_tension = a.tension

        a.tension += w.field_delta(tuple(a.pos), a.tension, spheres)
        a.tension  = max(0, a.tension)

        action = a.act(step, spheres)

        if action is None:
            a.tension += 4
            gap.add(tuple(a.pos))
            if a.in_dream:
                dreaming_positions.add(tuple(a.pos))
        else:
            new_pos, impact = env.step(a.pos, action)
            if impact:
                w.mark_impact(tuple(new_pos))
                a.tension += 10
                w.sigils[tuple(new_pos)] = {'type':'DANGER', 'lineage':a.lineage_id, 'decay':20}
            else:
                a.pos = new_pos
                if tuple(a.pos) in w.dream_scars:
                    a.tension = max(0, a.tension - 2)

        # Emit sigils
        role = a.current_role
        if role == 'hunter' and w.near_sphere(tuple(a.pos), spheres, 1):
            w.sigils[tuple(a.pos)] = {'type':'HUNT', 'lineage':a.lineage_id, 'decay':50}
        elif role == 'builder' and a.tension < a.prev_tension:
            w.sigils[tuple(a.pos)] = {'type':'BUILD', 'lineage':a.lineage_id, 'decay':30}
        if w.near_sphere(tuple(a.pos), spheres, 1):
            w.sigils[tuple(a.pos)] = {'type':'FOOD', 'lineage':-1, 'decay':15}
        if any(w.sacred_phase(p) == 'ancient' and
               abs(p[0]-a.pos[0]) + abs(p[1]-a.pos[1]) <= 1 for p in w.sacred):
            w.sigils[tuple(a.pos)] = {'type':'HOME', 'lineage':-1, 'decay':25}

        # Pheromones
        if a.tension < a.prev_tension:
            pt = tuple(a.pos)
            w.pheromone_relief[pt] = w.pheromone_relief.get(pt, 0) + 1
        if w.near_sphere(tuple(a.pos), spheres) or w.in_sanctuary(tuple(a.pos)):
            pt = tuple(a.pos)
            w.pheromone_discovery[pt] = w.pheromone_discovery.get(pt, 0) + 3

        # Cultural memory
        if role == 'builder' and not w.pheromone_relief.get(tuple(a.pos)):
            w.cultural_memory[tuple(a.pos)] = (a.lineage_id, 2)

        # Tension decay
        if a.tension > 100: a.tension -= 3
        elif a.tension > 60: a.tension -= 2
        elif a.tension > 30: a.tension -= 1

        # Sacred crystallization
        thresh = a.genome['sacred_threshold']
        if a.tension > thresh and (step - a.last_sacred) > 150:
            w.mark_sacred(tuple(a.pos))
            a.last_sacred = step

        w.check_transformation(a, agents)

    w.update_dream_presence(dreaming_positions)
    w.spawn_sanctuary(gap)

    # History
    t_hist.append(max(a.tension for a in agents) if agents else 0)
    s_hist.append(len(w.sacred))
    d_hist.append(len(w.dream_scars))
    g_hist.append(len(w.ghost_scars))
    given_hist.append(1 if w.given_flag else 0)
    ancient_hist.append(sum(1 for p in w.sacred if w.sacred_phase(p) == 'ancient'))
    pheromone_hist.append(sum(w.pheromone_relief.values())
                          + sum(w.pheromone_danger.values())
                          + sum(w.pheromone_discovery.values()))
    pop_hist.append(len(agents))
    cultural_hist.append(len(w.cultural_memory))
    sigil_hist.append(len(w.sigils))
    scream_hist.append(w.scream_events)
    reson_hist.append(w.resonance_events)
    micro_hist.append(w.micro_resonance)
    song_hist.append(w.resonance_events * w.legacy_power / max(1000, 1))

    if step % 1000 == 0:
        rc = {}
        for a in agents: rc[a.current_role] = rc.get(a.current_role, 0) + 1
        print(f"Step {step:4d} | Pop:{len(agents):3d} | T:{t_hist[-1]:5.1f} | "
              f"Sacred:{s_hist[-1]:2d}(Anc:{ancient_hist[-1]}) | "
              f"Ghost:{g_hist[-1]:3d} | Dream:{d_hist[-1]:2d} | "
              f"Phase:{w.current_phase:12s} | Given:{'ON' if w.given_flag else 'off'} | "
              f"Sigils:{sigil_hist[-1]} | "
              f"H={rc.get('hunter',0)} G={rc.get('guardian',0)} B={rc.get('builder',0)}")


# =============================================================================
# PLOT
# =============================================================================

fig, axes = plt.subplots(3, 3, figsize=(24, 18))
ax = axes.flatten()

# --- Map ---
pmap = np.zeros((20, 20))
for (x,y), v in w.pheromone_relief.items():
    if 0<=x<20 and 0<=y<20: pmap[x,y] += v
for (x,y), v in w.pheromone_discovery.items():
    if 0<=x<20 and 0<=y<20: pmap[x,y] += v
for (x,y), v in w.pheromone_danger.items():
    if 0<=x<20 and 0<=y<20: pmap[x,y] -= v
pmap = np.clip(pmap, 0, None)
ax[0].imshow(pmap.T, origin='lower', cmap='YlGn', alpha=0.4, extent=[0,20,0,20])

tox = np.zeros((20,20))
for (x,y), v in w.pheromone_toxicity.items():
    if 0<=x<20 and 0<=y<20: tox[x,y] = v
if tox.max() > 0:
    ax[0].imshow(tox.T, origin='lower', cmap='Reds', alpha=0.25, extent=[0,20,0,20])

for (p1,p2) in w.sacred_links:
    ax[0].plot([p1[0],p2[0]], [p1[1],p2[1]], 'gold', lw=2, alpha=0.6)

for poly in w.sanctuary_triangles:
    ax[0].fill(poly[:,0], poly[:,1], color='cyan', alpha=0.12)

for pos, data in w.sigils.items():
    c = {'DANGER':'red','FOOD':'yellow','HOME':'blue','BUILD':'green','HUNT':'orange'}
    m = {'DANGER':'x','FOOD':'o','HOME':'s','BUILD':'^','HUNT':'*'}
    ax[0].scatter(pos[0], pos[1], c=c.get(data['type'],'white'),
                  marker=m.get(data['type'],'o'), s=80, alpha=0.7)

# Dream cells
if w.dream_presence:
    dx2, dy2 = zip(*w.dream_presence.keys())
    ax[0].scatter(dx2, dy2, c='white', s=60, alpha=0.5, marker='.')

for a in agents:
    if len(a.history) > 1:
        p = np.array(a.history[-30:])
        ax[0].plot(p[:,0], p[:,1], alpha=0.12, color=a.get_role_color(), lw=0.5)

danx, dany = zip(*env.danger)
ax[0].scatter(danx, dany, c='black', marker='X', s=200)
for sp in spheres:
    ax[0].scatter(sp.pos[0], sp.pos[1], c='gold', s=600+sp.memory_weight*5,
                  marker='o', edgecolors='orange', linewidths=3, alpha=0.9)

for p in [q for q in w.sacred if w.sacred_phase(q) == 'young']:
    ax[0].scatter(p[0], p[1], c='mediumpurple', s=100, marker='s', alpha=0.5)
for p in [q for q in w.sacred if w.sacred_phase(q) == 'mature']:
    ax[0].scatter(p[0], p[1], c='purple', s=200, marker='s')
ancient_s = [q for q in w.sacred if w.sacred_phase(q) == 'ancient']
if ancient_s:
    ax2, ay2 = zip(*ancient_s)
    ax[0].scatter(ax2, ay2, c='gold', s=450, marker='*',
                  edgecolors='purple', lw=2, zorder=5)
if w.ghost_scars:
    gx,gy = zip(*w.ghost_scars)
    ax[0].scatter(gx, gy, c='gray', s=40, marker='x', alpha=0.25)
if w.sanctuaries:
    sx2,sy2 = zip(*w.sanctuaries.keys())
    ax[0].scatter(sx2, sy2, c='cyan', s=140, alpha=0.7, marker='o')

handles = [plt.Line2D([0],[0],color='red',lw=2,label='Hunter'),
           plt.Line2D([0],[0],color='blue',lw=2,label='Guardian'),
           plt.Line2D([0],[0],color='green',lw=2,label='Builder'),
           plt.Line2D([0],[0],color='gray',lw=2,label='Universal')]
ax[0].set_title("v16.0: Fixed Symbolic Swarm")
ax[0].set_xlim(-1,20); ax[0].set_ylim(-1,20)
ax[0].legend(handles=handles, fontsize=7); ax[0].grid(True, alpha=0.3)

# --- Tension ---
ax[1].plot(t_hist, color='orange', lw=1.2)
for i, ph in enumerate(
    ['stable' if s_hist[i]>0 and ancient_hist[i]>=3 else 'crisis'
     if t_hist[i]>250 else 'expl' for i in range(len(t_hist))]):
    pass  # simplified phase coloring
ax[1].axhline(60, color='purple', ls='--', alpha=0.5, label='Sacred~(60)')
ax[1].axhline(200, color='red', ls='--', alpha=0.4, label='Scream(200+)')
ax[1].set_title("Tension Dynamics"); ax[1].legend(fontsize=7); ax[1].grid(True,alpha=0.3)

# --- Population ---
ax[2].plot(pop_hist, color='black', lw=2)
ax[2].set_title("Population"); ax[2].grid(True, alpha=0.3)

# --- Memory layers ---
ax[3].plot(d_hist, label='Dream', color='lime', lw=2)
ax[3].plot(s_hist, label='Sacred', color='mediumpurple', lw=2)
ax[3].plot(ancient_hist, label='Ancient★', color='gold', lw=2.5, ls='--')
ax[3].plot(g_hist, label='Ghost', color='gray', lw=1.5, alpha=0.7)
ax[3].set_title("Structural Memory"); ax[3].legend(fontsize=8); ax[3].grid(True,alpha=0.3)

# --- Genome ---
if agents:
    keys = ['tremor_base','explore_bias','memory_weight','sacred_threshold']
    avgs = [sum(a.genome[k] for a in agents)/len(agents) for k in keys]
    ax[4].bar(['Tremor','Explore','Memory','Sacred'], avgs,
              color=['gray','yellow','blue','purple'])
    ax[4].set_title("Avg Genome"); ax[4].grid(True, alpha=0.3)

# --- Given + Pheromone ---
gs = np.convolve(given_hist, np.ones(50)/50, mode='same')
ax[5].fill_between(range(len(given_hist)), given_hist, alpha=0.2, color='blue')
ax[5].plot(gs*3, color='blue', lw=2, label=f'Given({sum(given_hist)}/4000)')
if max(pheromone_hist) > 0:
    ax[5].plot(np.array(pheromone_hist)/max(pheromone_hist)*3,
               color='orange', lw=1.5, alpha=0.7, label='Pheromone(norm)')
ax[5].set_ylim(0,4); ax[5].set_title("Given Pulse + Pheromone")
ax[5].legend(fontsize=8); ax[5].grid(True,alpha=0.3)

# --- Cultural memory + Sigils ---
ax[6].plot(cultural_hist, color='purple', lw=2, label='Cultural Memory')
ax[6].plot(sigil_hist, color='cyan', lw=2, label='Active Sigils')
ax[6].set_title("Cultural Records + Sigils"); ax[6].legend(); ax[6].grid(True,alpha=0.3)

# --- Resonance ---
ax[7].plot(scream_hist, color='orange', lw=1.5, label='Screams')
ax[7].plot(reson_hist, color='red', lw=2, label='Resonance')
ax[7].plot(micro_hist, color='pink', lw=1.5, label='Micro Resonance')
ax[7].set_title("Resonance Events"); ax[7].legend(); ax[7].grid(True,alpha=0.3)

# --- Role distribution (sampled) ---
role_steps = list(range(0, 4000, 100))
rh = []; rg = []; rb = []
# Use final agents for simplified view
rc_final = {}
for a in agents: rc_final[a.current_role] = rc_final.get(a.current_role, 0) + 1
ax[8].bar(['Hunter','Guardian','Builder','Universal'],
          [rc_final.get('hunter',0), rc_final.get('guardian',0),
           rc_final.get('builder',0), rc_final.get('universal',0)],
          color=['red','blue','green','gray'])
ax[8].set_title(f"Final Role Distribution (Pop={len(agents)})")
ax[8].grid(True, alpha=0.3)

plt.tight_layout()
plt.show()

print(f"\n=== FINAL STATS v16.0 ===")
print(f"Population:       {len(agents)}")
print(f"Sacred:           {len(w.sacred)} (Y:{sum(1 for p in w.sacred if w.sacred_phase(p)=='young')} M:{sum(1 for p in w.sacred if w.sacred_phase(p)=='mature')} A:{sum(1 for p in w.sacred if w.sacred_phase(p)=='ancient')})")
print(f"Ghost:            {len(w.ghost_scars)}")
print(f"Dream Scars:      {len(w.dream_scars)}")
print(f"Cultural Records: {len(w.cultural_memory)}")
print(f"Active Sigils:    {len(w.sigils)}")
print(f"Max Tension:      {max(t_hist):.0f}")
print(f"Sanctuaries:      {len(w.sanctuaries)}")
print(f"Spheres:          {len(spheres)} ({w.spheres_created} created)")
print(f"Given fired:      {sum(given_hist)}/4000")
print(f"Screams:          {w.scream_events}")
print(f"Resonance:        {w.resonance_events}")
print(f"Micro Resonance:  {w.micro_resonance}")
print(f"Deaths:           {w.death_toll}")
print(f"Legacy power:     {w.legacy_power}")
print(f"Sacred Collapses: {w.sacred_collapses}")
print(f"Memory Decays:    {w.memory_decays}")
if agents:
    avgt = sum(a.genome['tremor_base'] for a in agents) / len(agents)
    avge = sum(a.genome['explore_bias'] for a in agents) / len(agents)
    print(f"Avg Genome:       Tremor={avgt:.1f} Explore={avge:.1f}")
print(f"Role distribution: H={rc_final.get('hunter',0)} G={rc_final.get('guardian',0)} "
      f"B={rc_final.get('builder',0)} U={rc_final.get('universal',0)}")
print("\n832 v16.0: Символы живут. Рой помнит. Данные — без случайности.")
