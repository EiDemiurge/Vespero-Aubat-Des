

You are starting over with this task, the previous attempt shipped with issues. 

MAIN - you must compose the final result by running build.sh! 
This way, we use ready-made game logic src files and don't improvise with game logic which we should NOT be touching in this task. 


Only refer to the faulty () outcome when dealing with what is mentioned in ISSUES as an ANTI-EXAMPLE of what NOT to do (again). 

ISSUES (graphics): 

Spell panel composition was botched - we need to preserve 3-ally-spells | gap | 3-enemy-spells structure! 
spell components generated with squares instead of circles with icons inside - we should refer to ref_imgs/minigrid.png instead 

➮ Wounds must be displayed as blood-drop icons docked at the bottom of the unit's circle and spaced so that it can fit as many wounds as unit can have before death (2+toughness).
➮ There is no need to display all parameters - only status icons 

Hexes were placed in completely wrong positions - don't rely on anything related to this. We will not try to draw hexes dynamically, instead use the new getHexCenter() function from @hex_layout.js
Use it for spell mini-grid as well, but with smaller hexagon width parameter obviously. 

Hover was very unreliable - shapes seem to have different render coordinates from what the listener reacts to! 
 

When status says battle concluded there are 2 issues - 1) visual state is not updated 2) Confirm does not get us to the end screen still! Yet no errors in browser dev console 3) we are still able to keep pressing INDEED and get more Rounds! 


For unit icons, add ID next to Unicode symbol to identify units! 
In log panel, add a simple decorator to replace unit IDs with more readable strings like "Ally Footman [1]"


Used spells must not disappear but are overlaid with dark grey color and blocked for clicks. 

Spell FX icons (to place inside circles that form the spell mini-grid)
	Wounds: Blood Drop (red tint)
	Heal: Cross (blue tint)
	Move (both ally/enemy): Arrow (cyan tint)
	Lock: padlock (purple tint)
	(same as icon overlay for actual LOCK status) 



Design tweaks
(add this to the relevant specs )