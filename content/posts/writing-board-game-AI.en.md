---
title: "Writing board game AI"
date: 2020-09-04T19:59:38+09:00
lastmod: 2020-09-09T21:02:25+09:00
draft: false
author: "Toru Niina"
images: ["images/writing-board-game-AI/separo_featured_image.png"]
featuredImage: "images/writing-board-game-AI/separo_featured_image.png"
tags: ["board-game", "wasm", "rust"]
math:
  enable: true
code:
  copy: true
  maxshownLines: 65535
categories: ["Programming"]
---

{{< rawhtml >}}
<script async defer src="https://buttons.github.io/buttons.js"></script>
<a href="https://github.com/ToruNiina/separo-rs"><img src="https://img.shields.io/badge/view-on github-green"></a>
<a class="github-button" href="https://github.com/ToruNiina/separo-rs/subscription" data-show-count="true" aria-label="Watch ToruNiina/separo-rs on GitHub">Watch</a>
<a class="github-button" href="https://github.com/ToruNiina/separo-rs"              data-show-count="true" aria-label="Star ToruNiina/separo-rs on GitHub">Star</a>
<a class="github-button" href="https://github.com/ToruNiina/separo-rs/fork"         data-show-count="true" aria-label="Fork ToruNiina/separo-rs on GitHub">Fork</a>
{{< /rawhtml >}}

