<template>
  <svg :width="width" :height="height">
    <g v-for="(cell, index) in grid" :key="index">
      <rect 
        :x="cell.col * cellSize" 
        :y="cell.row * cellSize" 
        :width="cellSize" 
        :height="cellSize" fill='black'
        stroke='blue' 
        stroke-width='1' />
      <text v-if="cell.collapsed == true && cell.option != null" 
        :x="cell.col * cellSize + (cellSize - 60)/2"
        :y="cell.row * cellSize + (cellSize + 60)/2" 
        stroke='yellow' 
        stroke-width='1'
        font-size='100px'>
        {{ cell.option }}
      </text>
      <text v-if="cell.collapsed == false" 
        :x="cell.col * cellSize + 20"
        :y="cell.row * cellSize + 30" 
        stroke='green' 
        stroke-width='1'
        font-size='20px'>
        {{ cell.displayOptions.join(',') }}
      </text>
    </g>
    <line :x1="cellSize * 3" :y1="0" :x2="cellSize * 3" :y2="cellSize * 9" style="stroke:rgb(255,0,0);stroke-width:3" />
    <line :x1="cellSize * 6" :y1="0" :x2="cellSize * 6" :y2="cellSize * 9" style="stroke:rgb(255,0,0);stroke-width:3" />
    <line :x1="0" :y1="cellSize * 3" :x2="cellSize * 9" :y2="cellSize * 3" style="stroke:rgb(255,0,0);stroke-width:3" />
    <line :x1="0" :y1="cellSize * 6" :x2="cellSize * 9" :y2="cellSize * 6" style="stroke:rgb(255,0,0);stroke-width:3" />
    <line :x1="0" :y1="0" :x2="0" :y2="cellSize * 9" style="stroke:rgb(225, 255, 0);stroke-width:3" />
    <line :x1="0" :y1="0" :x2="cellSize * 9" :y2="0" style="stroke:rgb(225, 255, 0);stroke-width:3" />
    <line :x1="0" :y1="cellSize * 9" :x2="cellSize * 9" :y2="cellSize * 9"
      style="stroke:rgb(225, 255, 0);stroke-width:3" />
    <line :x1="cellSize * 9" :y1="0" :x2="cellSize * 9" :y2="cellSize * 9"
      style="stroke:rgb(225, 255, 0);stroke-width:3" />
  </svg>
  <div>
    <button @click="randomSet">RandomSet</button>
    <button @click="resetGrid">ResetGrid</button>
    <button @click="calcResultStep">CalcResultStep</button>
    <button @click="calcResult">CalcResult</button>
    <button @click="calcStop">CalcStop</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      grid: [],
      snapShots: [],
      message: "",
      timer: null,
      currentCell: null,
      width: 2000,
      height: 2000,
      cellSize: 200,
      rows: 9,
      cols: 9,
      options: [1, 2, 3, 4, 5, 6, 7, 8, 9],
      resetCount: 20,
    }
  },

  created() {
    this.initializeGrid();
  },

  methods: {
    initializeGrid() {
      for (let i = 0; i < this.rows; i++) {
        for (let j = 0; j < this.cols; j++) {
          this.grid.push({
            row: i,
            col: j,
            option: null,
            options: [1, 2, 3, 4, 5, 6, 7, 8, 9],
            displayOptions: [...this.options], 
            collapsed: false,
          });
        }
      }
    },

    getCell(row, col) {
      if (row >= 0 && row < this.rows && col >= 0 && col < this.cols) {
        return this.grid[this.cols * row + col];
      }
      else {
        return null;
      }
    },

    resetGrid() {
      this.grid.forEach(x => {
        x.options = [1, 2, 3, 4, 5, 6, 7, 8, 9];
        x.displayOptions = [1, 2, 3, 4, 5, 6, 7, 8, 9];
        x.option = null;
        x.collapsed = false;
      });
    },

    randomSet() {

      this.resetGrid();
      for (let i = 0; i < this.resetCount; i++) {
        let randomIndex = Math.floor(Math.random() * this.grid.length);
        let cell = this.grid[randomIndex];
        let row = cell.row;
        let col = cell.col;
        let rowNeighbors = this.grid.filter(item => item.row === row && item !== cell);
        let colNeighbors = this.grid.filter(item => item.col === col && item !== cell);
        let boxNeighbors = this.getBoxNeighbors(row, col).filter(item => item !== cell);
        let neighbors = rowNeighbors.concat(colNeighbors).concat(boxNeighbors);
        let neighborOptions = neighbors.map(item => item.option);
        let availableOptions = this.options.filter(option => !neighborOptions.includes(option));
        if (availableOptions.length > 0) {
          let randomOption = availableOptions[Math.floor(Math.random() * availableOptions.length)];
          cell.option = randomOption;
          cell.collapsed = true;

          // ????
          neighbors.forEach(neighbor => {
            if (!neighbor.collapsed) {
              neighbor.displayOptions = neighbor.displayOptions.filter(option => option !== randomOption);
            }
          });

        } else {
          i--;
        }
      }
    },

    getBoxNeighbors(row, col) {
      let boxStartRow = Math.floor(row / 3) * 3;
      let boxStartCol = Math.floor(col / 3) * 3;
      let neighbors = [];
      for (let i = boxStartRow; i < boxStartRow + 3; i++) {
        for (let j = boxStartCol; j < boxStartCol + 3; j++) {
          neighbors.push(this.grid[i * this.cols + j]);
        }
      }
      return neighbors;
    },

    calcResultStep() {
      if (this.grid.every(cell => cell.collapsed)) {

        return false;
      }
      const snapshot = this.grid.map(x => ({ cell: x, options: x.options.slice(), collapsed: x.collapsed }));
      this.currentCell = this.tryCollapse();
      if (this.currentCell) {
        this.snapShots.push({ before: snapshot, cell: this.currentCell, option: this.currentCell.option });
      } else {
        if (this.snapShots.length == 0) {
          this.message = "no solution";
          return false;
        }
        const lastSnapshot = this.snapShots.pop();
        lastSnapshot.before.forEach(x => {
          x.cell.options = x.options.slice();
          x.cell.collapsed = x.collapsed;
        });
        const optionIndex = lastSnapshot.cell.options.indexOf(lastSnapshot.option);
        if (optionIndex >= 0) {
          lastSnapshot.cell.options.splice(optionIndex, 1);
        }
        lastSnapshot.cell.collapsed = false;
      }
      return true;
    },

    calcResult() {
      this.timer = setInterval(() => {
        if (this.calcResultStep() == false) {
          clearInterval(this.timer);
          this.timer = null;
        }
      }, 100);
    },

    calcStop() {
      if (this.timer) {
        clearInterval(this.timer);
        this.timer = null;
      }
    },

    tryCollapse() {
      const current = this.getMinEntropyCell();
      if (current == null) {
        return null;
      }
      let row = current.row;
      let col = current.col;
      let rowNeighbors = this.grid.filter(item => item.row === row && item !== current && item.collapsed);
      let colNeighbors = this.grid.filter(item => item.col === col && item !== current && item.collapsed);
      let boxNeighbors = this.getBoxNeighbors(row, col).filter(item => item !== current && item.collapsed);

      let disRowNeighbors = this.grid.filter(item => item.row === row && item !== current );
      let disColNeighbors = this.grid.filter(item => item.col === col && item !== current );
      let disBoxNeighbors = this.getBoxNeighbors(row, col).filter(item => item !== current );
      let disNeighbors = disRowNeighbors.concat(disColNeighbors).concat(disBoxNeighbors);
      let neighbors = rowNeighbors.concat(colNeighbors).concat(boxNeighbors);
      let neighborOptions = neighbors.map(item => item.option);
      let availableOptions = current.options.filter(option => !neighborOptions.includes(option));
      if (availableOptions.length == 0) {
        return null;
      }
      let randomOption = availableOptions[Math.floor(Math.random() * availableOptions.length)];
      current.option = randomOption;
      current.collapsed = true;

      //?????
      disNeighbors.forEach(neighbor => {
        if(!neighbor.collapsed){
          neighbor.displayOptions = neighbor.displayOptions.filter(option => option !== randomOption);
        }
      });
  

      return current;
    },

    getMinEntropyCell() {
      let minEntropy = 10000;
      let minEntropyCell = null;
      for (let cell of this.grid) {
        if (cell.collapsed) continue;

        let row = cell.row;
        let col = cell.col;
        let rowNeighbors = this.grid.filter(item => item.row === row && item !== cell && !item.collapsed );
        let colNeighbors = this.grid.filter(item => item.col === col && item !== cell && !item.collapsed );
        let boxNeighbors = this.getBoxNeighbors(row, col).filter(item => item !== cell && !item.collapsed );
        let neighbors = rowNeighbors.concat(colNeighbors).concat(boxNeighbors);
        console.log("neighbors", neighbors);
        let entropy = neighbors.length;
        if (entropy < minEntropy) {
          minEntropy = entropy;
          minEntropyCell = cell;
        }
      }
      console.log("minEntropyCell", minEntropyCell);
      return minEntropyCell;
    },
  },
}
</script>