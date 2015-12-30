---
layout: post
title: Solving the Monty Hall problem with Python
categories: Programming
---

Recently someone on IRC brought up the Monty Hall paradox, and said the conclusion of this was that the probabilities were not always right.
I wanted to show that it was in fact a cognitive bias of our brain, and since the formal proof was not enough, I decided to test it in a little computer experiment.

For those of you who don't know the Monty Hall Paradox, it is a standard mathematical puzzle loosely based on the TV show "Let's Make a Deal", hosted by Monty Hall.
The problem is stated as follows:

>Suppose you're on a game show, and you're given the choice of three doors: Behind one door is a car; behind the others, goats.
>You pick a door, say No. 1, and the host, who knows what's behind the doors, opens another door, say No. 3, which has a goat.
>He then says to you, "Do you want to pick door No. 2?" Is it to your advantage to switch your choice?

The intuitive answer to this question is that changing door does not change anything
All doors should have a 33% chance of winning and changing the door won't change this fact.
However, an exhaustive analysis of all the game possibilities show that changing the door brings the win probability to 66%!
The complete reasoning behind this can be found on Wikipedia, but since the goal was to do it experimentally, I will use my favorite hammer in my digital toolbox: Python.
*Spoiler:* The results were the same to the theoretical ones (66% win probability when changing door, 33% when not changing).

First of all here is the complete code.
I will go into details aftewards.

{% highlight python %}
import random

DOORS = {1, 2, 3}

def choice(set):
    return random.choice(tuple(set))

def choose_new_door(picked, removed_door):
    """
    Changes the door based on the removed door.
    """
    possible_door_set = DOORS - {picked, removed_door}
    return list(possible_door_set)[0]

def pick_removed_door(winning, picked):
    """
    Returns the door that the presenter will remove.
    This door is not the winning door and not the door the player picked.
    """
    return choice(DOORS - {winning, picked})

def compute_proba(change_door, count=10000):
    """
    Returns the experimental probability of winning.

    The player strategy is to change the door after the presenter removed one
    if change_door is True, and to keep the initial door choice otherwise.
    """

    win_count = 0

    for _ in range(count):
        player_choice = choice(DOORS)
        winning_choice = choice(DOORS)

        removed_door = pick_removed_door(winning_choice, player_choice)

        if change_door:
            player_choice = choose_new_door(player_choice, removed_door)

        if player_choice == winning_choice:
            win_count = win_count + 1

    return win_count / count

print("When changing door the win proba is {}".format(compute_proba(True)))
print("When keeping the same door the win proba is {}".format(compute_proba(False)))

{% endhighlight %}

Let's start with this function:
{% highlight python %}
def compute_proba(change_door, count=1000000):
{% endhighlight %}

It is the main function of the program, which will run our experiment a given number of time and with a given strategy.
If `change_door` is true the player will decide to change its choice of door after the host removed a door.
Otherwise it will stick to its initial door choice.

{% highlight python %}
player_choice = choice(DOORS)
winning_choice = choice(DOORS)
removed_door = pick_removed_door(winning_choice, player_choice)
{% endhighlight %}

Here we simply randomly attribuate the winning door and the player choice.
The host then picks a door to remove.
This door cannot be the winning door nor the player choice (Python's sets are perfect for this).

{% highlight python %}
if change_door:
    player_choice = choose_new_door(player_choice, removed_door)
{% endhighlight %}
Here the player changes it's door choice if asked to do so.
The algorithm for this is easy: simply take the door that is not the initial choice nor the removed one.
Again we will use Python's sets for this.

{% highlight python %}
if player_choice == winning_choice:
    win_count = win_count + 1
{% endhighlight %}
Finally, if the player choice is the correct one we increment the counter and start again.

The rest of the code is pretty easy, so I won't dive in it.
Instead I will try to explain what are the sets and how you can use it in that kind of case.

#Python sets
The `set` object in Python is pretty much the same as mathematical set.
It is a non-ordered collection which guarantees that each stored object is unique.
The main advantage of sets is the ability to perform mathematical operations on it.

{% highlight python %}
{1, 2, 3} + {3, 4} # {1, 2, 3, 4}
{1, 2, 3} - {2} # {1, 3}
{1, 2, 3} & {2, 4} # {2}
{% endhighlight %}

It is very useful when you need a way to compute properties about group of objects without caring about the order.
Here I used them to compute the valid possibilities for door choice in different scenarios.
Doing it with list would have been much harder.


#References
* [http://en.wikipedia.org/wiki/Monty_Hall_problem](Wikipedia article)
* [https://docs.python.org/3.4/library/stdtypes.html#set-types-set-frozenset](Python set reference)
