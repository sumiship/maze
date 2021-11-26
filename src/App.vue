<template>
  <div class="maze">
    <div class="maze__board">
      <div :style="styleMazeRow" class="maze_row" v-for="(mazeRow, rowIndex) in mazeBoard" :key="rowIndex">
        <div :style="[styleMazeCell, colorMazeCell(mazeCell)]" class="maze_cell" v-for="(mazeCell, cellIndex) in mazeRow" :key="cellIndex"></div>
      </div>
    </div>
  </div>
  <div class="control">
    <div class="control__button" @click="reset()">RESET</div>
    <div class="control__button" @click="createMaze()">START</div>
    <div class="control__button" @click="search()" :style="styleRouteButton">ROUTE</div>
    <div class="control__range">
      <p>Speed:{{ viewSpeed }}</p>
      <input type="range" max="100" min="2" v-model="viewSpeed" />
    </div>
    <div class="control__button">
      <p style="font-size: 1.3vh">横(奇数)</p>
      <input type="number" v-model="mazeSize.x" step="2" min="5" @change="mazeSize.x = formOddNumber(mazeSize.x)" />
    </div>
    <div class="control__button">
      <p style="font-size: 1.3vh">縦(奇数)</p>
      <input type="number" v-model="mazeSize.y" step="2" min="5" @change="mazeSize.x = formOddNumber(mazeSize.x)" />
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";

interface Cell {
  x: number;
  y: number;
}
interface Data {
  mazeSize: { x: number; y: number };
  mazeBoard: number[][];
  startCells: Cell[];
  viewSpeed: number;
  made: boolean;
}