Have you ever heard of a game called "SEPARO"? Probably not.
It is a board game designed by my friend, [Takashi Suwa](https://github.com/gfngfn), when he was a high school student.
Some of his friends, including me, were interested in it and played it many times using a blackboard in a classroom.

It is clearly a brilliant board game.
Several weeks ago, I suddenly remembered and got the urge to play this game.
But, since only a few people know this game, I didn't have any opponents around me.

Building a server to play online is an option.
But it is not always easy to play a board game with friends because everyone needs to be free at the same time.
Another option is to create a player from scratch.
Like other problems, artificial intelligence (AI) comes to the rescue.
After all, AI always have time to play.

So, three weeks ago, I wrote AI which can play the board game.
Lately, I have been a bit busy, so I haven't had time to write this article.
But now that I have some time to spare, I'd like to share what I did.

## Background: Rule of SEPARO

![Example board state](images/writing-board-game-AI/separo_board.png)

SEPARO is a two-player board game using a 9x9 grid board, in which the aim is to separate the board into regions.
What determines the final score is the number of regions, and not their size.
The player with more regions at the end of the game wins.
Also, separated areas need to be larger than the size of one square-grid to be counted as a region.

![Examples of effective and ineffective territories](images/writing-board-game-AI/separo_territory.png)

One player uses red stones and the other uses blue ones. Red moves first and the game proceeds until neither player can make any valid move.

Each player take turns growing two "roots" that connect stones.
Players place stones on grid intersections and roots in the space between stones.
Roots can only grow from existing stones and in a specific way.
The first root always grows following a diagnoal line and needs to end at an empty intersection.
The second root follows from the first one and grows following either a horizontal or vertical line.
However, the intersection at the end of the second root can be occupied by one of your stones.
Finally, there is one extra rule: roots cannot form angles of 45 degree between them.

![Examples of valid_and_invalid moves](images/writing-board-game-AI/separo_valid_and_invalid_moves.png)

Is it too complicated? Don’t be afraid. My implementation is capable of showing the valid roots you can grow.
Also you can watch the game by two AI players to learn how to play.

You can play it on the following website. (The instructions are in Japanese, sorry!)
- https://toruniina.github.io/separo-rs/
- First you need to select players, Human or an algorithm. Then press "start".
- By dragging from an existing stone, you can grow your roots.

## Implementation of SEPARO

### Overall structure

Needless to say, a GUI is much user-friendly than a CUI for playing games.
There are many GUI libraries and frameworks for games, but nowadays we have another option: a web page.

I think web pages are more accessible to people than binary distributions.
Binaries run on the users’ environments, so users need to download it, install it, and finally run it.
Contrary, a web page requires only single step: clicking a link.
The browser automatically takes care of all the other complications.

Also, a binary needs to run on several operating systems and I would need to cross-compile it.
Since it is generally challenging to write GUI software that works correctly on multiple operating systems, some potential users who want to learn and play SEPARO may be discouraged by possible runtime errors.

Of course, web pages are not free from the difference in environments either.
There are several browsers, and sometimes the compatibility matters.
But still, if we decide to keep a simple design, it is relatively easy (I think).

When creating a web page as a GUI, a natural choice for the implementation is javascript.
But since I'm not very familiar with javascript, I'd prefer to write all the components but the UI (i.e., AI and the rules of the game) in another language.
Nowadays, the modern approach to connect javascript with other languages is to compile the code into webassembly.
Thus, I decided to implement the game in Rust and compile it using wasm-pack.

### Board state and Score calculation

To verify that a player's move is valid, the presence and color of stones on each grid and the roots' orientation connected to the grid are sufficient.
So the `Board` and the `Grid` needs to hold the following data.

```rust
pub struct Grid {
    color: Option<Color>,
    roots: ArrayVec<[Dir; 4]>,
}
pub struct Board {
    width: usize,     // == 9 normally
    grids: Vec<Grid>, // 9x9 grids
}
```

Next, let's figure out how to calculate the score.
In SEPARO, the score is determined by the number of regions a player divided the board.
Counting the number of territories on a board becomes equivalent to counting the number of connected components in the corresponding graph by mapping the units that cannot be divided further into nodes and the connection between those into graph edges.
Once we find the mapping, the implementation becomes easy because the number of connected components can be determined by a simple breadth-first search.

Looking at the board, we can observe that we can, at most, divide a grid into 4 small isosceles right triangles.
It means that the following graph structure is sufficient to count the number of territories.
Small cyan circles represent the nodes and thin black lines that connect nodes represent the edges.
Thick blue lines represent the blue roots. Each root breaks the edges of this internal graph.

![Internal graph used to calculate the score](images/writing-board-game-AI/separo_implementation.png)

This representation has another advantage.
Since all the regions corresponding to the nodes have the same size, the number of nodes in a connected component is proportional to the area of the territory corresponding to the component.
So whether the territory is effective or not can be determined by counting the number of nodes in the corresponding connected component.

I added this internal graph representation to the `Board`.
When a new root is grown, the corresponding edges are removed from this internal graph.
Note that two different `Graph`s are needed because scores of red and blue are calculated independently.

```rust
pub struct Board {
    width: usize,     // == 9 normally
    grids: Vec<Grid>, // 9x9 grids
    red:   Graph,     // to calculate score
    blue:  Graph,     // ditto
}
```

I decided to use the `<canvas>` element to render the board on the webpage.
To pass data from Rust to javascript, the [rustwasm tutorial](https://rustwasm.github.io/book/game-of-life/implementing.html#rendering-to-canvas-directly-from-memory) shows the way to access the webassemly memory region directly.
I know that is the most efficient way to pass large data from Rust/wasm to javascript.
But this time I took a simpler way. I encoded the board state into JSON and passed it to javascript as a `String`.
Because I don't need to render the board in 60 FPS, so the rendering is not a hotspot.

It took me a whole day to implement the board and related methods, including the rendering on the webpage.

## AI algorithms

Even a function that selects a move randomly from the possible moves can play against us.
But such an algorithm is not strong enough to entertain a human.

AI makes a decision responding to player actions.
Here, what is decided is the next move, i.e., the next roots to be grown.
To determine which move is the best, AI algorithm calculates each move's score and choose the one with the maximum score.
Therefore AI algorithm requires a function that takes a board state as input and returns the score of the board as output.
If we could obtain such a function, writing AI would be an easy task.
That is, first, list all the possible next moves.
Then, calculate each move's score.
And finally, select the move that has the maximum score.

The difficult part is to design such a function.
The scores and the number of possible moves should be an important factor, but balancing the two is not an easy task.
Also, the value of a move depends on the environment.
If a growing root cannot separate the board, the move is basically meaningless.
But if the root can inhibit the opponent's root that would separate the board, it becomes a valuable move.

Since SEPARO is a game that has only been around for a short time, no one knows what a good move is.
Designing an evaluation function manually is too hard, at least to me.
So I needed an algorithm to evaluate the score without the knowledge of the standard tactics.

The neural network technologies can, of course, resolve this problem.
Neural networks are essentially a function that approximate another function, e.g. a function that takes an image and returns a name of an object in the scene.
A neural network that takes a board state and returns the best next move could be trained, but I don’t have enough computational resources nor time to do that.
I wanted to play this game as soon as possible, so I employed a more classical approach.

### Monte Carlo SEPARO

The most obvious score is the win rate. But how do we calculate it?

One straightforward method is to approximate the win rate from many independent simulations.
By simulating the game until it ends, we can obtain a sample of the game and easily determine which player wins.
After collecting a large number of samples, we can approximate the win rate as how many times the AI wins.

The simplest way to simulate the game is to select moves at random. That is, taking a sample from a uniform distribution.
Of course, it is not the strongest algorithm. But still, it sometimes defeats a beginner.

![Graphical explanation of naive MC](images/writing-board-game-AI/separo_naive_MC.png)

Methods that obtain (often approximated) answers by repeating random sampling are called Monte Carlo method, after the city famous for its casinos because it is a game of chance.

### Monte Carlo Tree Search

The accuracy of the approximation of the win rate depends on the quality of the simulation.
If the players in the simulation make moves that are not actually chosen very often by the real players, the win rate would not be reliable.
Apparently, a random simulation is not so realistic, especially when playing against humans.

To perform a more realistic simulation, we need to know which move is more likely to be chosen for each board state.
Of course, we do not have a function that calculates the probability that a move being chosen. (that is precisely the problem we want to solve!)
That's why we need to use the estimated win rate in a better way to find the mosty promising move in each situation.

Monte Carlo tree search (MCTS) is an algorithm that constructs a search tree and finds the move that is most likely to be chosen.
A node of the search tree corresponds to a move.
For each node, we can assign a win rate by accumulating the subtree results under the node.

The MCTS algorithm consists of the following steps.

1. Starting from the root node that represents the current state of the board, it selects child nodes with the highest score until it reaches the leaf node.
2. If the leaf node is visited more times than the predefined threshold, it expands the tree from the leaf node. That is, it adds child nodes to the leaf node.
3. It performs a random playout starting from the leaf node. The result of the game will propagate from the leaf to the root.

![Graphical explanation of MCTS](images/writing-board-game-AI/separo_MCTS.png)

Assuming that both player want to maximize the win rate, it is better to collect more samples and analyze deeper at the node with the highest win rate.
This algorithm achieves it automatically.
It selects the node with the highest score within the child nodes when traversing from the root to the leaf.
That means that the node with the higher score will be visited more.
Because it expands a tree from the leaf node that is visited many times, it makes the tree deeper and collects more samples from the nodes with the highest score.

The problem of using a raw win rate as a score is that this algorithm always selects the node with the highest score, so it may end up with a node that has a high win rate just by chance.
To avoid it, I employed the "upper confidence bound 1 applied to trees" (UCT) algorithm developed by Levente Kocsis and Csaba Szepesvári in 2006.
The UCT algorithm uses the following score function called upper confidence bound 1 (UCB1) introduced by Peter Auer, Nicolò Cesa-Bianchi, and Paul Fischer in 2002.

\\[ \mathrm{score}(\mathrm{node}) = \frac{w_i}{n_i} + c\sqrt{\cfrac{\ln N_i}{n_i}} \\]

Here, \\(w_i\\) represents the number of wins for the nodes, and \\(n_i\\) represents the number of samples under the node.
So the first term, \\(w_i/n_i\\), shows the win rate of the node.
\\(N_i\\) stands for the total number of samples collected by the parent node and \\(c\\) is the parameter usually chosen empirically.
The second term corresponds to a sort of "error bar" of the win rate.
The smaller the number of samples collected from that node, the larger the value of the second term.
So the sum of the terms corresponds to the "upper confidence bound" of the win rate.

The UCB1 score function was originally developed in the field of [the multi-armed bandit problem](https://en.wikipedia.org/wiki/Multi-armed_bandit).
The problem is to find the optimal way to maximize the gain from a large number of slot machines of unknown probability of getting a reward.
Selecting a move that has the highest win rate from many possible moves is equivalent to the multi-armed bandit problem.

When I implemented the UCT algorithm, I made it possible to reuse the previous search tree to make it stronger.
After searching and expanding the tree for a while, the algorithm selects the next move from the child nodes of the root node.
Clearly, the subtree under the selected node can be reused.
It increases the total number of samples step by step and make the decision more accurate.
The AI needs to find a node that represents the current state of the board at the beginning of the turn, though.

![Graphical explanation of reusing a tree](images/writing-board-game-AI/separo_MCTS_impl.png)

This algorithm searches the best move efficiently. It was stronger than human, including the original creator of SEPARO, Takashi Suwa, and me, for a while.

## Conclusion

After I published the repository, some people played SEPARO.
The first day no one beat UCT, but the next day one of my friends, [Suguru Kato](https://github.com/0ncorhynchus), defeated UCT, taking double the score.
Curiously, many people have been able to defeat UCT since then.
The ability of humankind to learn things is a threat to machinery.

Through this, I found that having an AI player is effective when introducing a board game to someone else.
Artificial intelligence is always there for us when we want to play a game.
We can gain a lot of insights from a match between AI players.
Also, an AI with the right strength is useful for learning the game rules and practicing new strategies.

## Acknowledgement

I would like to thank T. Suwa for allowing me to use his game.
I am greateful to S. Kato for improving the code to make it work on Safari.
I also thank D. Ugarte for reading the draft and pointed out many grammatical errors.
