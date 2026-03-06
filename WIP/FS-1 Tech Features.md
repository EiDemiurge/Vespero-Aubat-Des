#### TRICKY: 
❖ Revert turn to apply a Lapis - just a basic state stack 
With 3 lapides, we don't intervene always...
(why wouldn't we wanna do it asap?)

❖ Multi-Cell Units 

❖ 

#### Logic Phase Pipeline (FSM Transitions)
Q: N of Attacks - complicates loop? 

Nested loops 
Whatever happens, we always get to the PLAYER CONFIRM state. 
##### L1: Round Cycle
➮ Determine the party turn order - which party has the fastest unit?
➮ 1st Party Turn 
*Units make their moves from fastest to slowest, draws are broken up randomly but based on a seed that is part of Reversible Game State.* 
➮ 2nd Party Turn 
(same)
➮ Log all results
➮ Listen for Player input: Confirm or Revert 
If player has NO SPELLS, Revert option is disabled. 

REVERT => go back to the Game State of the previous turn and unblock the Spell Panel for player. Wait for numeric input to apply a spell. 
Spell Applied => process its effects, then proceed with Round Cycle.

CONFIRM => wait for 1 second, play a sound (?), proceed to next Round Cycle unless we detect EXIT Condition - one of the parties has no more units left alive. 


Status reflects the state of the combat logic's Round-FSM.
Log is not just events, but also an aggregate report. 
##### L2: Attack Processing
Order of execution
Targeting Templates - acquire Target(s)
Q: Can attack go into 'void'? 
Number of attacks - just an outer loop ? 
	This does complicate things a bit. 
	What if some interrupting event happens after the FIRST of those attacks? We could still do it 'in-place' 
	

DR 

##### Damage Calculation
Reduce PROT
If negative, ceil to 0 and create Wound event


##### Wound Event
i: Maybe some 'wards' could save from W specifically! 
➮ Wounded Resolution
➮ If [X] - Death Resolution

Attach Tweens to these event-resolutions 

##### Controls
Confirm - space of simple large NEXT button 
Log - as chronicle, gamified - and that's where we also say "THIS WAS NEVER THUS"! This btw is the HEADLINE of the game, its PREMISE - the RUIN happens to Anphis, but he does NOT accept it ... 
(*in the future, we can have some rudimentary LLM generate RPG texts there?*)
➮ DENY - ESC or button 

Applying Spells 
Start -> Turn -> Confirm -> (revert, spell, turn again) -> Confirm
When you use revert, you HAVE to use a spell? 

LEFT | RIGHT - just use 2 X 3 = 6 hotkeys? 

#### Graphics 
Log Panel (!) 
Player and me - will wanna see EXACTLY what happened 

Status bar 



##### Info Tooltips 
Unit parameters 
Symbol effects 
A wall of text on the side of the screen? 



Hover-tooltip adjusts 
➮ Lapis
➮ Unit


##### Unit Overlays 
Wounds

##### Grid 
Unit cards

Cells 
Symbol overlays 

Corpse 