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