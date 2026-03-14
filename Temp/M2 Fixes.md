
#### Essential for visuals 

Make the size of everything x1.5 
Making text readable 

Centering of unit circle icons inside hex cells 
Why did it work for spells and not for units?! 

#### Make it Readable 

Monochrome white unicode icons - from Noto Emoji 
Use single sword symbol and flip it horizontally for enemy 

Re-use fonts and styling from original html! 


Layout errors: 

There is some alpha channel and layering problem going on! Units must be drawn strictly ON TOP of the grid and none of the elements have any alpha so we never see one thru the other! 

Confirm/Revert buttons are wrong - the mockup shows then inside Log Panel (but not scroll obviously) docked at the bottom and with more rpg-like text (INDEED vs NEVER SO)

Relative to other things, Status bar is way too wide, it's supposed to be same width as grid.

Both log and info panels start off with weird collapsed size. They should take up all the space immediately - fixed size! Only their contents ever change, not their bounds! 
#### Logical errors 

Let's add a static START button that is disabled by default and becomes available only in SETUP or ENDGAME states - it starts the game anew. 
(Is this supported by current game logic? )

While we're not doing animations, we can at least do WAIT? So that everything that is supposed to be animated at least produces a slight delay and the text does not drop all at once into the Log? 

Maybe adding anims will make it clearer what's going on ? 



