
### [✓ ] Tech Dictionary

##### FSM States (Logic Phases)
➮ SETUP
➮ COMBAT
➮ AFTERMATH
➮ SPELLCAST
➮ ENDGAME
##### [...] Commands 
Issued by the logic engine itself or directly from player input. 

START
	Issue initially when the game launches, or by player when they hit Restart from the End Game Screen. 
CONFIRM
	Overwrites the backup state variable with current state and proceeds with the Round Loop to COMBAT state that yields the next game state in AFTERMATH state.  
	If GameState finished flag is true, issues FINISH command instead. 
REVERT 
	Transitions into SPELLCAST state.
FINISH 
	Transitions into ENDGAME state, passing in Stats data.

##### [...] Pipeline
Everything in game logic is processed via Pipeline functions called in loops/nested recursive methods. 
These idempotent functions operate on a read-only copy of Game State and return Result objects that aggregate the following:
	A new Game State to replace previous one
	Graphic Events to animate 
	Logic Events to log and chronicle in Stats. 

A pipeline does some data-processing, then returns a Result that is processed in 2 ways:
1) Game State is applied immediately 
2) Graphic and Logic events are propagated up until they are aggregated into a batch to be animated and logged on the frontend side.

We aggregate a sequence of single-events or parallel event-groups from a Pipeline to transform into a matching sequence of Tweens animations.
While each Pipeline CAN be visualized (based on its returned Result), they do not have to be. 

FULL LIST
	Setup Pipeline
	Target Acquisition Pipeline
	Damage Pipeline
	New Round Pipeline 
		Remove temp effects
		Recalculate the Coin Stack 
	Spell Pipeline
	Endgame Pipeline

##### [✓ ] Data Models  => [Link to Content]
Party
	A 5-hexagon model with units. Each unit is mapped to a set of one or more unique hex cells without overlap.
Unit
	A flat data map of numeric and string parameters
Spell
	A super-hex mini-grid of 7 hexes with Symbols mapped to unique coordinates. 

Game State
	Round Number
	Coin Stack (100 Booleans)
	Player Party State 
	Enemy Party State 
	Remaining Spells 

Pipeline Result
	Game State
	Pipeline Type (for debugging)
	Graphics Events (nested list with parallel groups)
	Logic Events (flat list)


---


###                                       ✧✧✧✧✧ 
### [✓ ] Nested Loops 
##### Loop Nesting scheme:
Round => [Two Turn Results] => [Anim Events and Log Report]
	Party Turn => [Turn Result]
		Unit Turn => [Result Event groups for all Attacks]
			Attack n...N => [All Result Event for Targets]
				Target a...A => [All Result Event for Target]
					Damage Pipeline => [Result Event]

(➮ Depth first method stack)
##### L1 - Round Loop
The main outer loop of the combat game. When it exits, we execute the Endgame Pipeline. 
Loop steps:
1) New Round pipeline (clean-up and dynamic values refresh)
2) Determine the party turn order (party that has the fastest unit goes first)
3) Execute L2 (First Party)
4) Execute L2 (Second Party)
5) Log aggregate round result
6) Wait for player to issue CONFIRM or REVERT command. 

[Confirm] => wait for 1 second, play a generic battle sound, proceed to next Round Cycle unless we detect EXIT Condition - one of the parties has no more units left alive. 

[Revert]   => Go back to the Game State of the previous turn and unblock the Spell Panel for player. Wait for numeric input to apply a spell. 
	*This is essentially 'step 0' in Round Loop - when we enter it with a spell to apply.* 

[Spell Applied] => Enact the Apply Spell Pipeline, then proceed with Round Loop anew from step 1. 



##### L2 - Party Turn Loop
For each units that is not [DEAD] or [LOCKED], perform an action. Default one is Attack.
##### L3 - Unit Attack Loop
A for-i loop where i=Unit's "N of Attacks" parameter. 
Loop body: 
1) Try to acquire next unique target based on attacker's Attack Template (run the Target Pipeline). If none can be found, exit with PASS result event. 
2) Execute Damage Pipeline for the acquired target and return to step 1. 


###                                       ✧✧✧✧✧ 
### [✓ ] Pipelines

##### [...] Setup Pipeline

Given the Launch Data (Party 1, 2 and Spells), 
creates Game State to start the Round loop with. 

If no data is passed in, read the default data defined in code (see Content FS-1). 




##### [✓] Pipeline: New Round:
> 	Remove temp effects (Lock, Empower, Weaken)
> 	Init Coin Stack with 100 new random booleans. 
##### [✓] Pipeline: Damage 
1) Apply Damage reduction - subtract its value from incoming damage. If it amounts to zero or less, exit with BLOCKED result event.
2) Subtract remaining damage from Protection value. If the result is positive, exit with HIT result event.
3) Deal a Wound. If total Wounds do not exceed (2+target's Toughness parameter), exit with WOUNDED result event.
4) Kill the unit, applying DEAD status and exit with DEATH result event.

##### [✓] Pipeline: Apply Spell 
When activated, apply directly to Ally Zone or Enemy Zone, each symbol on its super-hex matching a cell in the target Zone. 
For each cell, if there is a unit mapped to it, apply the corresponding Symbol FX. 

##### [✓] Pipeline: Find Target 

For a given [Attack Template] and a Source unit reference, find the first target that hasn't been targeted during this Attack's processing. 
E.g. Melee Wide template may find 1, 2 or 3 targets sequentially. 
It runs in a nested while loop until it yields an empty set - then it exits.

##### [✓] Endgame Pipeline

Pass in the Stats data object to Compute and display Score
