Let AI format this when it is complete.
The next step will be to create an IMPL plan. 

### Tech Dictionary
Commands 
Issue by the logic engine itself or directly from player input. 

START
CONFIRM
	Overwrites the backup state variable with current state. 
REVERT 
FINISH 


Pipeline
Core loop will call these one-shot functions. 
Stateless - they return a ??? 

A pipeline does some data-processing, then returns Result Events to be animated on the frontend side.

Setup Pipeline
Damage Pipeline
Spell Pipeline
Endgame Pipeline

Result Events
Aggregate a sequence of single-events or parallel event-groups from a Pipeline to transform into a matching sequence of Tweens and animate

State Backup 
We only need to remember one Game State from the end of the previous Round. 

Impl notes - what features to keep in mind
Logging 

### [Compress old text] Game Logic 

#### [✓] Setup Pipeline

Define the data presets before launch:

CSS intro-animation: elements drift into position from outside the screen bounds 

#### [➮] Nested Loops

##### L1 - Round Loop
The main outer loop of the combat game. When it exits, we execute the Endgame Pipeline. 
Loop steps:
1) Determine the party turn order (party that has the fastest unit goes first)
2) Execute L2 (First Party)
3) Execute L2 (Second Party)
4) Log aggregate round result
5) Wait for player to issue CONFIRM or REVERT command. 

[Confirm] => wait for 1 second, play a generic battle sound, proceed to next Round Cycle unless we detect EXIT Condition - one of the parties has no more units left alive. 

[Revert]   => Go back to the Game State of the previous turn and unblock the Spell Panel for player. Wait for numeric input to apply a spell. 
	*This is essentially 'step 0' in Round Loop - when we enter it with a spell to apply.* 

[Spell Applied] => Enact the Apply Spell Pipeline, then proceed with Round Loop anew from step 1. 



##### L2 - Party Turn Loop
For each units that is not DEAD or LOCKED, perform an action.
Default one is Attack.


##### L3 - Unit Attack Loop
A for-i loop where i=Unit's "N of Attacks" parameter. 
Loop body: 
1) Try to acquire next unique target based on attacker's Attack Template. If none can be found, exit with PASS result event. 
2) Execute Damage Pipeline for the acquired target and return to step 1. 


##### Extras 
Clean-up:
	Remove temp effects (Lock, Empower, Weaken)
#### [...] Game State 
Stack 
Reversible 

#### Rule Processing
##### [✓] Damage Pipeline
1) Apply Damage reduction - subtract its value from incoming damage. If it amounts to zero or less, exit with BLOCKED result event.
2) Subtract remaining damage from Protection value. If the result is positive, exit with HIT result event.
3) Deal a Wound. If total Wounds do not exceed (2+target's Toughness parameter), exit with WOUNDED result event.
4) Kill the unit, applying DEAD status and exit with DEATH result event.

##### [✓] Applying a Spell 
When activated, apply directly to Ally Zone or Enemy Zone, each symbol on its super-hex matching a cell in the target Zone. 
For each cell, if there is a unit mapped to it, apply the corresponding Symbol FX. 

##### [...] Acquiring a Melee Target
(PIPELINE?)



### [Fill Outline] Logic Content Dictionary 


##### [✓] Spell FX 

WOHE: Wound/Heal
	Immediate add or remove 1 Wound for target enemy/ally.
VELO: Lock/Lock
	Grants Damage Reduction of 10. 
	Locked units cannot act. 
	Removed at the end of the Round.
EMWE: Empower/Weaken
	Empower ally - they deal 2x damage until end of Round.
	Weaken enemy - they deal 1/2x damage (rounded down) until end of Round.
MOSH: Move/Push
	Target must move to an adjacent cell in the same Row, preferring free ones. If it's occupied, they will swap positions with the unit there. 

##### [...] Perks 


##### [...] Attack Templates 
Melee Single
Melee Wide




### [✓ Link to Content] Data Models
Party
	A 5-hexagon model with units. Each unit is mapped to a set of one or more unique hex cells without overlap.
Unit
	A flat data map of numeric and string parameters
Spell
	A super-hex mini-grid of 7 hexes with Symbols mapped to unique coordinates. 

### [Refine] Graphics
Tweens - use smart transforms and coloring to animate the grid. 

*When a pipeline exits (e.g. Attack), it yields a sequence of events that we must animate - some sequentially, some in parallel.* 
Example: 
	Attack => Parallel [Hit A, Death B, Wound C]

##### Tween Animations - ATTACK pipeline
Attack (on source)
	Sway forward and back (halfway towards the (first) target within 0.2s). 
Blocked (on target)
	Bounce elastically and flash with white slightly 
Hit
	
Wound
	Flash red+ Yank Diagonally 
	Flash in (with white overexposure) the Wound icon (as defined in content specs).
	Wound icon is attached to unit icon's overlays.
Death
	Flash color-animation with Invert effect + Yank abruptly downwards, then return.
	0.5s Delay
	Fade out sine + Fade in Skull Icon on this cell 

##### Tween Animations - SPELL pipeline

Heal
Move
Lock


Hover Responsiveness


### [Fill Outline!] UI Layout

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

### [Refine Details] Player Controls 
Hover Reaction
Click Reaction 
##### Hotkeys 
Space: CONFIRM command (Confirm round outcomes to proceed)
ESC:    REVERT command (Revert round outcomes and activate spell-casting phase.)
1-6: mapped to Spells (disabled until player uses Revert for this Round)

### [???]  Text Content

Statuses

Help Info? 