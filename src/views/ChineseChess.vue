<template>
	<main class = "mt-2">
		<div class="flex m-4">
			<button class = "bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded" v-if = "['Win', 'Draw'].includes(gameState)" @click = "initGame">Restart</button>
			<span class = "bg-gray-500 text-white font-bold py-2 px-4 rounded">
				{{gameState === "New" ? 0 : Math.ceil((currentTime - startTime)/1000)}}s
			</span>
		</div>
		<table class = "relative m-4">
			<tr v-for = "rowIndex in 9" class = "flex">
				<td colspan="9" class = "border border-slate-700 w-80 h-10 bg-amber-400" v-if = "rowIndex === 5"></td>
				<td v-else class = "w-10 h-10 border border-slate-700 bg-amber-400 relative" v-for = "columnIndex in 8">
					<template v-for = "
						chessPiece in chessPieces.filter(piece=> 
							(piece.rowIndex === rowIndex-1 || (piece.rowIndex === rowIndex && rowIndex === 4) || (piece.rowIndex === rowIndex && rowIndex === 9)) && 
							(piece.columnIndex === columnIndex-1 || (piece.columnIndex === columnIndex && columnIndex === 8))
						)
					">
						<span :class = "
							'cursor-pointer w-8 h-8 rounded-full  absolute text-center align-middle text-lg font-bold '
							+ (chessPiece.side === 2 ? 'text-red-700': 'text-green-800')
							+ (chessPiece.columnIndex === 8 ? ' -right-4': ' -left-4')
							+ (chessPiece.rowIndex === 9 || chessPiece.rowIndex === 4 ? ' -bottom-4' : ' -top-4')
							+ (
								chosenPiece && chosenPiece.ID === chessPiece.ID 
								? ' bg-emerald-500' 
								: chosenPiece && validActions.validTakeovers.has(chessPiece.ID) 
								? ' bg-red-300'
								: ' hover:bg-amber-800 bg-amber-600')
						" @click = "clickPiece(chessPiece)">{{chessPiece.label}}</span>
					</template>
					<template v-if = "chosenPiece" v-for = "
						move in validActions.validMoves.filter(move=> 
							(move.rowIndex === rowIndex-1 || (move.rowIndex === rowIndex && rowIndex === 4) || (move.rowIndex === rowIndex && rowIndex === 9)) && 
							(move.columnIndex === columnIndex-1 || (move.columnIndex === columnIndex && columnIndex === 8))
						)
					">
						<span :class = "
							'cursor-pointer w-8 h-8 rounded-full  absolute text-center align-middle text-lg hover:bg-amber-800 bg-amber-600 opacity-80 text-bold '
							+ (chosenPiece.side === 2 ? 'text-red-700': 'text-green-800')
							+ (move.columnIndex === 8 ? ' -right-4': ' -left-4')
							+ (move.rowIndex === 9 || move.rowIndex === 4 ? ' -bottom-4' : ' -top-4')
						" @click = "movePiece(move)">{{chosenPiece.label}}</span>
					</template>
				</td>
			</tr>
		</table>
	</main>
</template>