export default defineComponent({
  data(): Data {
    return {
      mazeSize: { x: 31, y: 31 },
      mazeBoard: [[]],
      startCells: [],
      viewSpeed: 20,
      made: false,
    };
  },
  computed: {
    styleMazeRow(): string {
      return `height: ${(100 / this.mazeSize.y) * 0.85}vh`;
    },
    styleMazeCell(): string {
      return `width: ${(100 / this.mazeSize.y) * 0.85}vh`;
    },
    styleRouteButton(): string {
      if (this.made) {
        return "";
      } else {
        return "pointer-events: none; opacity: 0.3";
      }
    },
  },

  methods: {
    formOddNumber(number: number): number {
      let ans = Math.floor(number);
      if (ans % 2 != 1) ans++;
      if (ans < 5) ans = 5;
      return ans;
    },
    colorMazeCell(cell: number): string {
      let backgroundColor: string;
      switch (cell) {
        case 1:
          backgroundColor = "#D1E8E0";
          break;
        case 2:
          backgroundColor = "red";
          break;
        case 3:
          backgroundColor = "#45B4B4";
          break;
        case 4:
          backgroundColor = "#B44545";
          break;
        default:
          backgroundColor = "#0A412E";
      }
      return `background-color: ${backgroundColor}`;
    },
    async createMaze(): Promise<void> {
      const beginCell: Cell = { x: 1 + Math.floor(Math.random() * ((this.mazeSize.x - 1) / 2)) * 2, y: 1 + Math.floor(Math.random() * ((this.mazeSize.y - 1) / 2)) * 2 };
      for (let y = 0; y < this.mazeSize.y; y++) {
        for (let x = 0; x < this.mazeSize.x; x++) {
          if (x == beginCell.x && y == beginCell.y) {
            this.mazeBoard[y][x] = 1;
          } else {
            this.mazeBoard[y][x] = 0;
          }
        }
      }
      // 穴掘り開始
      await this.dig(beginCell);
    },
    async dig(cell: Cell): Promise<void> {
      for (;;) {
        let directions: number[] = [];
        if (cell.y - 2 >= 0) {
          if (this.mazeBoard[cell.y - 1][cell.x] == 0 && this.mazeBoard[cell.y - 2][cell.x] == 0) directions.push(0);
        }
        if (cell.x + 2 <= this.mazeSize.x) {
          if (this.mazeBoard[cell.y][cell.x + 1] == 0 && this.mazeBoard[cell.y][cell.x + 2] == 0) directions.push(1);
        }
        if (cell.y + 2 <= this.mazeSize.y - 1) {
          if (this.mazeBoard[cell.y + 1][cell.x] == 0 && this.mazeBoard[cell.y + 2][cell.x] == 0) directions.push(2);
        }
        if (cell.x - 2 >= 0) {
          if (this.mazeBoard[cell.y][cell.x - 1] == 0 && this.mazeBoard[cell.y][cell.x - 2] == 0) directions.push(3);
        }

        // 掘り進められない場合、ループを抜ける
        if (directions.length == 0) break;

        switch (directions.splice(Math.floor(Math.random() * directions.length), 1)[0]) {
          case 0:
            await this.setPath({ x: cell.x, y: cell.y - 1 });
            await this.setPath({ x: cell.x, y: cell.y - 2 });
            cell = { x: cell.x, y: cell.y - 2 };
            break;
          case 1:
            await this.setPath({ x: cell.x + 1, y: cell.y });
            await this.setPath({ x: cell.x + 2, y: cell.y });
            cell = { x: cell.x + 2, y: cell.y };
            break;
          case 2:
            await this.setPath({ x: cell.x, y: cell.y + 1 });
            await this.setPath({ x: cell.x, y: cell.y + 2 });
            cell = { x: cell.x, y: cell.y + 2 };
            break;
          case 3:
            await this.setPath({ x: cell.x - 1, y: cell.y });
            await this.setPath({ x: cell.x - 2, y: cell.y });
            cell = { x: cell.x - 2, y: cell.y };
            break;
        }
      }
      const startCell = this.getStartCell();

      if (startCell != null) {
        this.dig(startCell);
      } else {
        this.made = true;
      }
    },
    setPath(cell: Cell): Promise<void> {
      return new Promise((resolve) => {
        const id = setTimeout(() => {
          this.mazeBoard[cell.y][cell.x] = 1;
          if (cell.x % 2 == 1 && cell.y % 2 == 1) {
            this.startCells.push(cell);
          }
          clearTimeout(id);
          resolve();
        }, this.viewSpeed);
      });
    },
    getStartCell(): Cell | null {
      if (this.startCells.length == 0) return null;
      // ランダムに開始座標を取得する
      return this.startCells.splice(Math.floor(Math.random() * this.startCells.length), 1)[0];
    },

    async search() {
      this.made = false;

      let isGoaled = false;
      let queue: Cell[] = [];

      queue.push({ x: 1, y: 1 });
      let visited: Cell[][] = [...Array(this.mazeSize.y)].map(() => Array(this.mazeSize.x).fill({ x: -1, y: -1 }));

      while (!isGoaled && queue.length > 0) {
        const target: Cell = queue.splice(0, 1)[0];

        for (let i = 0; i < 4; i++) {
          let nextCell: Cell = JSON.parse(JSON.stringify(target));
          switch (i) {
            case 0:
              nextCell.y--;
              break;
            case 1:
              nextCell.x++;
              break;
            case 2:
              nextCell.y++;
              break;
            default:
              nextCell.x--;
          }

          if (nextCell.x >= 0 && nextCell.y >= 0 && nextCell.x < this.mazeSize.x && nextCell.y < this.mazeSize.y) {
            // 未探索の場合かつ通路の場合のみキューに詰めると同時に探索済情報設定
            // if (visited[nextCell.y][nextCell.x] == { x: -1, y: -1 } && this.mazeBoard[nextCell.y][nextCell.x] == 1) {
            if (this.mazeBoard[nextCell.y][nextCell.x] == 1) {
              // 探索済情報
              await this.setVisited(nextCell, target, visited);
              if (nextCell.x == this.mazeSize.x - 2 && nextCell.y == this.mazeSize.y - 2) {
                // 探索候補がゴールの場合すぐに抜けるために探索候補を削除して抜ける
                queue.length = 0;
                isGoaled = true;
                break;
              } else {
                // キューに詰める
                queue.push(nextCell);
              }
            }
          }
        }
      }
      if (isGoaled) {
        let followCell: Cell = { x: this.mazeSize.x - 2, y: this.mazeSize.y - 2 };
        await this.setFollowRoute(followCell);
        do {
          followCell = visited[followCell.y][followCell.x];
          await this.setFollowRoute(followCell);
        } while (!(followCell.x == 1 && followCell.y == 1));
      }
    },
    setVisited(nextCell: Cell, target: Cell, visited: Cell[][]): Promise<void> {
      return new Promise((resolve) => {
        setTimeout(() => {
          visited[nextCell.y][nextCell.x] = target;
          this.mazeBoard[nextCell.y][nextCell.x] = 3;
          resolve();
        }, this.viewSpeed);
      });
    },
    setFollowRoute(followCell: Cell): Promise<void> {
      return new Promise((resolve) => {
        setTimeout(() => {
          this.mazeBoard[followCell.y][followCell.x] = 4;
          resolve();
        }, this.viewSpeed);
      });
    },

    reset() {
      this.mazeBoard = [[]];
      this.made = false;
      for (let i = 0; i < this.mazeSize.y; i++) {
        this.mazeBoard[i] = Array(this.mazeSize.x).fill(0);
      }
    },
  },

  watch: {
    mazeSize: {
      handler: function () {
        if (this.mazeSize.x % 2 == 1 && this.mazeSize.y % 2 == 1) {
          this.reset();
        }
      },
      deep: true,
    },
  },

  mounted() {
    this.reset();
  },
});
</script>

<style>
* {
  padding: 0;
  margin: 0;
  box-sizing: border-box;
}
#app {
  height: 100vh;
  widows: 100vw;
  background-color: #3cb371;
}
.maze {
  position: relative;
  height: 90vh;
}
.maze__board {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
.maze_row {
  display: flex;
}
.maze_cell {
  /* border: 1px solid white; */
}

.control {
  height: 10vh;
  display: flex;
  justify-content: center;
  align-items: center;
}
.control__button {
  cursor: pointer;
  margin: 0 2vw;
  background-color: #79cf9f;
  font-size: 3vh;
  padding: 0.7vh;
  border-radius: 01vh;
}
.control__range {
  font-size: 2vh;
  text-align: center;
  width: 15vh;
}
.control__range input {
  width: 100%;
}
</style>
