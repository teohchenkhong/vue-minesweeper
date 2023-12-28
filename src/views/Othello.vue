<template>
	<main class = "mt-2">
		<div class="mb-4 flex">
			{{gameState}}
			<button class = "bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded" v-if = "['Win', 'Draw'].includes(gameState)" @click = "initGame">Restart</button>
			<span class = "bg-gray-500 text-white font-bold py-2 px-4 rounded">
				{{gameState === "New" ? 0 : Math.ceil((currentTime - startTime)/1000)}}s
			</span>

			<span>Current Player</span>
			<div class="bg-green-600 w-8 h-8">
				<div :class = "'rounded-full w-full h-full ' + (currentPlayer === 'Black' ? 'bg-black' : 'bg-white')"></div>
			</div>
		</div>
		<table class = "table-auto mt-2 mb-2">
			<tr v-for = "rowIndex in nRows">
				<td :class = "'cursor-pointer w-8 h-8  border border-slate-700 ' + getTileBackground(rowIndex-1, columnIndex-1) " v-for = "columnIndex in nColumns" @click = "clickTile(rowIndex-1, columnIndex-1)">
					<div :class = "'rounded-full w-full h-full ' + getTileColour(rowIndex-1, columnIndex-1)"></div>
				</td>
			</tr>
		</table>
		<div class="bg-green-600 w-16 h-12 mb-2 py-2 flex" v-for = "(tileCount, tileColour) in tileCount">
			<div :class = "'rounded-full w-8 h-8 ' + (tileColour === 'Black' ? 'bg-black' : tileColour === 'White' ? 'bg-white' : '')"></div>
			<span class = "ml-2">{{tileCount}}</span>
		</div>
		{{currentMoves}}
	</main>
</template>

