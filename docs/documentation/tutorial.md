# Tutorial

This tutorial walks through a simple game of Tic-Tac-Toe.

## Setup

We shall use [create-react-app](https://github.com/facebookincubator/create-react-app) to create a **React** app and add
**boardgame.io** to it. The code uses ES2015 features,
including [imports](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import) and
the [object spread](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator) syntax.

```
$ npx create-react-app game
$ cd game
$ npm install --save boardgame.io
```

## Moves

We create the game by providing the initial value of the
game state `G` (through the `setup` function), and the moves
of the game. The `setup` function also accepts a
`ctx` parameter if you need to customize the initial
state based on some field in `ctx` (the number of players, for example),
but we don't need that for Tic-Tac-Toe.

In Tic-Tac-Toe, we have just one type of move that we shall
name `clickCell`. The move function accepts
the game state `G` and updates it to the desired new state.
`ctx` is a framework managed
object that contains metadata like `turn` and `currentPlayer`.
Everything after that is an argument that you pass in at the
call-site of this move.

```js
// src/App.js

import { Client } from 'boardgame.io/react';

const TicTacToe = {
  setup: () => ({ cells: Array(9).fill(null) }),

  moves: {
    clickCell: (G, ctx, id) => {
      G.cells[id] = ctx.currentPlayer;
    },
  },
};

const App = Client({ game: TicTacToe });

export default App;
```

!> The move function can receive any number of additional
arguments that are passed to it from the call-site.

Run the app using:

```
npm start
```

```react
<iframe class='react' src='snippets/example-1' height='800' scrolling='no' title='example' frameborder='no' allowtransparency='true' allowfullscreen='true' style='width: 100%;'></iframe>
```

Notice that we have a fully playable game that we can
interact with using the Debug Panel with just this little piece of code!

?> You can make a move by clicking on `clickCell` on the
Debug Panel (or pressing the keyboard shortcut `c`),
entering `0` as an argument and pressing
`Enter` to have the current player make a move on the 0-th
cell. This `id` is available inside the move function as
the first argument after `G` and `ctx`. Notice how the
`cells` array now has a `0` in the 0-th position. You
can end the turn using `endTurn` and the next call to
`clickCell` will result in a `1` in the chosen cell.

!> The Debug Panel can be turned off by passing `debug: false`
in the `Client` config.

## Victory Condition

The Tic-Tac-Toe game we have so far doesn't really terminate.
Let's keep track of a winner in case one player wins the game.
In order to do this, we add a game termination condition
to `endIf`.

Let's also prevent players from being able to overwrite cells:

```js
// Return true if `cells` is in a winning configuration.
function IsVictory(cells) {
  ...
}

// Return true if all `cells` are occupied.
function IsDraw(cells) {
  return cells.filter(c => c === null).length == 0;
}

const TicTacToe = {
  setup: () => ({ cells: Array(9).fill(null) }),

  moves: {
    clickCell: (G, ctx, id) => {
      if (G.cells[id] === null) {
        G.cells[id] = ctx.currentPlayer;
      }
    },
  },

  endIf: (G, ctx) => {
    if (IsVictory(G.cells)) {
      return { winner: ctx.currentPlayer };
    }
    if (IsDraw(G.cells)) {
      return { draw: true };
    }
  },
};
```

!> `endIf` takes a function that determines if
the game is over. If it returns anything at all, the game ends and
the return value is available at `ctx.gameover`.

## Game Board

**React** is a great fit for board games because
it provides a declarative API to translate objects
to UI elements.

Creating a board is a fairly mechanical process of
translating the game state `G` into actual cells that
are clickable.

```js
import React from 'react';

class TicTacToeBoard extends React.Component {
  onClick(id) {
    if (this.isActive(id)) {
      this.props.moves.clickCell(id);
      this.props.events.endTurn();
    }
  }

  isActive(id) {
    if (!this.props.isActive) return false;
    if (this.props.G.cells[id] !== null) return false;
    return true;
  }

  render() {
    let winner = '';
    if (this.props.ctx.gameover) {
      winner =
        this.props.ctx.gameover.winner !== undefined ? (
          <div id="winner">Winner: {this.props.ctx.gameover.winner}</div>
        ) : (
          <div id="winner">Draw!</div>
        );
    }

    const cellStyle = {
      border: '1px solid #555',
      width: '50px',
      height: '50px',
      lineHeight: '50px',
      textAlign: 'center',
    };

    let tbody = [];
    for (let i = 0; i < 3; i++) {
      let cells = [];
      for (let j = 0; j < 3; j++) {
        const id = 3 * i + j;
        cells.push(
          <td style={cellStyle} key={id} onClick={() => this.onClick(id)}>
            {this.props.G.cells[id]}
          </td>
        );
      }
      tbody.push(<tr key={i}>{cells}</tr>);
    }

    return (
      <div>
        <table id="board">
          <tbody>{tbody}</tbody>
        </table>
        {winner}
      </div>
    );
  }
}
```

The important bit to pay attention to is about how to
dispatch moves. We have the following code in our click
handler:

```js
this.props.moves.clickCell(id);
this.props.events.endTurn();
```

`props.moves` is an object passed in by the framework that
contains functions to dispatch moves. `props.moves.clickCell`
dispatches the _clickCell_ move, and any data passed in is made
available in the move handler.

`props.events` is an object that contains functions to control
the flow of the game, including the ability to end the turn,
pass or end the current game phase (see [Phases](phases.md)).

!> The framework doesn't end the turn until `endTurn` is called.
This is to facilitate games where a player can take a number
of moves before the next player is allowed to play.

Now, we pass this component to our `Client` call.

```js
const App = Client({
  game: TicTacToe,
  board: TicTacToeBoard,
});

export default App;
```

And there you have it. A basic tic-tac-toe game!

```react
<iframe class='react' src='snippets/example-2' height='850' scrolling='no' title='example' frameborder='no' allowtransparency='true' allowfullscreen='true' style='width: 100%;'></iframe>
```

!> You can press `1` (or click on the button next to `reset`) to reset the
state of the game and start over.

## Bots

In this section we will show you how easy it is to add a bot that is
capable of playing your game. All you need to do is just tell the
bot how to find legal moves in the game, and it will find moves that
work well.

We shall first modify our game by adding a useful option
called `moveLimit` to automatically end the player's turn after
a single move has been made. That way the bot doesn't have to worry about
issuing `endTurn` calls (which, while possible, makes the game tree
a bit messier to search).

```js
const TicTacToe = {
  setup: () => { ... },
  moves: { ... }

  turn: {
    moveLimit: 1,
  },
}
```

After that, add an `ai` section to the game config:

```js
const TicTacToe = {
  ...

  ai: {
    enumerate: (G, ctx) => {
      let moves = [];
      for (let i = 0; i < 9; i++) {
        if (G.cells[i] === null) {
          moves.push({ move: 'clickCell', args: [i] });
        }
      }
      return moves;
    },
  },
};
```

That's it! Now that you can visit the AI section of the Debug Panel:

- `play` causes the bot to calculate and make a single move
  (shortcut: <kbd>2</kbd>)

- `simulate` causes the bot to play the entire game by itself
  (shortcut: <kbd>3</kbd>)

`play` helps you combine moves that you make yourself
and bot moves. For example, you can make
some manual moves to get two in a row and then verify that
the bot makes a block.

```react
<iframe class='react' src='snippets/example-3' height='850' scrolling='no' title='example' frameborder='no' allowtransparency='true' allowfullscreen='true' style='width: 100%;'></iframe>
```

!> The bot uses [MCTS](https://nicolodavis.com/blog/tic-tac-toe/) under the
hood to explore the game tree and find good moves. The default uses
1000 iterations per move.  This can be configured to adjust the
bot's playing strength.

The framework will come bundled with a few different bot algorithms, and an advanced
version of MCTS that will allow you to specify a set of objectives to optimize for.
For example, at any given point in the game you can tell the bot to gather resources
in the short term and wage wars in the late stages. You just tell the bot what to do
and it will figure out the right combination of moves to make it happen!

Detailed documentation about all this is coming soon. Adding bots to games for actual
networked play (as opposed to merely simulating moves) is also in the works.

[![Edit boardgame.io](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/boardgameio-wlvi2)
