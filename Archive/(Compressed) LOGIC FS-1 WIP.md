
### Header - Dictionary of Terms 
States
Commands - [C: X]
Pipelines  - [P: X]
Result Events - [E: X]
(internal vs external? logic vs graphic?)

### Game workflow skeleton
FSM with strict state transitions 
SETUP -> COMBAT -> AFTERMATH -> 
(1) SPELLCAST -> COMBAT ([C: Revert])
(2) COMBAT  ([C: Confirm])
(3) ENDGAME -> SETUP ([C: Confirm] + End Game condition == true )

Round Loop NESTS Party Turn Loop NESTS Unit Actions Loop 


### Functional Elements 
#### Commands


#### Pipelines 

[P: Spell]
[P: Damage]
[P: Target]

#### Result Events


### Features
Basic State Revert
	Single variable for State Backup - we only need to remember one Game State from the end of the previous Round. 
Logging aggregate

Event sequencing

Coin Stack 
	Random is based on Pre-generated stack of booleans


### Content Elements 
Party Preset
Unit
Spell 