<script type="text/javascript">
export default{
	props:{
		nRows: {
			type: Number
			,required: false
			,default: 8
		}
		,nColumns: {
			type: Number
			,required: false
			,default: 8
		}
	}
	,data(){
		return {
			tileStates: {}
			,gameState: "New"
			,startTime: 0
			,currentTime: new Date().getTime()
			,timer: null
			,currentPlayer: "White"
			// ,moves: ""
			,moves: "D3E3F2C2F3F5E6G3B1F1H3H4G6F4H2H1E1D1G2E2G1C6D6C3B2C4B4G4G5C7F6E7G7F7B6B7C5D7A7A8B8B5A4B3H5E8F8G8H8H7H6D8C8A6A5A3A2A1C1D2"
			,currentMoves: ""
		}
	}
	,mounted(){
		this.initGame();
		let colIndex = {
			"A": 0
			,"B": 1
			,"C": 2
			,"D": 3
			,"E": 4
			,"F": 5
			,"G": 6
		}
		setInterval(()=>{
			if (this.moves.length > 0){
				let currentMove = this.moves.substring(0,2);
				this.moves = this.moves.substring(2,this.moves.length);

				let columnIndex = colIndex[currentMove[0]];
				let rowIndex = parseInt(currentMove[1]) - 1;
				this.clickTile(rowIndex, columnIndex)
			}
		},300);
	}
	,beforeDestroy(){
		clearInterval(this.timer);
	}
	,computed:{
		tileCount(){
			return Object.values(this.tileStates).reduce((x,it)=>{
				x[it]++;
				return x;
			}, {"Black": 0, "White": 0, "Empty": 0})
		}
	}
	,methods:{
		//==================================== INTERACTIVITY ====================================//
		clickTile(rowIndex, columnIndex){
			if (["Game Over", "Win"].includes(this.gameState)) return;

			// First Move
			if (this.gameState === "New"){
				// this.initGame(rowIndex, columnIndex);
				this.gameState = "Playing";
				this.startTime = new Date().getTime();
				this.currentTime = new Date().getTime();
				this.timer = setInterval(()=>{
					this.currentTime = new Date().getTime();
				}, 1000)
			}
			let tileIndex = this.flattenIndex(rowIndex, columnIndex);
			
			// Check tile State
			if (this.tileStates[tileIndex.toString()] !== "Empty") return;
			let validDirections = this.checkTile(rowIndex, columnIndex);

			if (validDirections.length > 0){
				validDirections.forEach(dir=>{
					dir.flipTiles.forEach(flipTileIndex=>{
						this.setTile(flipTileIndex, this.currentPlayer);
					})
				})
				this.setTile(tileIndex, this.currentPlayer);
				this.currentPlayer = this.currentPlayer === "Black" ? "White" : "Black";
				this.checkWin();
				let alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
				this.currentMoves += alphabet[columnIndex] + (rowIndex + 1);
			}
		}

		//====================================== GRAPHICS =======================================//
		,getTileColour(rowIndex,columnIndex){
			let tileIndex = this.flattenIndex(rowIndex, columnIndex);
			switch(this.tileStates[tileIndex.toString()]){
				case "Black":{
					return "bg-black";
				}
				break;
				case "White":{
					return "bg-white";
				}
				break;
				case "Empty":{
					if (this.moves.length === 0 && this.checkTile(rowIndex, columnIndex).length > 0) return "opacity-30 " + (this.currentPlayer === "Black" ? "bg-black" : "bg-white");
					return "bg-green-600"
				}
			}
		}
		,getTileBackground(rowIndex,columnIndex){
			return "bg-green-600";
			if (this.moves.length > 0) return "bg-green-600"
			if (this.checkTile(rowIndex, columnIndex).length === 0) return "bg-green-600";
			return "bg-green-300";
		}
		,setTile(tileIndex, colour){
			this.tileStates[tileIndex] = colour;
		}
		//===================================== GAME ENGINE =====================================//
		,initGame(rowIndex, columnIndex){
			// Var Init
			let nTiles = this.nRows * this.nColumns;
			this.gameState = "New";

			this.tileStates = Object.fromEntries(
				new Array(nTiles).fill("Empty").map((r,rIndex)=>{
					return [rIndex, r];
				})
			);
			let blacks = [
				this.flattenIndex(this.nRows/2 - 1, this.nColumns/2 - 1)
				,this.flattenIndex(this.nRows/2, this.nColumns/2)
			];
			let whites = [
				this.flattenIndex(this.nRows/2 - 1, this.nColumns/2)
				,this.flattenIndex(this.nRows/2, this.nColumns/2 - 1)
			];
			whites.forEach(w=> this.tileStates[w.toString()] = "White");
			blacks.forEach(w=> this.tileStates[w.toString()] = "Black");
		}

		// Check Win / Loss
		,checkWin(){
			// No others left.
			if (this.tileCount["Black"] === 0 || this.tileCount["White"] === 0){
				clearInterval(this.timer);
				return this.gameState = "Win";
			}
			// Filled
			if (this.tileCount["Empty"] === 0){
				clearInterval(this.timer);
				if (this.tileCount["Black"] === this.tileCount["White"]){
					return this.gameState = "Draw";
				}
				return this.gameState = "Win";
			}
			let filledTiles = Object.entries(this.tileStates).filter(r=>{
				return r[1] !== "Empty";
			}).map(r=> parseInt(r[0]));
			let surroundingIndices = filledTiles.reduce((x,it)=>{
				let surroundingTiles = this.getTileSurrounding(it);
				surroundingTiles.forEach(surrTileIndex =>{
					if (this.tileStates[surrTileIndex.toString()] === "Empty") x.add(surrTileIndex);
				})
				return x;
			}, new Set());
			let validMoves = [...surroundingIndices].filter(surrTileIndex=>{
				let {rowIndex, columnIndex} = this.expandIndex(surrTileIndex);
				return this.checkTile(rowIndex, columnIndex).length > 0;
			})
			if (validMoves.length === 0){
				clearInterval(this.timer);
				if (this.tileCount["Black"] === this.tileCount["White"]){
					return this.gameState = "Draw";
				}
				return this.gameState = "Win";

			}
		}
		,checkTile(rowIndex, columnIndex){

			// Check whose turn it is.
			let currentPlayer = this.currentPlayer;
			let opponent = currentPlayer === "Black" ? "White" : "Black";

			// Get valid tiles.
			let tileIndex = this.flattenIndex(rowIndex, columnIndex);
			let surroundingIndices = this.getTileSurrounding(tileIndex).filter(surroundingIndex=>{
				return this.tileStates[surroundingIndex.toString()] === opponent;				
			});
			
			// Not a valid tile.
			if (surroundingIndices.length === 0) return [];

			// Get directions that are valid surrounding tiles.
			let directions = surroundingIndices.map(surroundingIndex=>{
				let {rowIndex: surroundingRowIndex, columnIndex: surroundingColumnIndex} = this.expandIndex(surroundingIndex);
				return {
					transformRow: surroundingRowIndex - rowIndex
					,transformColumn: surroundingColumnIndex - columnIndex
					,startRow: surroundingRowIndex
					,startColumn: surroundingColumnIndex
					,flipTiles: [surroundingIndex]
					,validity: "Unknown"
				}
			})

			// Check the directions if they are valid.
			let iteration = 1;
			while (directions.filter(dir=>dir.validity === "Unknown").length > 0){
				directions.forEach(dir=>{
					if (dir.validity !== "Unknown") return;
					let checkRow = dir.startRow + iteration*dir.transformRow;
					let checkColumn = dir.startColumn + iteration*dir.transformColumn;

					if (checkRow < 0 || checkColumn < 0 || checkRow > this.nRows - 1 || checkColumn > this.nColumns - 1){
						dir.validity = "Invalid";
						return;
					}

					let checkTileIndex = this.flattenIndex(checkRow, checkColumn);

					let checkTileState = this.tileStates[checkTileIndex.toString()];
					// console.log(dir.transformRow, dir.transformColumn, checkRow, checkColumn, checkTileState)
					if (checkTileState === currentPlayer){
						dir.validity = "Valid";
						// console.log("1")
					}
					else if (checkTileState === opponent){
						dir.flipTiles.push(checkTileIndex);
						// console.log("2")
					}
					else{
						dir.validity = "Invalid";
						// console.log("3")
					}
				})
				iteration++;
			}
			return directions.filter(dir=> dir.validity === "Valid");
		}

		// Helpers
		,flattenIndex(rowIndex, columnIndex){
			return this.nColumns * parseInt(rowIndex) + parseInt(columnIndex);
		}
		,expandIndex(tileIndex){
			return {
				rowIndex: Math.floor(tileIndex/this.nColumns)
				,columnIndex: tileIndex%this.nColumns
			}
		}
		,getTileSurrounding(tileIndex){
			let rowIndex = Math.floor(tileIndex/this.nColumns);
			let columnIndex = tileIndex%this.nColumns;
			let surroundingIndices = [];
			let hasBottom = rowIndex < this.nRows - 1;
			let hasTop = rowIndex > 0;
			let hasLeft = columnIndex > 0;
			let hasRight = columnIndex < this.nColumns - 1;
			//  Bottom
			if (hasBottom){
				// Same column
				surroundingIndices.push([rowIndex + 1, columnIndex]);

				// Right
				if (hasRight){
					surroundingIndices.push([rowIndex + 1, columnIndex + 1]);
				}
				// Left
				if (hasLeft){
					surroundingIndices.push([rowIndex + 1, columnIndex - 1]);
				}
			}
			// Same Row
				// Right
				if (hasRight){
					surroundingIndices.push([rowIndex, columnIndex + 1]);
				}
				// Left
				if (hasLeft){
					surroundingIndices.push([rowIndex, columnIndex - 1]);
				}
			// Top
			if (hasTop){
				// Same column
				surroundingIndices.push([rowIndex - 1, columnIndex]);

				// Right
				if (hasRight){
					surroundingIndices.push([rowIndex - 1, columnIndex + 1]);
				}
				// Left
				if (hasLeft){
					surroundingIndices.push([rowIndex - 1, columnIndex - 1]);
				}
			}
			surroundingIndices = surroundingIndices.map(([rowIndex, columnIndex])=> this.flattenIndex(rowIndex, columnIndex));
			return surroundingIndices;
		}
	}
}
</script>