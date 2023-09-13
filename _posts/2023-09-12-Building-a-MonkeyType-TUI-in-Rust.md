---
title: "Building a MonkeyType TUI Application in Rust - Part 1"
date: "2023-09-12 17:00:00 +0800"
description: "First part of a Rust Programming Language tutorial series. Learn by building a Terminal UI application using ratatui crate and crossterm. Building a terminal clone of MonkeyType. Complete guide on using ratatui, crossterm, tui-rs"
categories: [Rust, TUI]
tags: [cli, tui-rs, rust, ratatui]
img_path: /assets/img/
keywords: [Rust, Programming in Rust, TUI Application, Rust by example, MonkeyType, Rust Project]
abstract: "A new series where I build a TUI clone of MonkeyType in Rust using the ratatui crate. In this part I go over some background, and a general overview of the project as well as some intial setup."
proficiency: "Beginner"
---
![banner image of the monkeytype website](blog-001-02.png)
_MonkeyType_

Welcome to a new series where I build a TUI clone of MonkeyType in Rust using the [ratatui](https://github.com/ratatui-org/ratatui) crate. In this part I go over some background, and a general overview of the project as well as some intial setup.

## Background
If you are here for the technical stuff, you can [skip to main content](#overview). Six months ago, I began this journey to learn Rust after seeing how blazingly fast it was and how one could flex their undeniable superiority by coding in Rust. Being a NeoVim user myself (I also use Arch btw), this was the obvious next thing to learn.

Half a year later, I finally have an understanding of the basics, having explored most of the features of the language. The goal behind this project is to get comfortable building a larger application and keeping the code somewhat idiomatic.

This is my second attempt at writing this application, but I wouldn't call my first attempt a failure. I was still very new to Rust when I tried it the first time, and I got pretty far with it too. But I was learning the language at a much faster rate than I was developing this project and all the suboptimal decisions/anti-patterns became pretty obvious to me, so I stopped working on it. There were also very few tutorials/guides at that time on using [tui-rs](https://github.com/fdehau/tui-rs), which was the original (now deprecated) library I was using. Fortunately [ratatui](https://github.com/ratatui-org/ratatui), the successor of tui-rs, has included starter templates and examples for newcomers. Since it is also a fork of tui-rs, the interface remains mostly the same, which means that all the time that I spent learning tui-rs was not in vain.

## Overview
[MonkeyType](https://monkeytype.com) is a popular website for practicing touch-typing, with a simple but visually pleasing look. It has multiple test modes, provides analysis and statistics about your typing, has great themes, and a lot more features.


### Features I will be implementing
These are all the features (as outlined on the MonkeyType github page) that I will attempt to implement in this series:

- Type what you see, see what you type
- Live errors, wpm and accuracy displays
- Variety of test lengths and test modes
- Performance stats and analysis
- (some) theming options

These features are at the core of MonkeyType. The ones that I omitted are either irrelevant to my goals or too much to implement.

### Goal
I will have to make some compromises because of the inherent limitations of a terminal environment, but the goal is to not compromise on the quality of the code. I will be uploading the code on GitHub and would like to continue working on the app in the future and try to actively maintain it.

As is with most things that interest me, the primary goal is to learn something new. I hope to get better at Rust as well as programming in general by the end of this project, while also helping any readers along the way.

Although ratatui has a [starter template](https://github.com/ratatui-org/rust-tui-template), I will start from scratch so that I have a good understanding of how it all works, using the template only as a guide for better organization. 

## Project Setup 

I'll start by creating a new Rust project

```bash
$ cargo new tui-type
$ cd tui-type
```

In my first attempt at creating this application, I followed the `model-view-controller` pattern, but I implemented it poorly. After looking at some other TUI projects, I found a better way of organizing the application. It still follows the MVC pattern but in a nicer way. The starter-template also does things this way so we will stick to it for the time being. 

Here is the file structure from the rust-tui-template. I might separate each module into its own directory later if things get out of hand.

```
src/
├── app.rs     -> holds the state and application logic
├── event.rs   -> handles the terminal events (key press, mouse click, resize, etc.)
├── handler.rs -> handles the key press events and updates the application
├── lib.rs     -> module definitions
├── main.rs    -> entry-point
├── ratatui.rs -> initializes/exits the terminal interface
└── ui.rs      -> renders the widgets / UI
```
Ratatui is based on the principle of `immediate rendering` with intermediate buffers. The basic idea is that the UI consists of configurable widgets. They can be composed to create layouts which are then rendered in the terminal. The UI is immediately rendered at every frame meaning it holds no state, and is rebuilt at each frame. This sounds expensive but according to their docs, the overhead usually comes from the terminal emulator itself and not rust. Rust is blazingly fast I guess.

The rendering itself is done by a `terminal backend`, [crossterm](https://docs.rs/crossterm/latest/crossterm/) in my case, which is responsible for manipulating the terminal. It does so by executing "commands, where a command is just an action you can perform on the terminal e.g. cursor movement." Ratatui provides an abstraction on top of such a `backend`, and comes with easy to use widgets and layouts out of the box. It does not provide input handling or an event system so I have to manage input events myself using the crossterm backend. I will also have to handle the event loop myself and manage multiple threads and messaging if need be.

## Hello World
Let's add `ratatui` and `crossterm` as a dependency in `cargo.toml`.
```toml
# cargo.toml
[...]
[dependencies]
ratatui = "0.23.0"
crossterm = "0.27.0"
```

In order to start receiving keypresses, the app needs to enable `raw mode` on the terminal which disables the default behavior of the terminal and allows us to control events and inputs. Ratatui provides the `Terminal` struct which is a generic struct over the trait `Backend`. `CrosstermBackend` implements this trait and is what I will be using. Let's start by creating a new struct `Ratatui` which will hold an instance of `Terminal` in the `ratatui` module.

```rust
// ratatui.rs
use ratatui::{prelude::Backend, Terminal};

pub struct Ratatui<B: Backend> {
    terminal: Terminal<B>
}
```
Next I will implement an `init()` function on the Ratatui struct which will enable `raw mode` and enter an `Alternate Screen` which is a bit like opening a new buffer on top of the `Main` terminal where our app will do its thing. An `exit()` function is also needed to reset all the properties, disable raw mode and leave the `Alternate Screen`.

```rust
// ratatui.rs
use std::io;
use crossterm::terminal::{self, EnterAlternateScreen, LeaveAlternateScreen};
[...]
impl<B: Backend> Ratatui<B> {
    // Constructor
    pub fn new(terminal: Terminal<B>) -> Self {
        Self { terminal }
    }

    // Initializes terminal, enables raw mode
    // Sets terminal properties
    pub fn init(&mut self) -> io::Result<()> {
        terminal::enable_raw_mode()?;
        crossterm::execute!(io::stdout(), EnterAlternateScreen)?;

        self.terminal.hide_cursor()?;
        self.terminal.clear()?;
        Ok(())
    }

    // Exits the terminal interface
    pub fn exit(&mut self) -> io::Result<()>{
        terminal::disable_raw_mode()?;
        crossterm::execute!(io::stdout(), LeaveAlternateScreen)?;

        self.terminal.show_cursor()?;
        Ok(())
    }
}
```
To render to the screen, we can use the `draw()` method on the `Terminal` instance which takes a rendering closure, passing the current `Frame` to it. `Frame` has the method `render_widget()` which takes an implementation of the `Widget` trait as an argument. All rendering in Ratatui is done through widgets. It provides many useful `widgets` out of the box with a convenient builder-pattern interface.

```rust
use ratatui::{
    prelude::{Alignment, Backend},
    style::{Color, Style},
    widgets::{Block, BorderType, Borders, Paragraph},
    Terminal,
};

[...]

impl<B: Backend> Ratatui<B> {
    [...]
    // Draws widgets on the terminal
    pub fn draw(&mut self) -> io::Result<()> {
        self.terminal.draw(|frame| {
            frame.render_widget(
                Paragraph::new(format!("Hello world"))
                    .block(
                        Block::default()
                            .title("Template")
                            .title_alignment(Alignment::Center)
                            .border_style(Style::default().fg(Color::White))
                            .borders(Borders::ALL)
                            .border_type(BorderType::Thick),
                    )
                    .style(Style::default().fg(Color::Cyan).bg(Color::Black).bold())
                    .alignment(Alignment::Left),
                frame.size(),
            );
        })?;
        Ok(())
    }

}
```
I will explore widgets in detail in the future. For now, the closure calls `render_widget()` on the `frame` and renders a `Paragraph` widget which has some styling and `block` properties.

It's time to run the app! Initialise the app in our main function like so
```rust
use std::{io, thread, time::Duration};

use ratatui::{prelude::CrosstermBackend, Terminal};
use tui_type::ratatui::Ratatui;

fn main() -> io::Result<()> {
    // Initialize the terminal user interface.
    let backend = CrosstermBackend::new(io::stdout());
    let terminal = Terminal::new(backend)?;
    let mut ratatui = Ratatui::new(terminal);
    ratatui.init()?;

    ratatui.draw()?;
    thread::sleep(Duration::from_millis(5000));

    // Exit the user interface.
    ratatui.exit()?;
    Ok(())
}
```
Since we have no way of accepting input yet, there is no way to exit (^c does not work as we are in `raw mode`), so instead the main function calls `exit()` 5 seconds after starting the app.

```bash
$ cargo run 
```
![terminal output of "Hello World"](blog-001-01.png)
_Output_

That's a lot of work for a hello world program. It doesn't look like much but we have created a great starting point.

## Conclusion
In this first part of the series, I explored setting up a **TUI application** using [ratatui](https://github.com/ratatui-org/ratatui) and [crossterm](https://docs.rs/crossterm/latest/crossterm/). I was able to setup and teardown a terminal, entering and leaving raw-mode. I was also able to render widgets to the terminal which sets up a solid starting point for the app. There is still a long way to develop, and there are already a few things that can be improved about the current version of the code.

For example, what happens if the program panics before resetting the terminal? The terminal would get stuck in raw-mode, the cursor might still be hidden and so on. We will address these concerns in the next part.

The next part of the series will cover **events, event handling** and **inputs**.

If you enjoyed reading or found this post helpful, consider sharing it! I am very new to blogging so I appreciate any feedback and support :D
