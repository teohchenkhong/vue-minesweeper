<template>
	<main class = "mt-2">
		<div class="mb-4 flex">
			<button class = "bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded" v-if = "gameState === 'Game Over'" @click = "initGame">Restart</button>
			<button class = "bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded" v-if = "gameState === 'Win'" @click = "initGame">Won. Restart?</button>
			<span class = "bg-gray-500 text-white font-bold py-2 px-4 rounded">
				{{gameState === "New" ? 0 : Math.ceil((currentTime - startTime)/1000)}}s
			</span>
		</div>
		<table class = "table-auto mt-2 mb-2">
			<tr v-for = "rowIndex in nRows">
				<td :class = "'cursor-pointer w-8 h-8  border border-slate-500 ' + getTileColour(rowIndex-1, columnIndex-1)" v-for = "columnIndex in nColumns" @click = "clickTile(rowIndex-1, columnIndex-1)" @contextmenu = "flagTile($event, rowIndex-1, columnIndex-1)">
					<span v-if = "gameState === 'Game Over' && tileStates[flattenIndex(rowIndex-1, columnIndex-1)] === 'Hidden' && checkMine(flattenIndex(rowIndex-1, columnIndex-1))">
						B
					</span>
					<span v-if = "tileStates[flattenIndex(rowIndex-1, columnIndex-1)] === 'Revealed'">
						{{tileMines[flattenIndex(rowIndex-1, columnIndex-1)] || ""}}
					</span>
					<span v-else-if = "tileStates[flattenIndex(rowIndex-1, columnIndex-1)] === 'Flagged'">
						X
					</span>
				</td>
			</tr>
		</table>
		<span class = "bg-gray-500 text-white font-bold py-2 px-4 rounded">{{nMines}} Mines</span>
		<span class = "bg-gray-500 text-white font-bold py-2 px-4 rounded">{{Object.values(tileStates).filter(r=> r=== 'Flagged').length}} Flagged</span>
	</main>
</template>

<script type="text/javascript">
export default{
	props:{
		nRows: {
			type: Number
			,required: false
			,default: 16
		}
		,nColumns: {
			type: Number
			,required: false
			,default: 16
		}
		,nMines:{
			type: Number
			,required:false
			,default: 40
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
			,gameState: "New"
			,mineIndices: new Set()
			,cheating: false
			,startTime: 0
			,currentTime: new Date().getTime()
			,timer: null
		}
	}
	,mounted(){
		this.initGame();
	}
	,beforeDestroy(){
		clearInterval(this.timer);
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
			if (this.tileStates[tileIndex.toString()] !== "Hidden") return;

			// Check if it is a bomb
			if (this.checkMine(tileIndex)) {
				clearInterval(this.timer);
				return this.gameState = "Game Over";
			}

			// Check if it is empty and reveal
			if (this.tileMines[tileIndex.toString()] === 0) return this.revealCluster(tileIndex);
			this.revealTile(tileIndex);
			this.checkWin();
		}

		,revealCluster(tileIndex){
			let clusterIndex = this.find(tileIndex);
			let clusterTileIndices = this.tileClusters.map((tile,tileIndex)=> tileIndex).filter((tile, tileIndex)=>{
				return this.find(tileIndex) === clusterIndex;
			})
			clusterTileIndices.forEach(clusterTileIndex=> this.revealTile(clusterTileIndex));
			this.checkWin();
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
			this.checkWin();
		}
		//====================================== GRAPHICS =======================================//
		,getTileColour(rowIndex,columnIndex){
			let tileIndex = this.flattenIndex(rowIndex, columnIndex);
			let isMine = this.checkMine(tileIndex);

			if (this.gameState === 'Game Over' && isMine) return "bg-red-400";
			if (this.tileStates[tileIndex] === 'Hidden') return "bg-gray-400";
			if (this.tileStates[tileIndex] === 'Flagged') return "bg-yellow-400";
			if (this.cheating && isMine) return "bg-sky-300";
		}
		//===================================== GAME ENGINE =====================================//
		,initGame(rowIndex, columnIndex){
			// Var Init
			let nTiles = this.nRows * this.nColumns;
			this.gameState = "New";

			// Generate randomly placed mines.
			let listOfIndices = new Array(nTiles).fill(0).map((r,rIndex)=> rIndex);
			let mineIndices = new Array(this.nMines).fill(0);
			let excludeIndex = this.flattenIndex(rowIndex, columnIndex);
			listOfIndices.splice(excludeIndex, 1);
			for (var index = 0; index < this.nMines; index++){
				let currentIndex = Math.floor(Math.random() * listOfIndices.length);
				mineIndices[index] = listOfIndices[currentIndex];
				listOfIndices.splice(currentIndex, 1);
			}
			this.mineIndices = new Set(mineIndices);

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

		// Check Win / Loss
		,checkWin(){
			// // Check if we flagged all the mines.
			// let flaggedAllMines = [...this.mineIndices].every(mineIndex=>{
			// 	return this.tileStates[mineIndex.toString()] === "Flagged";
			// });
			// Check if all tiles are revealed.
			let revealedAllTiles = this.tileClusters.every((tileCluster,tileIndex)=>{
				return this.checkMine(tileIndex) || ["Flagged", "Revealed"].includes(this.tileStates[tileIndex.toString()]);
			})

			if (revealedAllTiles) {
				this.gameState = "Win";
				clearInterval(this.timer);
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
			surroundingIndices = surroundingIndices.map(([rowIndex, columnIndex])=> this.flattenIndex(rowIndex, columnIndex));
			return surroundingIndices;
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