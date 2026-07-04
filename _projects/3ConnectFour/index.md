---
layout: post
title: Connect Four
description: A console implementation of Connect Four built on a reusable abstract strategy-game framework. Supports a customizable board size and win-condition length, with full move validation and win detection in every direction.
skills: 
  - Java
  - Object-oriented design
  - Inheritance & polymorphism
  - 2D arrays

main-image: /board.svg
---

## Play it in your browser

Give it a try right here — take on a simple computer opponent, or switch to
two-player mode and challenge a friend.

<iframe src="/assets/connectfour/index.html" title="Playable Connect Four" loading="lazy"
        style="width:100%; max-width:520px; height:660px; border:1px solid #ddd; border-radius:12px; display:block; margin:0 auto; background:#f3f5fb;"></iframe>

<p style="text-align:center; margin-top:10px;">
  <a href="/assets/connectfour/index.html" target="_blank" rel="noopener">Open the game in a full page ↗</a>
</p>

---

## Overview

Connect Four is a two-player game where players take turns dropping tokens into columns;
the first to line up a run of tokens — vertically, horizontally, or diagonally — wins. This
implementation is built on top of a reusable AbstractStrategyGame framework, so the
same game engine (the Client) can run Connect Four, Tic-Tac-Toe, or any other perfect-
information strategy game without modification.

Two design choices make it flexible:

- Customizable rules — the board dimensions and the number of tokens needed to win are
  constructor parameters, so the default 6×7 / connect-4 game is just one configuration.
- Direction-agnostic win detection — a single helper checks for a run in any
  direction by taking a row/column step, so horizontal, vertical, and both diagonals reuse
  the same code.

---

## The contract: `AbstractStrategyGame`

Every game plugs into the engine by extending this abstract class. It defines what any
strategy game must be able to do — describe itself, report whose turn it is, read and apply
a move, and know when it's over — while leaving the actual rules to the subclass.

```java
import java.util.*;

/**
* A strategy game where all players have perfect information and no theme
* or narrative around gameplay.
*/
public abstract class AbstractStrategyGame {

    // Returns a String describing how to play the game.
    public abstract String instructions();

    // Returns a String representation of the current game state.
    public abstract String toString();

    // Returns true if the game has ended, and false otherwise.
    public boolean isGameOver() {
        return getWinner() != -1;
    }

    // Returns the index of the player who has won, or -1 if the game is not over.
    public abstract int getWinner();

    // Returns the index of the player who will take the next turn (-1 if the game is over).
    public abstract int getNextPlayer();

    // Prompts for and returns a String representation of the next player's move.
    public abstract String getMove(Scanner input);

    // Executes the move described by 'input', or throws if the move is illegal.
    public abstract void makeMove(String input);
}
```

Notice that `isGameOver()` is fully implemented in the base class in terms of the abstract
`getWinner()` — a small example of letting the superclass define shared behavior on top of
what subclasses provide.

---

## The implementation: `ConnectFour`

The board is a 2D `char` array. Dropping a token scans a column from the bottom up for the
first empty slot, then flips the turn. Win detection walks every filled cell and probes the
four directions with `checkWin`.

```java
import java.util.*;

public class ConnectFour extends AbstractStrategyGame {

    public static final char PLAYER_1_TOKEN = 'X';
    public static final char PLAYER_2_TOKEN = 'O';
    public static final char EMPTY = ' ';
    public static final int PLAYER_1 = 1;
    public static final int PLAYER_2 = 2;

    private char[][] board;
    private int goal;
    private int row;
    private int column;
    private boolean Xturn;

    // Default 6x7 board, connect 4 to win, Player 1 goes first.
    public ConnectFour() {
        this(6, 7, 4);
    }

    // Fully customizable board size and win length.
    public ConnectFour(int row, int column, int goal) {
        this.row = row;
        this.column = column;
        this.board = new char[row][column];
        this.goal = goal;
        for (int i = 0; i < row; i++) {
            for (int j = 0; j < column; j++) {
                this.board[i][j] = EMPTY;
            }
        }
        this.Xturn = true;
    }

    public String instructions() {
        return " The objective of the game is to be the first to form a horizontal,"
            + "vertical, or diagonal line of " + goal + " of one's own tokens. Two players"
            + "take turns dropping a token (X or O) into one of the " + column + " columns. "
            + "The token falls into the lowest empty space; the first player to get "
            + goal + " in a row, column, or diagonal wins. Enter a column number (1 - "
            + column + ") to make a move.";
    }

    public String toString() {
        String graph = "";
        for (int i = 0; i < this.row; i++) {
            graph += "|";
            for (int j = 0; j < this.column; j++) {
                graph += " " + this.board[i][j] + " |";
            }
            graph += "\n";
        }
        return graph;
    }

    public int getNextPlayer() {
        if (isGameOver()) {
            return -1;
        }
        return Xturn ? PLAYER_1 : PLAYER_2;
    }

    public String getMove(Scanner input) {
        if (input == null) {
            throw new IllegalArgumentException();
        }
        System.out.print("Choose Column 1 to " + this.column + ": ");
        return input.next();
    }

    // Drops the current player's token into the chosen column (lowest empty row),
    // validating the column, then switches turns.
    public void makeMove(String input) {
        if (input == null) {
            throw new IllegalArgumentException();
        }
        int col = Integer.parseInt(input) - 1;
        if (col >= this.column || col < 0) {
            throw new IllegalArgumentException();
        }
        if (this.board[0][col] != EMPTY) {
            throw new IllegalArgumentException();
        }
        boolean placed = false;
        for (int i = this.row - 1; i >= 0 && !placed; i--) {
            if (this.board[i][col] == EMPTY) {
                board[i][col] = Xturn ? PLAYER_1_TOKEN : PLAYER_2_TOKEN;
                placed = true;
            }
        }
        Xturn = !Xturn;
    }

    // Returns 1 / 2 for a winning player, 0 for a tie, or -1 if the game is not over.
    public int getWinner() {
        for (int i = 0; i < this.row; i++) {
            for (int j = 0; j < this.column; j++) {
                if (board[i][j] != EMPTY) {
                    if (checkWin(i, j, 1, 0) || checkWin(i, j, 0, 1)
                        || checkWin(i, j, 1, 1) || checkWin(i, j, 1, -1)) {
                        return this.board[i][j] == PLAYER_1_TOKEN ? PLAYER_1 : PLAYER_2;
                    }
                }
            }
        }
        for (int i = 0; i < this.column; i++) {
            if (this.board[0][i] == EMPTY) {
                return -1;
            }
        }
        return 0;
    }

    // Checks for a run of 'goal' identical tokens starting at (r, c), stepping by (dr, dc).
    // A single call covers a horizontal, vertical, or diagonal line depending on the step.
    public boolean checkWin(int r, int c, int dr, int dc) {
        char token = board[r][c];
        for (int i = 1; i < this.goal; i++) {
            int nextRow = r + i * dr;
            int nextCol = c + i * dc;
            if (nextRow < 0 || nextRow >= this.row || nextCol < 0 || nextCol >= this.column
                || board[nextRow][nextCol] != token) {
                return false;
            }
        }
        return true;
    }
}
```

The heart of it is `checkWin(r, c, dr, dc)`: by passing a direction as a `(dr, dc)` step —
`(0, 1)` for horizontal, `(1, 0)` for vertical, `(1, 1)` and `(1, -1)` for the two
diagonals — the same four lines of logic handle every winning direction instead of four
copy-pasted variants.
