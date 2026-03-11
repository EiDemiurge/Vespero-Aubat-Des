
Dead in 3 turns, BUT can apply a spell on the last turn e.g. ... 
Apply one spell? 

"wait for player input" - ? 

No this must be an auto-test that AI can rely on - if its impl is right, it must PASS. 

Maybe I can instead ask it to try and simulate combat results - then VERIFY them? 

I doubt that it will succeed right now. 




### Default Preset:
#### Ally Party:
[0] = FOOTMAN
[3] = FOOTMAN
[4] = ARCHER

#### Enemy Party:
[7] = FOOTMAN
[10] = FOOTMAN
[11] = FOOTMAN
[8] = ARCHER
[9] = ARCHER

#### Spells:
Spell_1:
	[0] = HP
	[2] = HP
	[4] = HP

Spell_2:
	[0] = MOVE
	[2] = MOVE
	[4] = MOVE

Spell_3:
	[0] = LOCK
	[2] = LOCK
	[4] = LOCK

Spell_3:
	[0] = DMG
	[2] = DMG
	[4] = DMG