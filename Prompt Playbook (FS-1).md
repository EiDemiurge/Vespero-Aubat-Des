
---
##### Preps
In this setup folder for a game project, let's examine the js code files for their fitness to serve as the starting point for ai coding agents. I want to avoid any redundancy, ambiguity and unnecessary yagni style stubs and comments. It needs to be compact and airtight, a solid foundation to return to if an agent fails to implement the full game. Let's start with the round pipeline - which is the main game loop. Make sure to read the full_spec first. 
##### Main Plan
Let’s create an Implementation plan for this project to be used by ai code tools, breaking up the code tasks realistically based on the specs, setup and stubs in this folder. Aim to split it into 3-4 tasks. Isolate where possible, or explicitly make it into sequential milestones, for example logic scope can start without spells included. Propose which structural changes will be helpful in the spec file structure to make this plan work. The state of project after each task completion must be verifiable in its progress towards the goal - to get the code into a state where it can reliably compute auto-battle scenarios round by round with given spells to cast (or none) - and create new baselines to use in regression testing of logic.  The end result must be easy to plug into a Phaser frontend to render 

##### Focused Task
and isolate just the information scope needed to implement Task 2 from this plan given that Task 1 was done successfully using its dedicated playbook 
Reference what is needed in previous steps, but not Milestone 3. 
 provide a custom simpler build script for task 2 to make the more complete game logic code file and test it. 

##### Impl 
(for given Task)
Follow the instructions in this file and implement the M3 Milestone. Retry up to 3 times or until the Verification Gate passes. List the most important notes of success, failure and decisions in the dev_log_m3.md and put it in newly created /dev/logs/{DD-MM-YY} folder. 

Follow the instructions in this file and implement the MX (1) Milestone. Output as single html.  List the most important notes of success, failure and decisions in the dev_log_fr_mX.md and put it in newly created /dev/logs/{DD-MM-YY} folder. 
Create a few essential tests for Verification Checklist and retry up to 3 times or until they pass based on the outcome m2 html file.


##### FR Sub-Task 1

given this extensive DOM+CSS/Phaser Canvas hybrid frontend spec, create a more focused impl plan doc for the DOM+CSS part. First let’s focus on M1: "Static HTML" - solid basis of dom, canvases and divs as required without any of the game logic or phaser complications Use mock data - static game state for log panel text and hover info table previews. Fill the rest with placeholder images ➮ grid - grid.png ➮ spells in 2 version - 3 ally_spell.png and 3 enemy_spell.png Overview of M1: HTML structure for all panels matching mockup layout CSS for the retro/pixel aesthetic, panel borders, colors Real DOM buttons wired to game.dispatch() or mocked version Spell card DOM elements with images as clickable buttons and small overlays to show their numeric hotkey as per mockup Scrollable log div with smart text label formatting Hover info panel population from unit/spell model or mock Placeholder canvas (or even simple DOM-based grid) showing unit positions Things to note about @hex-autobattler/docs/Frontend/mockup.png ➮ padding between 3 ally-spells and 3 enemy-spells ➮ text formatting in hover info panel and log - has a title heading and a well-formatted table filled with unit information or a vertical flow group of fx-info tooltip looking like markdown callouts or similar (inner tinting of rounded rectangle encompassing given text). ➮ text formatting in log panel - each entry has main info element (e.g. by logic event name - ‘Attack:” that is more like a mini heading, the rest is smaller font) Create a new compressed implementation playbook with only the data relevant for this milestone.


---
##### FR Sub-Task 2

Create a new compressed implementation playbook with only the data relevant for this milestone.

given this extensive DOM+CSS/Phaser Canvas hybrid frontend spec and the existing static part html (), create a more focused impl plan doc for M2 with Phaser - geometry objects, mouse responsiveness, hotkeys, everything except tween animations - those are for M3 and Out of Scope.

Instead, the game updating values immediately while essentially letting us play the game without fluff. 

To test, feed a sequence of logical and graphical events coming from a mock backend that uses a fixed data source for them. 

The outcome of M2 is a new html based on the M1 output, replace image placeholders with phaser objects and provision with listeners and animation hooks. 
