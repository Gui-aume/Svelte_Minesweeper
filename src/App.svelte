<script>
	import { fade } from 'svelte/transition'
	import { tick } from 'svelte'

	const icons = {
		closed: '',
		flag: '🚩',
		bomb: '💣'
	}

	// For score timer
	let startDate
	let timer = 0
	let setIntervalRef

	let isGameOver = false
	let win = false

	// For the creation form
	let inputHeight=10
	let inputWidth=10
	let inputMinesQuantity=10

	// For the playing grid
	let height
	let width
	let gridSize
	let minesQuantity
	let showGrid = false

	// Keep track of the cells
	let gridCells = []
	
	// Count number of undiscoverd mines
  	$: remainingMines = minesQuantity - gridCells.reduce((a,c) => a + (c === icons.flag), 0)

	// List of the mines location
	let minesLocation = []
	
	// Generate random positions for mines
	const createMines = () => {
		// Rules say : Max mines = (X-1)*(Y-1)
		minesQuantity = Math.min(inputMinesQuantity, (width-1) * (height - 1))

		const arr = new Set()
		let i=0
		// get position for all mines
		while(arr.size < minesQuantity) {
			arr.add(parseInt(Math.random()*gridSize))
		}
		minesLocation = Array.from(arr)
	}
	
	// Generate a new grid
	const createGrid = async () => {
		height = Math.min(inputHeight, 100)
		width = Math.min(inputWidth, 100)
		// number of cells
		gridSize = height * width

		showGrid = false
		createMines()

		const arr = []
		for(let i=0; i<gridSize; i++) {
			arr.push(icons.closed)
		}
		gridCells = arr
		
		// wait next tick to play new grid animation
		await tick()
		showGrid = true

		isGameOver=false
		win=false
		
		startDate = Date.now()
		
		// Start score timer
		clearInterval(setIntervalRef) // if still running
		setIntervalRef = setInterval(() => {
			timer = parseInt((Date.now() - startDate) / 1000)
		}, 1000)
	}
	
	const gameover = () => {
		clearInterval(setIntervalRef)
		
		isGameOver=true
		// show all bombs
		minesLocation.forEach(m => {
			gridCells[m] = icons.bomb
		})
	}

	// list the cells around
	const getNeighboursList = id => {
		if(id < 0 || id > gridSize) return []
		const neighboursList = []
		
		// if cell is on border, we must avoid the edges

		if(id % width !== 0){ // !left
			neighboursList.push(id-1) // add left
			if((id+1) % width !== 0){ // !right
				neighboursList.push(id+1) // add right
				if(id-width >=0) // not on top
					neighboursList.push(id-width, id-width-1, id-width+1)
				if(id+width < gridSize)
					neighboursList.push(id+width, id+width-1, id+width+1)
			} else { // right
				if(id-width >=0) // !top
					neighboursList.push(id-width, id-width-1)
				if(id+width < gridSize) // !bottom
					neighboursList.push(id+width, id+width-1)
			}
		} else { // left
			neighboursList.push(id+1) // add right
			if(id-width >=0) // !top
				neighboursList.push(id-width, id-width+1)
			if(id+width < gridSize) // !bottom
				neighboursList.push(id+width, id+width+1)
		}
		return neighboursList
	}
	
	// Count number of mines around a cell
	const countNeighboursMines = id => {
		if(id < 0 || id > gridSize) return -1
		
		let neightbors = 0
		const neighboursList = getNeighboursList(id)

		// will count the mines to show on the case
		neighboursList.forEach(n => {
			if(minesLocation.includes(n))
				neightbors++
		})
		return neightbors
	}

	// open all neighbours of the cell if there is no bomb around
	const openNeighbours = id => {
		if(id<0 || id > gridSize) return

		const neighboursList = getNeighboursList(id)
		neighboursList.forEach(n => {
			// only closed cells
			if(gridCells[n] === icons.closed){
				const neighbours = countNeighboursMines(n)
				gridCells[n] = neighbours
				// (recursive): if neighbour cell has no bomb arround we open its own neighbours
				if(neighbours === 0)
					openNeighbours(n)
			}
		})
	}

	/*
		Win condition:
		- All opened except bombes
		- All flags on bombs and other cases opened
	*/
	const checkWin = () => {
		// too much flags
		if(remainingMines < 0) return

		// count empty or flagged cells if they are not a mine
		const emptyCellsNumber = gridCells.reduce((a,c,i) => a + (!minesLocation.includes(i) && (c === icons.closed || c === icons.flag)), 0)

		// no more empty cells : WIN
		if(emptyCellsNumber === 0){
			clearInterval(setIntervalRef)

			isGameOver = true
			win = true
			// replace bombs with flags
			minesLocation.forEach(m => {
				gridCells[m] = icons.flag
			})
		}
	}

	// Right click: put or remove flag
	const rightClick = e => {
		e.preventDefault()
		const id = parseInt(e.target.id)
		// Only on closed cell
		if(gridCells[id] === icons.closed)
			gridCells[id] = icons.flag
		else if(gridCells[id] === icons.flag)
			gridCells[id] = icons.closed
		checkWin()
	}
	
	// Handle left click: open cell if still closed
	const openCell = e => {
		if(isGameOver) return
		const id = parseInt(e.target.id)
		if(gridCells[id] !== icons.closed) return

		// Found a mine: game is over
		if(minesLocation.includes(id)) {
			gameover(id)
		} else {
			// count bombs arround to set the number
			const neighbours = countNeighboursMines(id)
			gridCells[id] = neighbours
			// if no bomb arround we open the neighbours cells
			if(neighbours === 0) {
				openNeighbours(id)
			}
			
			checkWin()
		}
	}
</script>

<h1>Demineur</h1>

<div id='form'>
	<label for='height'>Hauteur : </label> <input id='height' type='number' bind:value={inputHeight} />
	<label for='width'>Largeur : </label> <input id='width' type='number' bind:value={inputWidth} />
	<label for='mines'>Mines : </label> <input id='mines' type='number' bind:value={inputMinesQuantity} />
	<button id='create' on:click={createGrid}>
		Nouvelle grille
	</button>
</div>

{#if showGrid}
<p id='time'>
	<span>Timer: {timer}s</span> 
	<span>Mines: {remainingMines}</span> 
	{#if isGameOver}<span class={win? 'win':'loose'}>{win ? 'VICTOIRE' : 'GAME OVER'}</span>{/if}
</p>

<!-- Generate the playing grid -->
<div in:fade id='grid' class:gameover="{isGameOver}" style="grid-template-columns: repeat({width}, 2em)">
	{#each gridCells as cell, i}
	<button id={i}
		on:click={openCell}
		on:contextmenu={rightClick}
		class={cell === '' ? 'closedCell' : cell !== icons.bomb ? 'openedCell' : 'bombCell'}>
			{cell ? cell : ''}
		</button>
	{/each}
</div>
{/if}