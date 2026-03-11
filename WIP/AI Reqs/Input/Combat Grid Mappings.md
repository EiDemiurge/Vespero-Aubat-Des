Grid is composed of 2 super-hexes overlapping horizontally (Ally Zone and Enemy Zone respectively). Each has 5 cells exclusive to it and 2 shared cells in the middle, with total of 5+5+2=12 cells. 

Being tiny, the Grid it does not use conventional hex coordinate space. 
Instead, each cell has an ID (from 0 to 11) assigned using this algorithm for incrementing: 
Starting with the left super-hex, we pick the top hex and move clockwise until we traversed all hexes (next hex already has id assigned), then pick the center hex next. Same for the right super-hex. 

Following are static mappings of Cell IDs to facilitate geometric operations and target-acquisition: 

ID sets for Rows: 
(for Bottom to Top order, reverse for Top to Bottom)
Ally Back =  [4, 5]
Ally Front = [3, 6, 0]
Middle =      [2, 1]
Enemy Back =  [9, 8]
Enemy Front = [10, 11, 7]

Cell names to id: 
	Ally Back Bottom = [4] 
	Ally Back Top = [5] 
	Ally Front Bottom = [3] 
	Ally Front Center = [6] 
	Ally Front Top = [0] 
	Middle Bottom = [2] 
	Middle Top = [1] 
	Enemy Front Bottom = [10] 
	Enemy Front Center = [11] 
	Enemy Front Top = [7] 
	Enemy Back Bottom = [9] 
	Enemy Back Top = [8]