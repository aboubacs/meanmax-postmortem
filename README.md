# Meanmax-postmortem

Woooooo ! 5th ! :D But it does feel like a win to me ! 

Super fun contest, big thanks to Magus, Agade, pb4 and reCurse ! 

I will describe my strategies and how I approached the game. I believe my AI had what it takes to get even further, but missed out on tuning to be able to approach this super close top 4.

Some background is important to understand why I did things like I did. I'm a totally Codingame-made player, as I knew close to nothing about AI's before landing in here. My first contest was Coders Strike Back, and I watched Jeff06 take over the contest with his genetic algorithm. I read his article explaining his algorithm with amazement, and could only dream about being able to do something like that one day. With time, more contests, more reading of post-mortems, I was able to gather information and learn a lot from the people in this community, until I was able to reproduce that in the arena.

When I saw Mean-Max, it reminded me a lot about Coders Strike Back, and I knew it was time to go back to the roots. 

## Evolution

I'm not going to go into the details on how an evolutionary algorithm works, because some people wrote about it far better than I would ([Jeff06 Post mortem](https://www.codingame.com/blog/genetic-algorithms-coders-strike-back-game/), [Magus article](http://files.magusgeek.com/csb/csb_en.html)). I'm just going to describe what characteristics did work for me. 

**Disclaimer** : if you find any troubling similarities with features presented in past contest reports... it's because I shamelessly copied from them.

- My genome is made of triplets of floats between 0 and 1. Each triplet encodes an action for one of my units for a turn. So I have 3 triplets of genes for a turn, and I simulated at depth 3.

- From a triplet, you can deduce an action. For example, for my reaper it would be :

```c++

if (gene3 < 0.1) {
	move to the closest wreck at thrust 300;
} else if (gene3 < 0.95) {
    if (gene1 > 0.75) {
    	thrust = 300;
    } else if (gene1 < 0.25) {
    	thrust = 0;
    } else {
        else thrust = 300 * (gene1 - 0.25) * 2;
    }
	Same idea with angle and gene 2
} else {
	Use your skill (use gene1 and gene2 to know in which direction)
}
```

There are 2 things to notice from that pseudo code :
- Hardcoded moves are included in the possibilities. This was a noticeable improvement for my AI.
- The limits of some genes makes my algorithm select more frequently very low and very high values for thrust. The reasoning is that you will probably want to go at thrust 300 or thrust 0 more often than any random value in between. So you increase the odds of your genes generating those solutions.

I then generated a population of 5 random genomes plus the best solution found by my algorithm the turn before, to a total of 6 genomes. While I have time left, I select randomly one of those 6, mutate it and evaluate it. If it's better than any of the 6, it will take its place. Repeat.

When time goes on, the amount by which a genome can mutate is reduced. At the start of the simulation, the "mutation" generates a nearly random genome. However, if there is 10% time left on the clock, then any gene can only move up or down of 0.1.

## Evaluation function

Then it was time to play with the evaluation function. And damn I played hard. 

### Data to consider

The first step was to find some characteristics of the game I wanted to look for when evaluating a situation. I took :
- The score of the player.
- The rage of the player.
- The average distance between the reaper of the player and water, weighted by the amount of water in those flasks of water.
- The distance between the reaper of the player and its destroyer.
- The distance between the reaper of the player and enemy doofs.
- The distance between the destroyer of the player and the closest tanker.

I made some formulas that would make all those parameters fit between 0 and 1, with 1 being "good" and 0 "bad". I multiplied them by A, B, C, D, E and F. To evaluate my score, I used : ``eval(me) - (eval(opponent1) + eval(opponent2)) / 2``. I then built a local arena that would reproduce the rules of the game. With this, I was able to try different versions of A, B, C, D, E and F, and make those versions play against one another. 

### Evolution of the parameters

What is quite funny is that I actually used the same algorithm I had to evolve my genome in the game, but this time to make those parameters evolve. I had a population of 9 programs that would play against each other. Every 5 rounds, I would remove the 3 worst programs, and replace them with mutations of the 3 best programs. The amount by which a program can mutate is reduced depending on the time already spent. Programs that placed 4 to 6 were left untouched in the population. 

Interestingly, it didn't work at all at first. No program would survive more than a couple of generations. I figured that the game had enough variance that even a "good" algorithm could fail 5 games and get lost forever. To fix this, I stopped resetting ranking between each session of 5 rounds. This caused an other problem : quickly, there would be a gap too big between the 6 programs already in the population and the 3 that would come in, a gap so big that no newcomer could ever make it in 5 rounds. Finally, it started to show very interesting results when I mixed those approaches (reset the ranking... but not too much !).

Every couple of hours, I would take the current best A, B, C, D, E and F from my local arena, submit it in the real arena, and see my rank improve each time. It was very satisfying. But something was missing...

## Predicting the opponent

I thought that what would make the difference would be how accurate one could predict its opponents. Every round, I had an hardcoded move for my opponent. I don't think it was super effective, so I tried another approach a lot of people use in those kind of challenges : use your own algorithm for some time and apply it to your opponents. 

### Evolutionary approach

When I confronted in my arena a version of my algorithm using an hardcoded move for my opponent against one using my own algorithm to predict it, the results were stunning. The change was soooooo much better ! I never submitted in the real arena a change so fast... and it was catastrophic in the online arena. It took me some time to understand why, and some help from my friend Garvys, with whom we shared a lot of ideas during this contest. He pointed out that obviously, my algorithm was very good at predicting... itself, which is why he would beat *my* previous version, but it didn't mean it was good at predicting other players, who might have different "strategies". 

It actually made a lot of sense, and it was confirmed when I looked closely on cg.stats (the online tool from Magus), to check winrates against individual players. I had a lot of very good and very bad winrates against people around me. Those super bad winrates would prevent me from going further in the rankings.

### Evolve the evolution...

So what did I do ? Well, this is getting fancy ! I used different hardcoded patterns + my own algorithm to predict the moves of my opponents each turn. I calculated the expected position of every unit according to each of those possibilities. The turn after, I would calculate the difference between the **actual** position of the units with the expected position for each pattern. Then, I would choose to apply for each unit the pattern that was the least wrong in the past. This means that for example if my algorithm was good at predicting the reaper of opponent n°1, then it would be used to predict it, but if my hardcoded pattern n°2 was good at predicting the doof of opponent n°2, then it would be used. The "errors" at each turn were added so that my code could "change his mind" at one point and switch for example at turn 75 from hardcoded pattern n°1 to genetic evolution for destroyer n°2.

I felt like it was very weird to mix things like that, but it proved to be not that bad ! And then at 9h53, I changed a parameter from 0.546349 to 0.53333 to try my luck, and went to work. :)

# Conclusion

I'm super happy and proud of my result ! This is the first time I manage to have a ranking like this in a contest ! This is for sure due to CodinGame, and the amazing resources here that I was able to learn from, so thank you to all of the people at CodinGame and the community. It was my goal to be able to piece together an AI like that one day, and I feel like for the first time I did it ! It took a lot of time and energy, but in the end, if you have a goal that you deep down inside know feels right, it is right ! ;)

