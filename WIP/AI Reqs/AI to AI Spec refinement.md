Look at the source spec and
1) Find typos and mistakes 
2) Find logical errors 
3) Assess viability for using it as a one-shot claude code prompt to make this game in Phaser. 


Based on the information in the source, I want you to create different data formats 
Let's start with formatting it nicely to make it uniform and readable. Identify key types of information in this structure and align the formatting on those. 

output as .md 

Human readable formatted
Human readable compressed 
Distributed AI Code Tool Spec 
maximum compression in the core navigation document, all details are pushed into separate .md files that are referenced. 


Into technical terms without losing readability 



Let's discuss what is needed in terms of tech-specs for claude code to successfully one-shot a relatively small but non-trivial game demo.
It's a logic-heavy turn-based game with a lot of UI. 
The tech stack is Phaser - the outcome is a single html that should be runnable in isolation on any system to just quickly test the game.


GOOD, so we can use any number of assets? But not sprite sheets? 

Phaser JS framework research: 
Content usage - how easy is it to use locally stored content, say, an asset folder adjacent to the html that is used to run the game? 

--------


Maybe some parts of this could be brainstormed w/ AI in isolation? 

UI Layout  - for sure 

Hex Lib Reqs
Attack Templates, Spell Configs 
 - we don’t really need a coordinate system, just some basic mappings! 

0-5 + central is 6
12 grid cells - but also represented as two distinct Super-Hexes 

When applying: 
0-6 maps either directly to 0-6 for grid, or diff numbers for enemy zone
clockwise starting with the top hex cell and ending with the center one.
The shared cells between will have indices of 1 and 2
Enemy Zone then is: [7, 8, 9, 10, 2, 1, 11] 

Attack Templates

