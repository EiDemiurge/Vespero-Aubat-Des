(to be split up into IMPL plan tasks?)

#### General

No external assets - vector graphics and colored unicode symbols
Icons - unicode emojis rendered as white text and tinted dynamically 

##### Phaser  
Phaser objects for Grid geometric shapes - hexes and circles 
Also register mouse motion reactions 

Animations - tweens for everything 

#### GRID

Create flat-topped hexagons with `add.polygon(6)`


Unit icon composition
Colored circle with tinted monochrome unicode symbol centered inside 

##### Color Specs
Ally background color: CYAN
Ally foreground color: WHITE
Enemy background color: MAGENTA
Enemy foreground color: BLACK

Hex colors

#### Player Controls 

##### Hover Reaction
➮ Spells - symbol fx info list composed from corresponding text pieces for each distinct FX in the spell 

Spells Blocked - use Revert to go back in time and cast a Spell before this combat round resolves! 


➮ Units - set unit info table as contents of the Hover Panel

Highlight and pop (enlarge) the circle icon


##### Click Reaction 
Unit - Same as Hover but it locks the unit's info until anything else is clicked (anywhere on the screen)
Spell - Activates if enabled, nothing otherwise
##### Hotkeys 
Space, Enter: CONFIRM command
ESC, TAB:    REVERT command 
1, 2, 3: CAST command with corresponding Ally Spell (disabled until player uses Revert for this Round and game enters CAST phase)
4, 5, 6: some for Enemy Spells (also disabled unless CAST phase)


#### UI features 

Static panel sizes - 
Build once based on current display settings and nothing ever resizes dynamically 

Dynamics
Status - cyclic alpha animation making it feel more dynamic without being too distracting 

Impactful gradient-based recoloring animations on button hover 

