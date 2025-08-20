# Mock-Auction-basic===============================

Auction Engine Monolith

===============================

import random
import re
import json
import os
import requests
from bs4 import BeautifulSoup
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Tuple
from enum import Enum


---

util helpers


---

CRORE = 1.0  # represent all amounts in crores as float

def fmt_cr(amount: float) -> str:
return f"{amount:.2f}Cr"

def round_to_increment(amount: float) -> float:
if amount <= 9.0:
tick = 0.25
return round(amount / tick) * tick
else:
tick = 0.5
return round(amount / tick) * tick

def next_increment(current: float) -> float:
return 0.25 if current < 9.0 else 0.5

def clamp(value: float, lo: float, hi: float) -> float:
return max(lo, min(hi, value))

def in_band(value: float, lo: float, hi: float) -> bool:
return lo <= value <= hi

def safe_money(value: float) -> float:
return round(value + 1e-9, 2)


---

models


---

class Role(str, Enum):
B = "B"   # Batter
W = "W"   # Wicket-keeper
A = "A"   # Allrounder
P = "P"   # Pace
S = "S"   # Spin

TIERS = [
(1, 0.30, 1.50),
(2, 1.50, 2.75),
(3, 3.00, 4.75),
(4, 5.00, 6.50),
(5, 7.00, 8.50),
(6, 9.00,10.50),
(7,11.00,12.50),
(8,13.00,15.50),
(9,16.00,18.00),
]

@dataclass
class Player:
name: str
slot: str
base_price: float
previous_team: Optional[str]
age: Optional[int]
role: Optional[Role]
tier_id: Optional[int] = None
tier_lo: Optional[float] = None
tier_hi: Optional[float] = None
hidden_reserve: Optional[float] = None
stats: Dict[str, str] = field(default_factory=dict)

@dataclass
class Team:
name: str
total_slots: int
overseas_slots: int
purse: float
rtm_available: int
squad: List[Player] = field(default_factory=list)
overseas_count: int = 0

def can_afford(self, price: float) -> bool:
return self.purse >= price

def has_slot(self) -> bool:
return len(self.squad) < self.total_slots

def add_player(self, player: Player, price: float, is_overseas: bool):
self.squad.append(player)
self.purse = round(self.purse - price, 2)
if is_overseas:
self.overseas_count += 1

@dataclass
class Constraints:
min_B: int = 5
min_PS: int = 5
min_A: int = 4
min_W: int = 2

@dataclass
class Bid:
team: str
amount: float

@dataclass
class SaleResult:
player: Player
sold_to: Optional[str]
price: Optional[float]
role: Role
rtm_used: bool = False
reason: Optional[str] = None

@dataclass
class AuctionState:
teams: Dict[str, Team]
constraints: Constraints
human_team: str
sold_log: List[SaleResult] = field(default_factory=list)
unsold_stack: List[Player] = field(default_factory=list)
batch_size: int = 8


---

rules


---

def base_tier_for_role(role: Role) -> int:
mapping = {
Role.A: 5,
Role.W: 4,
Role.B: 3,
Role.P: 4,
Role.S: 4,
}
return mapping.get(role, 3)

def apply_age_adjustments(base_tier: int, age: int) -> int:
t = base_tier
if age > 36:
t -= 2
elif 25 <= age <= 33:
t += 1
return max(1, min(9, t))

def tier_band(tier_id: int) -> Tuple[float,float]:
for tid, lo, hi in TIERS:
if tid == tier_id:
return (lo, hi)
raise ValueError("Invalid tier")

def snap_to_increment(amount: float) -> float:
step = 0.25 if amount <= 9.0 else 0.5
steps = int(amount / step + 1e-9)
return round(steps * step, 2)

