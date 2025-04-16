# Day 4 – Finishing the Guessing Game in Rust 🦀

**Date:** [Insert Date Here]

## ✅ What We Completed

We finished our first Rust CLI project: a **Guessing Game** 🎉  
Learned how to handle:
- User input
- Loops and conditional exit
- Pattern matching (`match`)
- String parsing & input sanitization
- Using `Ordering` enum from `std::cmp`

## 🔁 Final Game Logic

- Random number generated (1–100)
- Player guesses the number
- Game gives feedback (greater/less/equal)
- Player can type `stop` or `end` to exit anytime

## 🧪 Final Code

```rust
use rand::prelude::*;
use std::{cmp::Ordering, io};

// Guessing Game
fn main() {
    println!("Welcome to Guessing Game!");

    // Step 1: Computer guess a random number and
    // Step 2: stores in memory
    let mut rng = rand::rng();
    let computer_rand_number: i32 = rng.random_range(1..=100);
    // println!("{computer_rand_number}");

    println!("Guess a Number:");

    loop {
        // Step 3: ask the user to guess the number stored in the memory
        let mut guess = String::new();
        // Step 4: Users inputs number
        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to get input");

        // Step 4a: End Guess if user enters stop
        if guess.to_lowercase().contains("stop") || guess.to_lowercase().contains("end") {
            println!("Closing .....");
            break;
        }

        // Step 5a: Convert user input from string to integer
        let parsed_guess: i32 = guess.trim().parse().expect("Please input a real number!");

        // Step 6: Compare and respond
        let cmp_guess = parsed_guess.cmp(&computer_rand_number);
        match cmp_guess {
            Ordering::Less => println!("Your Guess is less! Guess Again"),
            Ordering::Greater => println!("Your Guess is greater! Guess Again"),
            Ordering::Equal => {
                println!("Your Guess is correct.");
                break;
            }
        }
    }
}

```

## 🧠 Key Rust Learnings

- `loop` + `break` for custom termination logic
- `.trim().parse::<i32>()` for safe input parsing
- Using match statements for elegant control flow

---

📌 Another mini project complete. Grateful to be learning Rust the practical way.