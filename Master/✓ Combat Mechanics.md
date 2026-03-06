##### --- `Basic` ---

Units: [Grid Position]
	Fixed at the start of combat.
	Human allies can only move via Effects.
	Daimons may move instead of attacking.
	*RULE: VAN cells - cannot START on them, but can move there via FX*
Units: [Speed]
	Defines the order in which units move within their party turn.
	*RULE: Party with the fastest unit moves first* 
	➮ Turn order can be "A B | A B | B A "
	*"Killing off fast enemy units gives you a double turn!"* 

	
Units: [Attacks]
	Deterministic auto-targeted attacks with one of several Templates 
	A unit can have N attacks - this may matter if they have bonus dmg or reduction on target. 
	
Units: [Targeting Templates]
	🕂 Straight Melee - DIS rules: if none in line, then it's  'next target'
	:: Can be in the Back row as well if there are none in Front.
	🕂 Slam - always 3-cell zone, depending on front position
	🕂 Slam (back) - same, but 2 zone variants 
	🕂 Random Cell (back) -  one out of those 3 cells, randomly picked
	🕂 Polearm (back) - melee-like rule (first target from top/bottom) plus the target behind it. 

Units: [Damage]
	There are 8 damage types:
	➮ *Weapon damage - default for humans and low-tier Daimons* 
	➮ *6 Alchemic types linked to the Symbols*
	➮ *1 Extra type linked to the Master Symbol*
	These matter for piercing Wards, and may have some synergy with unit perks etc. 


Units: [Protection]
	When taking Damage, unit uses their Protection value to absorb it. 
	If there is not enough Protection to absorb all damage, unit gets a Wound (incremental counter).
	If Protection drops to zero, unit becomes Vulnerable. In this state, they can receive more than 1 Wound per attack. 
	

Units: [Wounds]
	Once a unit accumulates [2 + Toughness Bonus] Wounds, they are immediately destroyed. 
##### --- `Advanced` ---

Block
	Absorbs a single instance of Weapon Damage.
	Gained by using Defend or via effects. 
	
Ward
	Unit can have Warded state with a linked Symbol which acts as a 'key' to remove that ward. In this state, Weapon attacks deal no damage to the unit and Alchemic attacks deal only half, rounded down. An attack of matching type will deal full damage and remove the Warded state. 

Damage Reduction
	Flat reduction of incoming damage. 
	Unit has Weapon DR plus one of six Alchemic DR's (which reduces all alchemic damage except the linked one, same as Ward's type).

🕂 [Symbols]
	When affected by a Symbol, unit is subject to its Effect and also a Ritual Slot is filled. 
	When unit has more than 2 Symbols of the same type in its Ritual Slots, their damage type becomes linked to that Symbol type. 


---