def assign_tier_and_reserve(player: Player, rng: random.Random) -> Player:
if player.role is None or player.age is None:
return player
base = base_tier_for_role(player.role)
final_tier = apply_age_adjustments(base, player.age)
lo, hi = tier_band(final_tier)
anchor = max(player.base_price, lo)
reserve = rng.uniform(anchor, hi)
reserve = snap_to_increment(reserve)
player.tier_id = final_tier
player.tier_lo = lo
player.tier_hi = hi
player.hidden_reserve = reserve
return player


---

AI Policy


---

@dataclass
class AITendencies:
aggressiveness: float
jitter_prob: float
reserve_respect: float

def role_need_weight(team: Team, role: Role) -> float:
counts = {r: sum(1 for p in team.squad if p.role == r) for r in Role}
need = 0.5
if role == Role.B and counts[Role.B] < 5: need += 0.4
if role in (Role.P, Role.S) and (counts[Role.P]+counts[Role.S]) < 5: need += 0.4
if role == Role.A and counts[Role.A] < 4: need += 0.5
if role == Role.W and counts[Role.W] < 2: need += 0.4
remaining_slots = team.total_slots - len(team.squad)
if remaining_slots > 0 and team.purse/remaining_slots > 6.0: need += 0.1
return min(1.3, need)

def ai_should_bid(team: Team, player: Player, current_bid: Optional[Bid], first_bidder: bool, rng: random.Random) -> Optional[float]:
cap = player.tier_hi or 0.0
base = player.base_price
need_weight = role_need_weight(team, player.role or Role.B)
hidden = player.hidden_reserve or base
target = min(cap, max(base, hidden * (0.9 + 0.3 * rng.random())))
target *= (0.7 + 0.6 * need_weight)
target = min(cap, target)
if rng.random() < 0.25:
target = min(cap, target + (0.25 if target < 9 else 0.5))
if current_bid is None:
opening = max(base, min(target, cap))
if team.can_afford(opening) and team.has_slot():
return round(opening, 2)
return None
next_bid = safe_money(current_bid.amount + next_increment(current_bid.amount))
if next_bid <= cap and team.can_afford(next_bid) and team.has_slot() and next_bid <= target:
return next_bid
return None


---

Wikipedia & Cricmetric Enricher


---

def enrich_player(player: Player) -> Player:
try:

Fetch from Wikipedia

url = f"https://en.wikipedia.org/wiki/{player.name.replace(' ', '_')}"
res = requests.get(url, timeout=10)
if res.status_code == 200:
soup = BeautifulSoup(res.text, "html.parser")
infobox = soup.find("table", {"class": "infobox"})
if infobox:
text = infobox.get_text(" ")

crude parse: look for "Born" year

m = re.search(r"Born.*?(\d{4})", text)
if m:
birth_year = int(m.group(1))
player.age = 2025 - birth_year
if "Batting" in text or "Bowling" in text:
if "Bowling" in text and "Batting" in text:
player.role = Role.A
elif "Bowling" in text:
player.role = Role.P
else:
player.role = Role.B
if "Wicket-keeper" in text:
player.role = Role.W

Fetch basic stats from Cricmetric

cric_url = f"http://www.cricmetric.com/playerstats.py?player={player.name.replace(' ', '+')}"    
res2 = requests.get(cric_url, timeout=10)    
if res2.status_code == 200:    
    soup2 = BeautifulSoup(res2.text, "html.parser")    
    tables = soup2.find_all("table")    
    if tables:    
        # store first stats table    
        player.stats["summary"] = tables[0].get_text(" ")

except Exception as e:
player.stats["error"] = str(e)
return player


---

Engine


---

class AuctionEngine:
def init(self, state: AuctionState, seed: Optional[int] = None):
self.state = state
self.rng = random.Random(seed)

