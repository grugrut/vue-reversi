<template>
  <div class="reversi">
    <p>黒 {{ countColor(1) }} : {{ countColor(2) }} 白</p>
    <table border="1" cellspacing="0">
      <tr v-for="y in 8" :key="y">
        <td v-for="x in 8" :key="x" v-on:click="clicked(x, y)">
          <span v-if="cells[y * 10 + x] === 1">&#x26ab;</span>
          <span v-if="cells[y * 10 + x] === 2">&#x26aa;</span>
        </td>
      </tr>
    </table>
    <p>
      <span v-if="turn === 1">黒の番です</span>
      <span v-if="turn === 2">白の番です</span>
    </p>
    <p>
      <span v-if="passFlag">パス</span>
      <span v-if="endFlag">{{ msg }}</span>
    </p>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";

const WALL = -1;
const EMPTY = 0;
const BLACK = 1;
const WHITE = 2;
const NUM_CELLS = 10 * 10;
const NUM_STACK = (6 * 3 + 3) * 8 * 8;

const DIR_NORTH = -10;
const DIR_NORTHEAST = -9;
const DIR_EAST = 1;
const DIR_SOUTHEAST = 11;
const DIR_SOUTH = 10;
const DIR_SOUTHWEST = 9;
const DIR_WEST = -1;
const DIR_NORTHWEST = -11;

function opponentColor(color: number) {
  return color === BLACK ? WHITE : BLACK;
}

let stack = Array(NUM_STACK);
let sp = 0;

function pushStack(num: number) {
  stack[sp] = num;
  ++sp;
}

function popStack(): number {
  if (sp === 0) {
    return -1;
  }
  --sp;
  return stack[sp];
}

