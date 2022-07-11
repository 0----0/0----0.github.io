---
title: Illusory Friends -- In Review
tags: TeXt
---

Illusory Friends is a game prototype I made during the 2021 Rusty Jam.  I set my goals a little high for the time I had available and spent most of my time yak shaving, but it was fun anyway, and I got to play around with some cool game design concepts and some very interesting technical concepts on the back end.  With that in mind, I'd like to discuss a few fun little bits of technique that I applied to the prototype.

To view the prototype online in your browser or download the source code, click [here](https://brackets.itch.io/illusory-friends).

<!--more-->

## Dialogue Trees as Asynchronous Functions

```rust
async fn ghost_customize_player_class(game: Game) -> anyhow::Result<()> {
    let m = Some((Portrait::Maribelle, PortraitOrientation::Right));
    let g = Some((Portrait::Ghost, PortraitOrientation::Left));
    let player_class_id = game
        .show_choice(["A WITCH", "A PRINCESS", "A KNIGHT"])
        .await?;
    let player_class = match player_class_id {
        0 => PlayerClass::Witch,
        1 => PlayerClass::Princess,
        _ => PlayerClass::Knight,
    };
    match player_class {
        PlayerClass::Witch => {
            game.show_portrait(m);
            game.show_text("I AM THE GREAT WITCH, MARIBELLE.\nYOU ARE A SERVANT I HAVE CONJURED.")
                .await?;
            game.show_portrait(g);
            game.show_text("WOW! YOU CREATED ME?\nYOUR MAGIC IS REALLY POWERFUL!")
                .await?;
        }
        PlayerClass::Princess => {
            game.show_portrait(m);
            game.show_text("I AM THE CROWN PRINCESS, MARIBELLE.\nYOU ARE MY LOYAL SUBJECT.")
                .await?;
            game.show_portrait(g);
            game.show_text("THE PRINCESS? WHAT AN HONOR!\nYOUR WISH IS MY COMMAND, HIGHNESS!")
                .await?;
        }
        PlayerClass::Knight => {
            game.show_portrait(m);
            game.show_text("I AM THE QUESTING KNIGHT, MARIBELLE.\nWOULD YOU LIKE TO BE MY SQUIRE?")
                .await?;
            game.show_portrait(g);
            game.show_text("OF COURSE!\nI ALWAYS WANTED TO GO QUESTING!")
                .await?;
        }
    }
    game.end_dialogue();
    game.0.borrow_mut().info.player_class = Some(player_class);
    Ok(())
}
```

Encoding dialogue scripts in video games often poses a unique challenge to game developers.  Dialogue in video games often does not follow a linear route where a game character says A, then B, then C, etc, but may involve a complicated chain of logic and conditions, request player input, or read and write gamestate in realtime.  Intuitively, the best tool for describing this chain of logic and state updates is just what you'd imagine: Code.  However, writing dialogue trees in the same language as the game itself is written in poses several challenges, the primary one being that most languages are specialized towards synchronous execution of code, while dialogue in video games must execute in discrete "steps", waiting for the user to press a button in order to advance the dialogue.

Most games achieve the goals of flexible and expressive dialogue systems that resemble code + asynchronicity by using a Domain-Specific Language/Scripting Language expressly for the purpose of describing the flow of dialogue.  However, for this project, I decided to try writing the dialogue in pure Rust, using Rust's own asynchronous capabilities to enable the rust code to function asynchronously.  While reducing iteration times compared to a scripting language, this came with the advantage of easy, direct read/write of various aspects of game state from within dialogue.

This solution uses a `LocalPool` for asynchronicity, since these are lightweight game design abstractions and have no need for concurrency.  The dialogue task is run after the main game update with `run_until_stalled()`, meaning that no work is done if the task is still waiting for user input to proceed.  The methods that the dialogue tree task uses to communicate with the main game object are lightweight and return a "one-shot" channel back to the dialogue task, which serves as a way for the game to communicate to the dialogue task when it's ready to progress.  If the dialogue task is waiting for a choice, then it doubles as a way of sending the player's user input to the dialogue task.

The dialogue task itself contains only the logical code for deciding what dialogue and portraits to show on the screen next, with the main game update handling the complex and state-dependent logic of animating the text, processing user input, showing cohices, and etc.  As a result, the dialogue code is remarkably lightweight, and holds a minimum of additional complexity over a comparable scripting language solution.

## Entity Component Systems

In modern game engines used and created by professionals and hobbyists alike, the ECS model has become a predominant model for describing the structure of game state.  However, while the fundamentals of *what it is* are approachable and capture the imagination of many developers, *how to use it* to take full advantage of its strengths and promote good code structure has mystified those who approach it unprepared.

As someone who's been exposed to the ECS model often in interactions with the Rust game development community, as well as in experimenting with the Unity and Unreal game engines, both of which use variations on this model, I decided to take my first crack at using this model in a real project without a pre-made game engine.  The challenge of designing the fundamental rules of how basic game objects would be represented in the ECS model proved very enlightening for understanding the advantages and proper usage of ECS.

```rust
struct PositionComponent(Vec2);

struct SpriteComponent {
    texture: TextureId,
    source: Option<Rect>,
    offset: Vec2,
    centered: bool,
    flip_h: bool,
    layer: i32,
}

struct AnimationComponent {
    id: AnimatedSpriteId,
    animation: Ustr,
    frame: usize,
    offset: Vec2,
}
```

The core concept that most people understand about the Entity Component Systems model is that it involves splitting an individual game object (an Entity) into many Components, each representing a certain aspect of its state.  However, it's my belief that many people fall into the subtle misunderstanding that the purpose of this is strictly to minimize the amount of state stored, so that objects only store the state they are using.  This can be one benefit of an ECS, but in my opinion, it is not the priority and should never be treated as such.

Rather, in my opinion, the primary function of an ECS is to *homogenize* your object behavior by making your Compoments *generic*, so that your Systems can, as much as possible, operate with minimal knowledge of an object's components.

As an example here, consider the `SpriteComponent`.  The `SpriteComponent` can be considered the "master" component of rendering.  When paired with the `Position` component, it contains all the information required to render a sprite object onto the screen at a given position at a given moment.  It can be used for both static and dynamic objects.

Many objects may not use all of the information stored within the `SpriteComponent`--for instance, many objects are never flipped.  Many objects do not need the "source" field, which is used for identifying a subsection of a spritesheet to render.

However, *if* that information *is* present, then the sprite rendering system *needs* to know that information in order to render the sprite on the screen.  Therefore, the `SpriteComponent` represents the smallest set of information that *may be required* for the sprite rendering system to do its job.

Furthermore, giving the `SpriteComponent` so much power actually makes *other* components simpler and easier.  For example, consider the `AnimationComponent`.  Because the `SpriteComponent` has the power to express "render this slice of this spritesheet", the `AnimationComponent` actually isn't needed for the sprite rendering system at all.  Instead, the `AnimationComponent` can do its job solely by *updating* the `SpriteComponent` with new information each frame, meaning that all of the "render a sprite" code is centralized in one System that only needs to read from `SpriteComponent`.  This keeps the architecture trim and tidy, reduces code duplication, and makes rendering easy to refactor or add functionality to.

## Conclusions

Although the code is rough around the edges due to the game jam timeframe, the solutions to architectural and technical problems that I found remain viable, and there's a pretty good chance I'll extract those for use in further projects down the line.  Overall I think the experiment was a success.