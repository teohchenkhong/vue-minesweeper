<template>
	<main>
		<button v-if = "gameState === 'Game Over'" @click = "initGame">Restart</button>
		<table class = "table-auto">
			<tr v-for = "rowIndex in nRows">
				<td :class = "'cursor-pointer w-8 h-8  border border-slate-500 ' + (checkMine(flattenIndex(rowIndex, columnIndex)) ? 'bg-sky-300' : '')" v-for = "columnIndex in nColumns" @click = "clickTile(rowIndex, columnIndex)" @contextmenu = "flagTile($event, rowIndex, columnIndex)">

					<span v-if = "gameState === 'Game Over' && tileStates[flattenIndex(rowIndex, columnIndex)] === 'Hidden' && checkMine(flattenIndex(rowIndex, columnIndex))">
						B
					</span>
					<span v-if = "tileStates[flattenIndex(rowIndex, columnIndex)] === 'Revealed'">
						{{checkMine(flattenIndex(rowIndex, columnIndex)) ? "X" : tileMines[flattenIndex(rowIndex, columnIndex)]}}
					</span>
					<span v-else-if = "tileStates[flattenIndex(rowIndex, columnIndex)] === 'Flagged'">
						X
					</span>
				</td>
			</tr>
		</table>
	</main>
</template>

<script type="text/javascript">
export default{
	props:{
		nRows: {
			type: Number
			,required: false
			,default: 12
		}
		,nColumns: {
			type: Number
			,required: false
			,default: 12
		}
		,nMines:{
			type: Number
			,required:false
			,default: 20
		}
		,mineColours:{
			type: Array
			,required: false
			,default(){
				return ["blue", "green", "yellow", "orange", "red"]
			}
		}
	}
	,data(){
		return {
			tileStates: {}
			,tileMines: {}
			,tileClusters: []
			,gameState: "Playing"
			,mineIndices: new Set()
		}
	}
	,mounted(){
		this.initGame();
	}
	,methods:{
		//==================================== INTERACTIVITY ====================================//
		clickTile(rowIndex, columnIndex){
			if (this.gameState === "Game Over") return;
			let tileIndex = this.flattenIndex(rowIndex, columnIndex);
			
			// Check tile State
			if (this.tileStates[tileIndex.toString()] !== "Hidden") return;

			// Check if it is a bomb
			if (this.checkMine(tileIndex)) return this.gameState = "Game Over";

			// Check if it is empty and reveal
			if (this.tileMines[tileIndex.toString()] === 0) return this.revealCluster(tileIndex);
			this.revealTile(tileIndex);
		}

		,revealCluster(tileIndex){
			let clusterIndex = this.find(tileIndex);
			let clusterTileIndices = this.tileClusters.map((tile,tileIndex)=> tileIndex).filter((tile, tileIndex)=>{
				return this.find(tileIndex) === clusterIndex;
			})
			clusterTileIndices.forEach(clusterTileIndex=> this.revealTile(clusterTileIndex));
		}
		,revealTile(tileIndex){
			this.tileStates[tileIndex.toString()] = "Revealed";
		}
		,flagTile(event, rowIndex, columnIndex){
			let tileIndex = this.flattenIndex(rowIndex, columnIndex).toString();
			switch(this.tileStates[tileIndex]){
				case "Hidden":{
					this.tileStates[tileIndex] = "Flagged";
				}
				break;
				case "Flagged":{
					this.tileStates[tileIndex] = "Hidden";
				}
				break;
			}
			event.preventDefault();
		}

		//===================================== GAME ENGINE =====================================//
		,initGame(){
			// Var Init
			let nTiles = this.nRows * this.nColumns;

			// Generate randomly placed mines.
			let listOfIndices = new Array(nTiles).fill(0).map((r,rIndex)=> rIndex);
			let mineIndices = new Array(this.nMines).fill(0);
			for (var index = 0; index < this.nMines; index++){
				let currentIndex = Math.floor(Math.random() * listOfIndices.length);
				listOfIndices.splice(currentIndex, 1);
				mineIndices[index] = listOfIndices[currentIndex];
			}
			this.mineIndices = new Set(mineIndices);

			// Set Game State and Tile States
			this.gameState = "playing";

			this.tileStates = Object.fromEntries(
				new Array(nTiles).fill("Hidden").map((r,rIndex)=>{
					return [rIndex, r];
				})
			);
			this.tileMines = Object.fromEntries(
				new Array(nTiles).fill(0).map((r,rIndex)=>{
					return [rIndex, r];
				})
			)

			for (var tileIndex = 0; tileIndex < nTiles; tileIndex++){
				// Get indices to check fo mine
				let surroundingIndices = this.getTileSurrounding(tileIndex);
				let mineCount = 0;
				surroundingIndices.forEach((surroundingIndex,index)=>{
					mineCount += this.checkMine(surroundingIndex);
				})
				this.tileMines[tileIndex] = mineCount;
			}

			// Tile clusters
			this.tileClusters = new Array(nTiles).fill(0).map((r,rIndex)=> rIndex);

			for (var tileIndex = 0; tileIndex < nTiles; tileIndex++){
				let mineCount = this.tileMines[tileIndex];
				
				if (mineCount === 0 && !this.checkMine(tileIndex)){
					let surroundingIndices = this.getTileSurrounding(tileIndex);
					surroundingIndices.forEach(surroundingIndex=>{
						if (!this.checkMine(surroundingIndex)){
							this.union(tileIndex, surroundingIndex);
						}
					})
				}
			}
		}

		// Helpers
		,flattenIndex(rowIndex, columnIndex){
			return this.nColumns * parseInt(rowIndex) + parseInt(columnIndex);
		}
		,checkMine(tileIndex){
			return this.mineIndices.has(tileIndex) ? 1 : 0;
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

			return surroundingIndices.map(([rowIndex, columnIndex])=> this.flattenIndex(rowIndex, columnIndex));
		}

		// Union Find Algorithm
		,find(tileIndex){
			if (this.tileClusters[tileIndex] === tileIndex) return tileIndex;

			// Path Compression
			let clusterIndex = this.find(this.tileClusters[tileIndex]);
			this.tileClusters[tileIndex] = clusterIndex;
			return clusterIndex;
		}
		,union(tileIndex, checkTileIndex){
			let irep = this.find(tileIndex);
			let checkRep = this.find(checkTileIndex);
			this.tileClusters[irep] = checkRep;
		}
	}
}
</script>