# Day 3 – Intro to Rust with a Guessing Game

**Date:** [Insert Date Here]

## 🚀 What We Did

- Kicked off our journey into Rust programming!
- Started building a simple **Guessing Game** using the official Rust book:
  👉 [Rust Guessing Game Tutorial](https://doc.rust-lang.org/book/ch02-00-guessing-game-tutorial.html)

## 🧱 Concepts Introduced

- Rust syntax and structure
- Using external crates (like `rand`)
- Reading user input with `std::io`
- Mutability and variable bindings
- Random number generation
- Basic logic and flow control in Rust

## 🧪 Code So Far

```rust
use rand::prelude::*;
use std::io;

// Guessing Game
fn main() {
    println!("Welcome to Guessing Game!");

    // Step 1: Computer guess a random number and
    // Step 2: stores in memory
    let mut rng = rand::rng();
    let computer_rand_number: i32 = rng.random_range(1..=100);
    println!("{computer_rand_number}");

    // Step 3: ask the user to guess the number stored in the memory
    let mut guess = String::new();
    io::stdin()
        .read_line(&mut guess)
        .expect("Failed to read line");

    // Step 4: Users inputs number
    // Step 5: Computer compares users input wih computer's initial guess
    // Step 6: If number is correct inform user wins, else inform user if guess lower or higher
    // Step 7: Start from step 1
}
```

> 🔧 Note: This is a work in progress. We'll continue refining the logic and completing the game in the next class.

## 🗣️ Build In Public

Tweet/thread documenting the journey 👉 [Insert link once posted]

---

✅ Progress logged, excited for next session!