export default defineComponent({
  name: "Reversi",
  data() {
    return {
      cells: Array(NUM_CELLS),
      turn: BLACK,
      passFlag: false,
      endFlag: false,
      msg: "",
    };
  },
  created() {
    for (let i = 0; i < NUM_CELLS; ++i) {
      this.cells[i] = WALL;
    }
    for (let y = 1; y <= 8; ++y) {
      for (let x = 1; x <= 8; ++x) {
        this.cells[10 * y + x] = EMPTY;
      }
    }
    this.cells[44] = BLACK;
    this.cells[45] = WHITE;
    this.cells[54] = WHITE;
    this.cells[55] = BLACK;
  },
  methods: {
    getPos: function (x: number, y: number): number {
      return y * 10 + x;
    },
    clicked: function (x: number, y: number) {
      if (this.flip(this.turn, this.getPos(x, y)) > 0) {
        this.changeTurn();
      }
    },
    changeTurn: function () {
      this.passFlag = false;
      this.turn = opponentColor(this.turn);
      if (!this.canPlay(this.turn)) {
        this.turn = opponentColor(this.turn);
        if (!this.canPlay(this.turn)) {
          // game end
          this.endFlag = true;
          if (this.countColor(BLACK) > this.countColor(WHITE)) {
            this.msg = "黒の勝ち";
          } else if (this.countColor(WHITE) > this.countColor(BLACK)) {
            this.msg = "白の勝ち";
          } else {
            this.msg = "引き分け";
          }
        } else {
          this.passFlag = true;
        }
      }
      if (this.turn == WHITE) {
        this.endSearch(5, -100, 100, this.turn, 0);
        this.changeTurn();
      }
    },
    flipLine: function (color: number, pos: number, dir: number): number {
      let result = 0;
      let p: number;
      const op = opponentColor(color);
      for (p = pos + dir; this.cells[p] == op; p += dir); /* Do nothing */
      if (this.cells[p] == color) {
        for (p -= dir; this.cells[p] == op; p -= dir) {
          ++result;
          this.cells[p] = color;
          pushStack(p);
        }
      }
      return result;
    },
    flip: function (color: number, pos: number): number {
      let result = 0;
      if (this.cells[pos] != EMPTY) {
        return 0;
      }

      result += this.flipLine(color, pos, DIR_NORTH);
      result += this.flipLine(color, pos, DIR_NORTHEAST);
      result += this.flipLine(color, pos, DIR_EAST);
      result += this.flipLine(color, pos, DIR_SOUTHEAST);
      result += this.flipLine(color, pos, DIR_SOUTH);
      result += this.flipLine(color, pos, DIR_SOUTHWEST);
      result += this.flipLine(color, pos, DIR_WEST);
      result += this.flipLine(color, pos, DIR_NORTHWEST);
      if (result > 0) {
        this.cells[pos] = color;
        pushStack(pos);
        pushStack(opponentColor(color));
        pushStack(result);
      }
      return result;
    },
    unflip: function (): number {
      if (sp === 0) {
        return 0;
      }
      const result = popStack();
      const color = popStack();
      this.cells[popStack()] = EMPTY;
      for (let i = 0; i < result; ++i) {
        this.cells[popStack()] = color;
      }
      return result;
    },
    countFlipLine: function (color: number, pos: number, dir: number) {
      let result = 0;
      let p: number;
      const op = opponentColor(color);
      for (p = pos + dir; this.cells[p] == op; p += dir) {
        ++result;
      }
      if (this.cells[p] != color) {
        return 0;
      }
      return result;
    },
    countFlip: function (color: number, pos: number) {
      let result = 0;
      if (this.cells[pos] != EMPTY) {
        return 0;
      }
      result += this.countFlipLine(color, pos, DIR_NORTH);
      result += this.countFlipLine(color, pos, DIR_NORTHEAST);
      result += this.countFlipLine(color, pos, DIR_EAST);
      result += this.countFlipLine(color, pos, DIR_SOUTHEAST);
      result += this.countFlipLine(color, pos, DIR_SOUTH);
      result += this.countFlipLine(color, pos, DIR_SOUTHWEST);
      result += this.countFlipLine(color, pos, DIR_WEST);
      result += this.countFlipLine(color, pos, DIR_NORTHWEST);
      return result;
    },
    canFlip: function (color: number, pos: number): boolean {
      if (this.cells[pos] != EMPTY) {
        return false;
      }
      if (this.countFlipLine(color, pos, DIR_NORTH) > 0) {
        return true;
      }
      if (this.countFlipLine(color, pos, DIR_NORTHEAST) > 0) {
        return true;
      }
      if (this.countFlipLine(color, pos, DIR_EAST) > 0) {
        return true;
      }
      if (this.countFlipLine(color, pos, DIR_SOUTHEAST) > 0) {
        return true;
      }
      if (this.countFlipLine(color, pos, DIR_SOUTH) > 0) {
        return true;
      }
      if (this.countFlipLine(color, pos, DIR_SOUTHWEST) > 0) {
        return true;
      }
      if (this.countFlipLine(color, pos, DIR_WEST) > 0) {
        return true;
      }
      if (this.countFlipLine(color, pos, DIR_NORTHWEST) > 0) {
        return true;
      }

      return false;
    },
    canPlay: function (color: number) {
      for (let x = 1; x <= 8; ++x) {
        for (let y = 1; y <= 8; ++y) {
          if (this.canFlip(color, this.getPos(x, y))) {
            return true;
          }
        }
      }
      return false;
    },
    countColor: function (color: number) {
      let result = 0;
      for (let x = 1; x <= 8; ++x) {
        for (let y = 1; y <= 8; ++y) {
          if (this.cells[this.getPos(x, y)] == color) {
            ++result;
          }
        }
      }
      return result;
    },
    endSearch: function (
      depth: number,
      alpha: number,
      beta: number,
      color: number,
      pass: number
    ) {
      let max_x = 0,
        max_y = 0,
        max = -100;
      for (let x = 1; x <= 8; ++x) {
        for (let y = 1; y <= 8; ++y) {
          if (this.flip(color, this.getPos(x, y)) > 0) {
            const val = -this.endSearchSub(
              depth - 1,
              -beta,
              -alpha,
              opponentColor(color),
              pass
            );
            this.unflip();
            console.log(val);
            if (val > max) {
              max = val;
              max_x = x;
              max_y = y;
            }
          }
        }
      }
      this.flip(this.turn, this.getPos(max_x, max_y));
    },
    endSearchSub: function (
      depth: number,
      alpha: number,
      beta: number,
      color: number,
      pass: number
    ): number {
      let canMove = false;
      let value: number;
      let max = alpha;
      if (depth == 0) {
        return this.countColor(color) - this.countColor(opponentColor(color));
      }
      for (let x = 1; x <= 8; ++x) {
        for (let y = 1; y <= 8; ++y) {
          if (this.flip(color, this.getPos(x, y)) > 0) {
            if (!canMove) {
              canMove = true;
            }
            value = -this.endSearchSub(
              depth - 1,
              -beta,
              -max,
              opponentColor(color),
              0
            );
            this.unflip();
            if (value > max) {
              max = value;
              if (max >= beta) {
                return beta;
              }
            }
          }
        }
      }
      if (!canMove) {
        if (pass) {
          max = this.countColor(color) - this.countColor(opponentColor(color));
        } else {
          max = -this.endSearchSub(depth, -beta, -max, opponentColor(color), 1);
        }
      }
      return max;
    },
  },
});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
table {
  margin: auto;
}
td {
  width: 8vmin;
  height: 8vmin;
  font-size: 5vmin;
  line-height: 5vmin;
  padding: 0;
}
.reversi {
  margin: auto;
}
</style>
