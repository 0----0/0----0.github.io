---
title: Fighting Game -- In Review
tags: TeXt
---

When I set out to create a fighting game by myself, I did it with the mindset of a limit test.  Rather than believethat it was in-scope for me to complete an entire 2D fighting game by myself, I wanted to see just how much progress I could make, and explore what methods I could use to make the process more efficient.  In that respect, the project has thus far been a very encouraging success, and over the time I've spent working on it, the concept of a 2D fighting game in the style of Street Fighter or Guilty Gear made by a single person has gone from a lofty ambition to something that seems very reasonable to achieve.

In light of that, I'd like to review some of the important design decisions that led me to this place of confidence.

## Architecture

<!-- TODO: review how I made an architecture that easily supports rollback netcode -->

Conventional game engine design makes few assumptions about the *kind* of state stored on game objects, and this creates many opportunities for confusion, especially in languages with a "loose" concept of data ownership.  After deliberating, I chose to write the core architecture of this game in Rust, and the precise concept of ownership inherent in Rust's design helped me craft a robust architecture that supported advanced technical game features, including rollback netcode.

A crucial distinction in the design of the engine is drawing a strong barrier between *game state* (for example, the positions and velocities of objects on the screen) and *static game data* (for example, the amount of damage that a fireball deals).  *Static game data* is information required to run the game, but that doesn't change during the runtime of the game.  As such, it should never be stored *with* game state--game state should only *refer* to it indirectly.  (E.g. "My current state is Fireball, and I can look up the Fireball in the static game data to know how much damage I should deal if I hit.")

In a language like C# or C++, this distinction is often slightly blurred by giving game state objects *pointers* to the relevant static game data that's used to dictate their behavior.  However, once this direct link is established, the set of operations you can perform on game state data decreases--serializing and deserializing game state data becomes more difficult, as does hot-reloading static game data.

In contrast, Rust's strict data ownership model encourages you to adapt around this, creating indirect rather than direct connections between game state and static game data.  Though this is often more difficult for newer game developers to understand, in the long run, this encourages a much more flexible game architecture which has enabled me to write many features to improve game performance and development speed, such as rollback netcode, hot reloading, in-game static game data editing, and more.

## Flexibility

<!-- Discuss how I wrote a "scriptable" system that could support arbitrary moves and properties and could easily be extended-->

Once static game data was separated so distinctly from game state, it opened up many possibilities for how to design the way that characters in the game are described in the static game data to allow the game developers maximal flexibility.  Instead of hardcoding various move properties and game logic, by implementing the ability to write simple predicates within the game data, I quickly achieved the ability to design a wide variety of moves with minimal changes to the underlying software.  This allowed me to rapidly prototype and implement the high volume of different attacks and abilities required for a single character in a fighting game, creating interesting and unique gamefeel in each one.


## Developer UX

<!-- TODO: Review how I set up a convenient UX very quickly with immediate mode UI for development of new moves -->

In order to maintain a tight prototyping loop between design, implementation, and testing, it was high-priority for me to have an easy and visual way to make changes to the static game data.  By using a mixture of advanced metaprogramming features and by leveraging the existing work put into immediate mode UI libraries by the talented open-source Rust gamedev community, I was able to very quickly prototype functional interfaces suitable for developer use in making adjustments to existing moves and even constructing entirely new ones.  As the sole developer and artist, I was able to use the interface on my own with great efficiency, but I was also able to explain the interface to others knowledgable in the fundamental concepts of fighting games such that they were also able to quickly achieve a degree of familiarity and understanding of it.

## Conclusions

Although there's a lot of work left to do, the core "skeleton" of the game is essentially complete, leaving a lot of room for creativity in game design that requires minimal refactoring of core systems.  As time allows, I hope to continue enjoying working on this as a hobby project in the future.