def process_player(self, player: Player, human_overseas: bool=False) -> SaleResult:
player = enrich_player(player)   # <--- AI auto-enrichment
player = assign_tier_and_reserve(player, self.rng)
order = list(self.state.teams.keys())
self.rng.shuffle(order)
current_bid: Optional[Bid] = None
while True:
all_pass = True
for tname in order:
team = self.state.teams[tname]
is_human = (tname == self.state.human_team)
if is_human:
decision = self.on_human_turn(player, current_bid)
if decision is None: continue
bid_amount = decision
else:
bid_amount = ai_should_bid(team, player, current_bid, first_bidder=(current_bid is None), rng=self.rng)
if bid_amount is None: continue
if current_bid and bid_amount <= current_bid.amount: continue
if not team.can_afford(bid_amount) or not team.has_slot(): continue
current_bid = Bid(team=tname, amount=bid_amount)
all_pass = False
if all_pass: break
if current_bid is None:
result = SaleResult(player, None, None, player.role or Role.B, False, "unsold")
self._record_sale(result)
return result
winner_team = current_bid.team
price = current_bid.amount
rtm_used = False
prev = player.previous_team
if prev and prev in self.state.teams:
prev_team = self.state.teams[prev]
if prev_team.rtm_available > 0 and prev_team.can_afford(price) and prev_team.has_slot():
if self.on_rtm_decision(prev, player, price):
winner_team = prev
rtm_used = True
prev_team.rtm_available -= 1
win_team_obj = self.state.teams[winner_team]
win_team_obj.add_player(player, price, is_overseas=False)
result = SaleResult(player, winner_team, price, player.role or Role.B, rtm_used, "highest_bid")
self._record_sale(result)
return result

def on_human_turn(self, player: Player, current_bid: Optional[Bid]) -> Optional[float]:
return None
def on_rtm_decision(self, prev_team: str, player: Player, price: float) -> bool:
return price <= (player.hidden_reserve or 1e9)
def _record_sale(self, sale: SaleResult):
self.state.sold_log.append(sale)
if len(self.state.sold_log) % self.state.batch_size == 0:
save_sold_batch(self.state.sold_log[-self.state.batch_size:], len(self.state.sold_log)//self.state.batch_size)


---

Persistence


---

LOG_DIR = "auction_logs"
os.makedirs(LOG_DIR, exist_ok=True)

def save_sold_batch(batch: List[SaleResult], batch_index: int):
data = []
for s in batch:
data.append({
"player": s.player.name,
"sold_to": s.sold_to or "UNSOLD",
"price": None if s.price is None else round(s.price, 2),
"role": s.role.value if s.role else "UNK",
"RTM": "Yes" if s.rtm_used else "No",
})
fname = os.path.join(LOG_DIR, f"sold_batch_{batch_index}.json")
with open(fname, "w", encoding="utf-8") as f:
json.dump(data, f, ensure_ascii=False, indent=2)

def save_state(state: AuctionState, filename: str = "state.json"):
d = {"teams": {k: {"total_slots": v.total_slots,"overseas_slots": v.overseas_slots,"purse": round(v.purse,2),
"rtm_available": v.rtm_available,"squad":[p.name for p in v.squad],"overseas_count": v.overseas_count}
for k,v in state.teams.items()},
"human_team": state.human_team}
with open(filename, "w", encoding="utf-8") as f:
json.dump(d, f, ensure_ascii=False, indent=2)


---


Player Tier Allocation Module

================================

from typing import Optional
from dataclasses import dataclass, field
from enum import Enum
import random

-------------------------------

Role & Tier Definitions

-------------------------------

class Role(str, Enum):
B = "B"  # Batter
W = "W"  # Wicket-keeper
A = "A"  # Allrounder
P = "P"  # Pace
S = "S"  # Spin

TIERS = [
(1, 0.30, 1.50),
(2, 1.50, 2.75),
(3, 3.00, 4.75),
(4, 5.00, 6.50),
(5, 7.00, 8.50),
(6, 9.00,10.50),
(7,11.00,12.50),
(8,13.00,15.50),
(9,16.00,18.00),
]

-------------------------------