<script type="text/javascript">
export default{
	props:{
	}
	,data(){
		return {
			chessPieces: []
			,gameState: "New"
			,startTime: 0
			,currentTime: new Date().getTime()
			,timer: null
			,currentPlayer: 2
			,chosenPiece: null
		}
	}
	,mounted(){
		this.initGame();
	}
	,beforeDestroy(){
		clearInterval(this.timer);
	}
	,computed:{
		validActions(){
			let chosenPiece = this.chosenPiece;
			if (this.chosenPiece === null) return [];
			let boardIndices = new Array(10).fill([]).map((row, rowIndex)=>{
				return new Array(9).fill([]).map((col,columnIndex)=> ([rowIndex,columnIndex]));
			})

			// Var declaration
			let {label, rowIndex, columnIndex, ID, side} = chosenPiece;
			// Check for each type of chess piece.
			let validMoves = [];
			let validTakeovers = new Set();
			switch(label){
				case "車":
				case "俥":{
					let chosenPieceRow = boardIndices[rowIndex];
					let chosenPieceColumn = boardIndices.map(r=> r[columnIndex]);

					// Get pieces in the same row / column
					let piecesInSameRow = this.chessPieces.filter(r=>{
						return r.ID !== ID && r.rowIndex === rowIndex;
					}).sort((a,b)=> a.columnIndex - b.columnIndex);
					let piecesInSameColumn = this.chessPieces.filter(r=>{
						return r.ID !== ID && r.columnIndex === columnIndex;
					}).sort((a,b)=> a.rowIndex - b.rowIndex);

					// Get closest pieces left/right
					let leftPiece = null;
					let leftDisplacement = -Infinity;
					let rightPiece = null;
					let rightDisplacement = Infinity;
					piecesInSameRow.forEach(piece=>{
						let displacement = piece.columnIndex - columnIndex;
						if (displacement < 0 && displacement > leftDisplacement){
							leftPiece = piece;
							leftDisplacement = displacement;
						}
						if (displacement > 0 && displacement < rightDisplacement){
							rightPiece = piece;
							rightDisplacement = displacement;
						}
					})

					// Get closest pieces top/bot
					let topPiece = null;
					let topDisplacement = -Infinity;
					let bottomPiece = null;
					let bottomDisplacement = Infinity;
					piecesInSameColumn.forEach(piece=>{
						let displacement = piece.rowIndex - rowIndex;
						if (displacement < 0 && displacement > topDisplacement){
							topPiece = piece;
							topDisplacement = displacement;
						}
						if (displacement > 0 && displacement < bottomDisplacement){
							bottomPiece = piece;
							bottomDisplacement = displacement;
						}
					})

					let topIndex = topPiece ? topPiece.rowIndex + 1: 0;
					let bottomIndex = bottomPiece ? bottomPiece.rowIndex : 10;
					let leftIndex = leftPiece ? leftPiece.columnIndex + 1 : 0;
					let rightIndex = rightPiece ? rightPiece.columnIndex : 9;

					var loopIndex = 0;
					// Move up
					for (loopIndex = topIndex; loopIndex < rowIndex; loopIndex++){
						validMoves.push({
							rowIndex: loopIndex
							,columnIndex
						});
					}
					// Move down
					for (loopIndex = rowIndex; loopIndex < bottomIndex; loopIndex++){
						validMoves.push({
							rowIndex: loopIndex
							,columnIndex
						});
					}
					// Move left
					for (loopIndex = leftIndex; loopIndex < columnIndex; loopIndex++){
						validMoves.push({
							rowIndex
							,columnIndex: loopIndex
						});
					}
					// Move right
					for (loopIndex = columnIndex; loopIndex < rightIndex; loopIndex++){
						validMoves.push({
							rowIndex
							,columnIndex: loopIndex
						});
					}
					
					validTakeovers = new Set([topPiece,bottomPiece,leftPiece,rightPiece].filter(piece=>{
						return piece && piece.side !== side;
					}).map(r=> r.ID));
				}
				break;
				case "傌":
				case "馬":{
					let validMoveWithoutCheck = [
						[-1,2], [1,2]
						,[-1,-2], [1,-2]
						, [2,-1],[2,1]
						, [-2,-1],[-2,1]
					].map(([rowTransform, columnTransform])=>{
						return {
							rowIndex: rowIndex + rowTransform
							,columnIndex: columnIndex + columnTransform
							,rowTransform
							,columnTransform
						};
					}).filter(({rowIndex, columnIndex})=>{
						// Check within board
						return rowIndex >= 0 && rowIndex <= 9
							&& columnIndex >= 0 && columnIndex <= 10;
					});
					validMoveWithoutCheck.forEach(move=>{
						let kickMaJiao = Math.abs(move.rowTransform) === 1 ? 
							this.chessPieces.find(chessPiece=>{
							return chessPiece.rowIndex === rowIndex
								&& chessPiece.columnIndex === columnIndex + move.columnTransform/Math.abs(move.columnTransform);
						})
						: this.chessPieces.find(chessPiece=>{
							return chessPiece.rowIndex === rowIndex + move.rowTransform/Math.abs(move.rowTransform)
								&& chessPiece.columnIndex === columnIndex;
						});
						if (kickMaJiao) return;

						let otherChessPiece = this.chessPieces.find(chessPiece=>{
							return chessPiece.rowIndex === move.rowIndex 
								&& chessPiece.columnIndex === move.columnIndex;
						});
						if (!otherChessPiece) return validMoves.push(move);
						if (otherChessPiece.side !== side) validTakeovers.add(otherChessPiece.ID);
					})
				}
				break;
				case "相":
				case "象":{
					let validMoveWithoutCheck = [
						[-2,2], [2,2]
						,[-2,-2], [2,-2]
					].map(([rowTransform, columnTransform])=>{
						return {
							rowIndex: rowIndex + rowTransform
							,columnIndex: columnIndex + columnTransform
							,rowTransform
							,columnTransform
						};
					}).filter(({rowIndex, columnIndex})=>{
						// Check within board without going across the river.
						return columnIndex >= 0 && columnIndex <= 10
							&& (
								(side === 1 && rowIndex >= 0 && rowIndex <= 4)
							|| (side === 2 && rowIndex >= 5 && rowIndex <= 9)
							);
					});
					validMoveWithoutCheck.forEach(move=>{
						let kickXiangJiao = this.chessPieces.find(chessPiece=>{
							return chessPiece.rowIndex === rowIndex + move.rowTransform/Math.abs(move.rowTransform)	
								&& chessPiece.columnIndex === columnIndex + move.columnTransform/Math.abs(move.columnTransform);
						});
						if (kickXiangJiao) return;

						let otherChessPiece = this.chessPieces.find(chessPiece=>{
							return chessPiece.rowIndex === move.rowIndex 
								&& chessPiece.columnIndex === move.columnIndex;
						});
						if (!otherChessPiece) return validMoves.push(move);
						if (otherChessPiece.side !== side) validTakeovers.add(otherChessPiece.ID);
					})
				}
				break;
				case "仕":
				case "士":{
					let validMoveWithoutCheck = [
						[-1,1], [1,1]
						,[-1,-1], [1,-1]
					].map(([rowTransform, columnTransform])=>{
						return {
							rowIndex: rowIndex + rowTransform
							,columnIndex: columnIndex + columnTransform
						};
					}).filter(({rowIndex, columnIndex})=>{
						// Check within board without exiting the palace.
						return columnIndex >= 3 && columnIndex <= 5
							&& (
								(side === 1 && rowIndex >= 0 && rowIndex <= 2)
							|| (side === 2 && rowIndex >= 7 && rowIndex <= 9)
							);
					});
					validMoveWithoutCheck.forEach(move=>{
						let otherChessPiece = this.chessPieces.find(chessPiece=>{
							return chessPiece.rowIndex === move.rowIndex 
								&& chessPiece.columnIndex === move.columnIndex;
						});
						if (!otherChessPiece) return validMoves.push(move);
						if (otherChessPiece.side !== side) validTakeovers.add(otherChessPiece.ID);
					})
				}
				break;
				case "帥":
				case "將":{
					let validMoveWithoutCheck = [
						[0,1], [0,-1]
						,[-1,0], [1,0]
					].map(([rowTransform, columnTransform])=>{
						return {
							rowIndex: rowIndex + rowTransform
							,columnIndex: columnIndex + columnTransform
						};
					}).filter(({rowIndex, columnIndex})=>{
						// Check within board without exiting the palace.
						return columnIndex >= 3 && columnIndex <= 5
							&& (
								(side === 1 && rowIndex >= 0 && rowIndex <= 2)
							|| (side === 2 && rowIndex >= 7 && rowIndex <= 9)
							);
					});
					validMoveWithoutCheck.forEach(move=>{
						let otherChessPiece = this.chessPieces.find(chessPiece=>{
							return chessPiece.rowIndex === move.rowIndex 
								&& chessPiece.columnIndex === move.columnIndex;
						});
						if (!otherChessPiece) return validMoves.push(move);
						if (otherChessPiece.side !== side) validTakeovers.add(otherChessPiece.ID);
					});
					
					// Fly Emperor
					let piecesInSameColumn = this.chessPieces.filter(r=>{
						return r.ID !== ID && r.columnIndex === columnIndex;
					});
					if(piecesInSameColumn.length === 1 && ["帥", "將"].includes(piecesInSameColumn[0].label)){
						validTakeovers.add(piecesInSameColumn[0].ID)
					}
				}
				break;
				case "炮":
				case "砲":{
					let chosenPieceRow = boardIndices[rowIndex];
					let chosenPieceColumn = boardIndices.map(r=> r[columnIndex]);

					// Get pieces in the same row / column
					let piecesInSameRow = this.chessPieces.filter(r=>{
						return r.ID !== ID && r.rowIndex === rowIndex;
					}).sort((a,b)=> a.columnIndex - b.columnIndex);
					let piecesInSameColumn = this.chessPieces.filter(r=>{
						return r.ID !== ID && r.columnIndex === columnIndex;
					}).sort((a,b)=> a.rowIndex - b.rowIndex);

					// Get closest and second closest pieces left/right
					let leftPiece = null;
					let leftLeftPiece = null;
					let leftDisplacement = -Infinity;
					let rightPiece = null;
					let rightRightPiece = null;
					let rightDisplacement = Infinity;
					piecesInSameRow.forEach(piece=>{
						let displacement = piece.columnIndex - columnIndex;
						if (displacement < 0 && displacement > leftDisplacement){
							leftLeftPiece = JSON.parse(JSON.stringify(leftPiece || {}));
							leftPiece = piece;
							leftDisplacement = displacement;
						}
					})
					piecesInSameRow.sort((a,b)=> b.columnIndex - a.columnIndex).forEach(piece=>{
						let displacement = piece.columnIndex - columnIndex;
						if (displacement > 0 && displacement < rightDisplacement){
							rightRightPiece = JSON.parse(JSON.stringify(rightPiece || {}));
							rightPiece = piece;
							rightDisplacement = displacement;
						}
					})
					// Get closest and 2nd closest pieces top/bot
					let topTopPiece = null;
					let topPiece = null;
					let topDisplacement = -Infinity;
					let bottomPiece = null;
					let bottomBottomPiece = null;
					let bottomDisplacement = Infinity;
					piecesInSameColumn.forEach(piece=>{
						let displacement = piece.rowIndex - rowIndex;
						if (displacement < 0 && displacement > topDisplacement){
							topTopPiece = JSON.parse(JSON.stringify(topPiece || {}));
							topPiece = piece;
							topDisplacement = displacement;
						}
					})
					piecesInSameColumn.sort((a,b)=> b.rowIndex - a.rowIndex).forEach(piece=>{
						let displacement = piece.rowIndex - rowIndex;
						if (displacement > 0 && displacement < bottomDisplacement){
							bottomBottomPiece = JSON.parse(JSON.stringify(bottomPiece || {}));
							bottomPiece = piece;
							bottomDisplacement = displacement;
						}
					})

					let topIndex = topPiece ? topPiece.rowIndex + 1: 0;
					let bottomIndex = bottomPiece ? bottomPiece.rowIndex : 10;
					let leftIndex = leftPiece ? leftPiece.columnIndex + 1 : 0;
					let rightIndex = rightPiece ? rightPiece.columnIndex : 9;

					var loopIndex = 0;
					// Move up
					for (loopIndex = topIndex; loopIndex < rowIndex; loopIndex++){
						validMoves.push({
							rowIndex: loopIndex
							,columnIndex
						});
					}
					// Move down
					for (loopIndex = rowIndex; loopIndex < bottomIndex; loopIndex++){
						validMoves.push({
							rowIndex: loopIndex
							,columnIndex
						});
					}
					// Move left
					for (loopIndex = leftIndex; loopIndex < columnIndex; loopIndex++){
						validMoves.push({
							rowIndex
							,columnIndex: loopIndex
						});
					}
					// Move right
					for (loopIndex = columnIndex; loopIndex < rightIndex; loopIndex++){
						validMoves.push({
							rowIndex
							,columnIndex: loopIndex
						});
					}
					
					validTakeovers = new Set([topTopPiece,bottomBottomPiece,leftLeftPiece,rightRightPiece].filter(piece=>{
						return piece && piece.side !== side;
					}).map(r=> r.ID));
				}
				break;
				case "卒":
				case "兵":{
					let validMoveWithoutCheck = [
						[side === 1 ? 1 : -1,0]
					]

					if (side === 1 && rowIndex >= 5) validMoveWithoutCheck = validMoveWithoutCheck.concat([[0,-1],[0,1]])
					else if (side === 2 && rowIndex <= 4) validMoveWithoutCheck = validMoveWithoutCheck.concat([[0,-1],[0,1]])

					validMoveWithoutCheck = validMoveWithoutCheck.map(([rowTransform, columnTransform])=>{
						return {
							rowIndex: rowIndex + rowTransform
							,columnIndex: columnIndex + columnTransform
						};
					}).filter(({rowIndex, columnIndex})=>{
						// Check within board
						return rowIndex >= 0 && rowIndex <= 9
							&& columnIndex >= 0 && columnIndex <= 10;
					});
					validMoveWithoutCheck.forEach(move=>{
						let otherChessPiece = this.chessPieces.find(chessPiece=>{
							return chessPiece.rowIndex === move.rowIndex 
								&& chessPiece.columnIndex === move.columnIndex;
						});
						if (!otherChessPiece) return validMoves.push(move);
						if (otherChessPiece.side !== side) validTakeovers.add(otherChessPiece.ID);
					})
				}
				break;
			}

			// Return
			return {
				validMoves
				,validTakeovers
			}
		}

	}
	,methods:{
		//==================================== INTERACTIVITY ====================================//
		clickPiece(chessPiece){
			if (this.gameState === "Win") return;
			// Clicked opponent piece.
			if (this.currentPlayer !== chessPiece.side) {
				if (!this.chosenPiece || !this.validActions.validTakeovers.has(chessPiece.ID)) return;
				return this.takeOver(chessPiece);
			}

			// Clicked another piece.
			if (this.chosenPiece && this.chosenPiece.ID === chessPiece.ID) return this.chosenPiece = null;

			// Choose piece.
			this.chosenPiece = chessPiece;
		}
		,movePiece(move){
			this.chosenPiece.rowIndex = move.rowIndex;
			this.chosenPiece.columnIndex = move.columnIndex;

			this.chosenPiece = null;
			this.swapPlayer();
			// First Move
			if (this.gameState === "New"){
				// this.initGame(rowIndex, columnIndex);
				this.gameState = "Playing";
				this.startTime = new Date().getTime();
				this.currentTime = new Date().getTime();
				this.timer = setInterval(()=>{
					this.currentTime = new Date().getTime();
				}, 100)
			}
		}
		,takeOver(chessPiece){
			// Move
			this.movePiece(chessPiece);
			this.chessPieces = this.chessPieces.filter(piece => piece.ID !== chessPiece.ID);

			if (["帥", "將"].includes(chessPiece.label)){
				this.gameState = "Win"
				clearInterval(this.timer);
			}
		}

		//===================================== GAME ENGINE =====================================//
		,initGame(){
			this.chessPieces = [
				//============== Side 1
				// Row 0
				{
					"label": "車"
					,"ID": 1
					,"side": 1
					,"rowIndex": 0
					,"columnIndex": 0
				}
				,{
					"label": "馬"
					,"ID": 2
					,"side": 1
					,"rowIndex": 0
					,"columnIndex": 1
				}
				,{
					"label": "象"
					,"ID": 3
					,"side": 1
					,"rowIndex": 0
					,"columnIndex": 2
				}
				,{
					"label": "士"
					,"ID": 4
					,"side": 1
					,"rowIndex": 0
					,"columnIndex": 3
				}
				,{
					"label": "將"
					,"ID": 5
					,"side": 1
					,"rowIndex": 0
					,"columnIndex": 4
				}
				,{
					"label": "士"
					,"ID": 6
					,"side": 1
					,"rowIndex": 0
					,"columnIndex": 5
				}
				,{
					"label": "象"
					,"ID": 7
					,"side": 1
					,"rowIndex": 0
					,"columnIndex": 6
				}
				,{
					"label": "馬"
					,"ID": 8
					,"side": 1
					,"rowIndex": 0
					,"columnIndex": 7
				}
				,{
					"label": "車"
					,"ID": 9
					,"side": 1
					,"rowIndex": 0
					,"columnIndex": 8
				}
				// Row 2
				,{
					"label": "砲"
					,"ID": 10
					,"side": 1
					,"rowIndex": 2
					,"columnIndex": 1
				}
				,{
					"label": "砲"
					,"ID": 11
					,"side": 1
					,"rowIndex": 2
					,"columnIndex": 7
				}
				// Row 3
				,{
					"label": "卒"
					,"ID": 12
					,"side": 1
					,"rowIndex": 3
					,"columnIndex": 0
				}
				,{
					"label": "卒"
					,"ID": 13
					,"side": 1
					,"rowIndex": 3
					,"columnIndex": 2
				}
				,{
					"label": "卒"
					,"ID": 14
					,"side": 1
					,"rowIndex": 3
					,"columnIndex": 4
				}
				,{
					"label": "卒"
					,"ID": 15
					,"side": 1
					,"rowIndex": 3
					,"columnIndex": 6
				}
				,{
					"label": "卒"
					,"ID": 16
					,"side": 1
					,"rowIndex": 3
					,"columnIndex": 8
				}

				//============== Side 2
				// Row 6
				,{
					"label": "兵"
					,"ID": 17
					,"side": 2
					,"rowIndex": 6
					,"columnIndex": 0
				}
				,{
					"label": "兵"
					,"ID": 18
					,"side": 2
					,"rowIndex": 6
					,"columnIndex": 2
				}
				,{
					"label": "兵"
					,"ID": 19
					,"side": 2
					,"rowIndex": 6
					,"columnIndex": 4
				}
				,{
					"label": "兵"
					,"ID": 20
					,"side": 2
					,"rowIndex": 6
					,"columnIndex": 6
				}
				,{
					"label": "兵"
					,"ID": 21
					,"side": 2
					,"rowIndex": 6
					,"columnIndex": 8
				}
				,{
					"label": "炮"
					,"ID": 22
					,"side": 2
					,"rowIndex": 7
					,"columnIndex": 1
				}
				,{
					"label": "炮"
					,"ID": 23
					,"side": 2
					,"rowIndex": 7
					,"columnIndex": 7
				}
				,{
					"label": "俥"
					,"ID": 24
					,"side": 2
					,"rowIndex": 9
					,"columnIndex": 0
				}
				,{
					"label": "傌"
					,"ID": 25
					,"side": 2
					,"rowIndex": 9
					,"columnIndex": 1
				}
				,{
					"label": "相"
					,"ID": 26
					,"side": 2
					,"rowIndex": 9
					,"columnIndex": 2
				}
				,{
					"label": "仕"
					,"ID": 27
					,"side": 2
					,"rowIndex": 9
					,"columnIndex": 3
				}
				,{
					"label": "帥"
					,"ID": 28
					,"side": 2
					,"rowIndex": 9
					,"columnIndex": 4
				}
				,{
					"label": "仕"
					,"ID": 29
					,"side": 2
					,"rowIndex": 9
					,"columnIndex": 5
				}
				,{
					"label": "相"
					,"ID": 30
					,"side": 2
					,"rowIndex": 9
					,"columnIndex": 6
				}
				,{
					"label": "傌"
					,"ID": 31
					,"side": 2
					,"rowIndex": 9
					,"columnIndex": 7
				}
				,{
					"label": "俥"
					,"ID": 32
					,"side": 2
					,"rowIndex": 9
					,"columnIndex": 8
				}
			];
			this.gameState = "New";
			this.startTime = 0;
			this.currentTime = 0;
			this.side = 1;
		}
		// Swap player
		,swapPlayer(){
			this.currentPlayer = this.currentPlayer === 1 ? 2 : 1;
		}
	}
}
</script>