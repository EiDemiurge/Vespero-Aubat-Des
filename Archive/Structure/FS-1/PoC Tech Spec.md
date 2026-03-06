Let AI format this when it is complete.
The next step will be to create an IMPL plan. 

### Game Logic 


#### Board Setup

#### Nested Loops

L1 - Round Loop

L2 - Party Turn Loop

L3 - Unit Attack Loop
A for-i loop where i=Unit's "N of Attacks" parameter. 
Loop body: 
Acquire the target based on attacker's Attack Template.


#### Game State 
Stack 
Reversible 



#### Rule Processing
#### Damage Pipeline
1) Apply Damage reduction - subtract its value from incoming damage. If it amounts to zero or less, exit with BLOCKED result event.
2) Subtract remaining damage from Protection value. If the result is positive, exit with HIT result event.
3) Deal a Wound. If total Wounds do not exceed (2+target's Toughness parameter), exit with WOUNDED result event.
4) Kill the unit, applying DEAD status and exit with DEATH result event.

#### Applying a Spell 
When activated, apply directly to Ally Zone or Enemy Zone, each symbol on its super-hex matching a cell in the target Zone. 
For each cell, if there is a unit mapped to it, apply the corresponding Symbol FX. 


### Logic Content Dictionary 


Spell FX 

Perks 

Attack Templates 

### Data Models
Party
	A 5-hexagon model with units. Each unit is mapped to a set of one or more unique hex cells without overlap.
Unit
	A flat data map of numeric and string parameters
Spell
	A super-hex mini-grid of 7 hexes with Symbols mapped to unique coordinates. 

### Graphics
Tweens
Use smart transforms and coloring to animate the grid. 

Attack
Hit
	
Wound
	
Death
	Flash color-animation with Invert effect + Yank abruptly downwards, then return.
	0.5s Delay
	Fade out sine + Fade in Skull Icon on this cell 

Hover Responsiveness




### UI Layout

#### General
Scale  up or down by weights to fit horizontally - Grid Panel (0.7), Log Panel (0.1), Hover Text Panel (0.2). 
Minimum sizes - 
Max sizes - use outer filler to pad the excess space 

#### Positioning
Log Panel
Hover Text Panel
Status
	Width capped by Grid Panel size
Grid Panel
Spell Panel

#### UI Elements Details

Log Panel
	
Hover Text Panel
	
Status
	
Grid Panel
	
Spell Panel
	
Spell Button
Grid Cell
Unit Icon

### Player Controls 
Hover Reaction
Click Reaction 
Hotkeys 

### Text Content

