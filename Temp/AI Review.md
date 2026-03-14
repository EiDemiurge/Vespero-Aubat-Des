Baseline Scenario generation 
I could also just try to 'give it a shot' and the verify the outcome - if it could be formatted clearly. 

Analyzing the SETUP folder 


IMPL Plan Review 

Analyze and find the right place to add these rule clarifications concisely and unambiguously: 
- [ ] Add a rule for Move-related specs - we specifically forbid units from back row to move to middle row directly
- [ ] When two units from different sides are both in the Middle Row - Melee targeting will always yield the other unit in middle row as the first target.

- [ ] Spell/Symbols - UNIQUE names are needed
- [ ] For spells on enemy zone, 4=1, 5=2 


Next:

Convert unit data model into a table same as others 

















- [ ] Coin Stack
- [ ] Ambiguous - DMG source
- [ ] Ambiguous - spell applied to 5/7 party 
- [ ] Ambiguous - AFTERMATH (waiting for player)
- [ ] Ambiguous - Speed (fastest unit)
- [ ] Backup State - static field 
- [ ] Weak/Power - in math details section
- [ ] Finished flag checked whenever we transit into AFTERMATH?  (what if we CAST, but still LOSE? YES - after a CAST, do we SKIP confirm?! Or just block Revert?)

Formatted spec - tables? 

Overall, the model of course did NOT have enough CTX - missing refs and assumed content/data all over the place. 

