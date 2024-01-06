<template>
	<main class = "mt-2">
		<div class="mb-4 flex">
			<button class = "bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded mr-2" v-if = "gameState === 'Game Over'" @click = "initGame">Restart</button>
			<button class = "bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded mr-2" v-if = "gameState === 'New'" @click = "initGame">Start Game</button>
			<button class = "bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded mr-2" v-else-if = "gameState === 'Playing'" @click = "pauseGame">Pause</button>
			<button class = "bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded mr-2" v-else-if = "gameState === 'Paused'" @click = "resumeGame">Resume</button>
			<span class = "bg-gray-500 text-white font-bold py-2 px-4 rounded">
				{{gameState === "New" ? 0 : Math.ceil(elapsedTime/1000)}}s
			</span>
			<span class = "bg-gray-500 text-white font-bold py-2 px-4 rounded ml-2">
				{{currentScore.toLocaleString("en-US")}}
			</span>
		</div>
		<input 
			type="text" class = "opacity-0"
			ref = "inputRef"
			@keydown = "keyDownHandler"
		>
		<table class = "table-auto mt-2 mb-2 bg-gray-300" @click = "inputFocus">
			<tr v-for = "rowIndex in nRows">
				<td :class = "'cursor-pointer w-8 h-8 border-2 border-slate-800 ' + getTileColour(rowIndex-1, columnIndex-1)" v-for = "columnIndex in nColumns">
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
			,default: 20
		}
		,nColumns: {
			type: Number
			,required: false
			,default: 10
		}
	}
	,data(){
		return {
			tileStates: {}
			,currentBlock: null
			,possibleBlocks:[
				{
					label: "T-Shaped"
					,blocks: [
						[0,-1]
						,[0,0]
						,[0,1]
						,[1,0]
					]
					,maxRotate: 360
				}
				,{
					label: "Reverse L-Shaped"
					,blocks: [
						[1,-1]
						,[1,0]
						,[1,1]
						,[0,-1]
					]
					,maxRotate: 360
				}
				,{
					label: "L-Shaped"
					,blocks: [
						[1,-1]
						,[1,0]
						,[1,1]
						,[0,1]
					]
					,maxRotate: 360
				}
				,{
					label: "Straight"
					,blocks: [
						[0,-2]
						,[0,-1]
						,[0,0]
						,[0,1]
					]
					,maxRotate: 180
				}
				,{
					label: "Left Zig-Zag"
					,blocks: [
						[0,-1]
						,[0,0]
						,[1,0]
						,[1,1]
					]
					,maxRotate: 180
				}
				,{
					label: "Right Zig-Zag"
					,blocks: [
						[0,1]
						,[0,0]
						,[1,0]
						,[1,-1]
					]
					,maxRotate: 180
				}
				,{
					label: "Square"
					,blocks: [
						[0,0]
						,[0,1]
						,[1,0]
						,[1,1]
					]
					,maxRotate: 90
				}
			]
			,blockSequence: []
			,currentScore: 0

			,gameState: "New"
			,startTime: 0
			,currentTime: new Date().getTime()
			,previousTime: 0
			,timer: null
			,dropTimer: null
			,dropDelay: 1500
			,highestRow: this.nRows
			,elapsedTime: 0

			,currentLevel: 1
			,scores: [0, 100, 300, 500, 800]
		}
	}
	,mounted(){
	}
	,beforeDestroy(){
		clearInterval(this.timer);
	}
	,computed:{
		blockIndices(){
			let currentBlockIndices = this.getBlockIndices(this.currentBlock);
			return new Set(currentBlockIndices.map(([rowIndex, columnIndex])=> this.flattenIndex(rowIndex, columnIndex)));
		}
		,instantDropBlockIndices(){
			if (!this.currentBlock) return new Set();
			let deltaRow = this.getInstantDropBlockDeltaRow();
			let currentBlockIndices = this.getBlockIndices({
				...this.currentBlock
				,rowIndex: this.currentBlock.rowIndex + deltaRow
			});
			return new Set(currentBlockIndices.map(([rowIndex, columnIndex])=> this.flattenIndex(rowIndex, columnIndex)));
		}
	}
	,methods:{
		//==================================== INTERACTIVITY ====================================//
		inputFocus(){
			this.$refs.inputRef.focus();
		}
		,keyDownHandler(e){
			switch(e.key){
				case "ArrowUp":{
					this.rotateBlock(90);
				}
				break;
				case "ArrowRight":{
					this.moveBlock(0, 1);
				}
				break;
				case "ArrowDown":{
					this.moveBlock(1, 0);
				}
				break;
				case "ArrowLeft":{
					this.moveBlock(0, -1);
				}
				break;
				case " ":{
					this.instantDropBlock();
				}
				break;
				case "a":{
					this.rotateBlock(90);
				}
				break;
				case "s":{
					this.rotateBlock(-90);
				}
				break;
			}
		}
		//====================================== GRAPHICS =======================================//
		,getTileColour(rowIndex,columnIndex){
			let tileIndex = this.flattenIndex(rowIndex, columnIndex);
			if (this.tileStates[tileIndex] === 'Occupied') return "bg-sky-300";
			if (this.blockIndices.has(tileIndex)) return "bg-yellow-300";
			if (this.instantDropBlockIndices.has(tileIndex)) return "bg-green-400 opacity-80";

			return "bg-transparent";
		}
		//===================================== GAME ENGINE =====================================//
		,initGame(rowIndex, columnIndex){
			// Var Init
			let nTiles = this.nRows * this.nColumns;
			this.gameState = "Playing";
			this.tileStates = Object.fromEntries(
				new Array(nTiles).fill("Empty").map((r,rIndex)=>{
					return [rIndex, r];
				})
			);
			this.startTime = new Date().getTime();
			this.currentTime = this.startTime;
			this.currentLevel = 1;
			this.currentScore = 0;
			this.spawnBlock();
			this.resumeGame();
		}
		,endGame(){
			this.gameState = "Game Over";
			clearInterval(this.timer);
			clearInterval(this.dropTimer);
			this.currentBlock = null;
		}
		//====================== Block Manager
		,spawnBlock(blockLabel){
			let newBlock;
			if (blockLabel) newBlock = this.possibleBlocks.find(block=> block.label === blockLabel)
			else{
				// Generate new sequence
				if (this.blockSequence.length === 0){
					let currentIndex = this.possibleBlocks.length,  randomIndex;
					this.blockSequence = this.possibleBlocks.map((block, blockIndex)=> blockIndex);

					// While there remain elements to shuffle.
					while (currentIndex > 0) {
						// Pick a remaining element.
						randomIndex = Math.floor(Math.random() * currentIndex);
						currentIndex--;

						// And swap it with the current element.
						[this.blockSequence[currentIndex], this.blockSequence[randomIndex]] = [
						this.blockSequence[randomIndex], this.blockSequence[currentIndex]];
					}
				}
				// Get new block from sequence
				let nextBlockIndex = this.blockSequence.pop();
				newBlock = this.possibleBlocks[nextBlockIndex];
			} 
			if (!newBlock) return;

			let spawnRow = this.highestRow === 1 ? -1: 0;
			let spawnColumn = Math.floor(this.nColumns/2);

			this.currentBlock = {
				...newBlock
				,rowIndex: spawnRow
				,columnIndex: spawnColumn
				,orient: 0
			}
		}
		,rotateBlock(rotationAngle){
			let {orient, maxRotate, columnIndex, rowIndex} = this.currentBlock;
			let newOrient = orient + rotationAngle;
			if (newOrient >= maxRotate) newOrient -=maxRotate;
			if (newOrient <= -maxRotate) newOrient +=maxRotate;

			let newBlockIndices = this.getBlockIndices({
				...this.currentBlock
				,orient: newOrient
			});
			let blockRows = newBlockIndices.map(block => block[0]);
			let blockColumns = newBlockIndices.map(block => block[1]);
			let minRow = Math.min(...blockRows);
			let maxRow = Math.max(...blockRows);
			let minColumn = Math.min(...blockColumns);
			let maxColumn = Math.max(...blockColumns);

			let newColumnIndex = columnIndex;
			let newRowIndex = rowIndex;
			if (minColumn < 0) newColumnIndex += 1;
			if (maxColumn > this.nColumns - 1) newColumnIndex -= 1;
			if (maxRow > this.nRows - 1) newRowIndex -= 1;

			this.currentBlock.orient = newOrient;
			this.currentBlock.columnIndex = newColumnIndex;
			this.currentBlock.rowIndex = newRowIndex;
		}
		,moveBlock(moveRow, moveColumn){
			let newRowIndex = this.currentBlock.rowIndex + moveRow;
			let newColumnIndex = this.currentBlock.columnIndex + moveColumn;

			let newBlockIndices = this.getBlockIndices({
				...this.currentBlock
				,rowIndex: newRowIndex
				,columnIndex: newColumnIndex
			})
			let blockRows = newBlockIndices.map(block => block[0]);
			let blockColumns = newBlockIndices.map(block => block[1]);
			let minRow = Math.min(...blockRows);
			let maxRow = Math.max(...blockRows);
			let minColumn = Math.min(...blockColumns);
			let maxColumn = Math.max(...blockColumns);

			// Out of bound or hit an existing block
			let hasOccupiedTile = newBlockIndices.some(r=>{
				return this.tileStates[this.flattenIndex(r[0], r[1])] === "Occupied";
			})
			// if (moveRow === 1 && (hasOccupiedTile || maxRow > this.nRows-1)) return this.nextBlock();
			if (hasOccupiedTile || maxRow > this.nRows-1 || minColumn < 0 || maxColumn > this.nColumns-1) return;

			this.currentBlock.rowIndex = newRowIndex;
			this.currentBlock.columnIndex = newColumnIndex;
		}
		,nextBlock(){
			// Solidify current block.
			let currentBlockIndices = this.getBlockIndices(this.currentBlock);
			currentBlockIndices.forEach(([rowIndex, columnIndex])=> this.tileStates[this.flattenIndex(rowIndex, columnIndex)] = "Occupied");

			// Get rows cleared
			let rowsCleared = []; 
			for (var rowIndex = 0; rowIndex < this.nRows; rowIndex++){
				let tileIndices = new Array(this.nColumns).fill(1).map((r,rIndex)=> rIndex + rowIndex*this.nColumns);
				if (tileIndices.every(tileIndex=> this.tileStates[tileIndex] === "Occupied")){
					rowsCleared.push(rowIndex);
				}
			}

			// Copy over blocks excluding cleared tiles.
			if (rowsCleared.length > 0){
				let newTileStates = Object.fromEntries(
					Object.keys(this.tileStates).map((tileIndex)=> ([tileIndex, "Empty"]))
				);
				let currentRowIndex = this.nRows - 1;
				for (var rowIndex = this.nRows - 1; rowIndex >= 0; rowIndex--){
					if (!rowsCleared.includes(rowIndex)){
						for (var columnIndex = 0; columnIndex < this.nColumns; columnIndex++){
							newTileStates[this.flattenIndex(currentRowIndex,columnIndex).toString()] = this.tileStates[this.flattenIndex(rowIndex,columnIndex).toString()]
						}
						currentRowIndex--;
					}
				}
				this.tileStates = newTileStates;
			}
			this.currentScore += this.scores[rowsCleared.length] * this.currentLevel;

			// Set highest row
			this.highestRow = Math.min(...Object.entries(this.tileStates).filter(r=> r[1] !== "Empty").map(r=> Math.floor(r[0]/this.nColumns)), this.nRows);

			if (this.highestRow <= 0){
				return this.endGame();
			}
			
			// Spawn next block randomly.
			this.spawnBlock();
		}
		,instantDropBlock(){
			let deltaRow = this.getInstantDropBlockDeltaRow();
			this.moveBlock(deltaRow, 0);
			this.nextBlock();
		}
		,getInstantDropBlockDeltaRow(){
			// Get Block Indices
			let blockIndices = this.getBlockIndices(this.currentBlock);
			let blockColumns = [...new Set(blockIndices.map(r=> r[1]))];
			let rowIndices = new Array(this.nRows).fill(1).map((r,rowIndex)=> rowIndex);

			// Get Delta per column
			let deltaPerColumn = blockColumns.map(columnIndex=>{
				// Get max row of the block in the column.
				let maxBlockRow = Math.max(...[...new Set(blockIndices.filter(index=> index[1] === columnIndex).map(r=> r[0]))]);

				// Get minimum row where collision happens.
				let minCollisionRow = Math.min(...rowIndices.filter(rowIndex=>{
					return rowIndex > maxBlockRow && this.tileStates[this.flattenIndex(rowIndex, columnIndex)] !== "Empty";
				}), this.nRows);
				return minCollisionRow - maxBlockRow - 1;
			}).filter(r=> !isNaN(r))
			// Return the minimum
			return Math.min(...deltaPerColumn);
		}
		//====================== Helpers
		,getBlockIndices(block){
			if (!block) return [];
			let {blocks, rowIndex, columnIndex, orient} = block;

			// Create rotation matrix
			let angle = orient/180*Math.PI;
			let rotationMatrix = [
				[Math.cos(angle), -Math.sin(angle)]
				,[Math.sin(angle), Math.cos(angle)]
			];
			return blocks.map(([blockRowIndex, blockColumnIndex])=>{
				let rotatedRowIndex = blockRowIndex*rotationMatrix[0][0] + blockColumnIndex *rotationMatrix[0][1];
				let rotatedColumnIndex = blockRowIndex*rotationMatrix[1][0] + blockColumnIndex *rotationMatrix[1][1];

				return [rowIndex + Math.round(rotatedRowIndex), columnIndex + Math.round(rotatedColumnIndex)];
			});
		}
		,flattenIndex(rowIndex, columnIndex){
			return this.nColumns * parseInt(rowIndex) + parseInt(columnIndex);
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
		//====================== Timers
		,updateDropTimer(dropDelay){
			clearInterval(this.dropTimer)
			this.dropTimer = setInterval(()=> {
				this.dropBlock();
			}, dropDelay);
		}
		,dropBlock(){
			let nextBlockIndices = this.getBlockIndices({
				...this.currentBlock
				,rowIndex: this.currentBlock.rowIndex + 1
			});

			// Stop and spawn next block
			let stopFlag = nextBlockIndices.some(r=>{
				return r[0] >= this.nRows || this.tileStates[this.flattenIndex(r[0], r[1])] === "Occupied";
			});
			if (stopFlag) return this.nextBlock();

			this.moveBlock(1,0);
		}
		,pauseGame(){
			clearInterval(this.dropTimer);
			clearInterval(this.timer);
			this.gameState = "Paused";
		}
		,resumeGame(){
			this.gameState = "Playing";
			this.updateDropTimer(this.dropDelay);
			this.currentTime = new Date().getTime();
			this.previousTime = this.currentTime;
			this.timer = setInterval(()=>{
				this.currentTime = new Date().getTime();
				this.elapsedTime += this.currentTime - this.previousTime;
				this.previousTime = this.currentTime;
			}, 100);
			this.inputFocus();
		}
	}
}
</script>