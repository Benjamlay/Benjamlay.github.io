---
layout: post
title:  "Building a Seamless Multiplayer Experience: MVC and the Magic of Rollback"
date: 2026-05-01 08:00:00 +0100
categories: jekyll update
---

![first image](/images/rollbackGamePicture.png)

# Introduction

As part of my second year of studies, I had the opportunity to dive deep into network programming and multiplayer game development. The biggest lesson I learned? When it comes to fast-paced online games, network latency is the ultimate enemy. 

While traditional delay-based netcode forces the game to wait for everyone's inputs—resulting in a sluggish, heavy feel—I wanted to tackle something far more ambitious for my project: **Rollback Netcode**.

Rollback predicts player inputs locally to provide zero-latency responsiveness, and invisibly rewinds time to correct any network discrepancies. But as I quickly discovered, you can't just slap a rollback system onto a standard game loop. It requires an incredibly disciplined approach to your codebase. Here is how I structured the game's architecture to make rollback a reality.

---

## Part 1: The Foundation - Why Strict MVC is Non-Negotiable

To understand rollback, you must understand its golden rule: **Determinism**. If you feed a game state the exact same inputs for a specific frame, it must produce the exact same output on every player's machine, regardless of their hardware or framerate. 

To achieve this, I built the game entirely around a strict **Model-View-Controller (MVC)** architecture. If you break this separation, rollback breaks with it.

### The Model: The Absolute Truth
The Model is the pure, mathematical simulation of the game. It knows nothing about graphics, pixels, or screen refresh rates. 
* **Fixed-Point Math:** Floating-point numbers are calculated slightly differently depending on the CPU. To guarantee determinism, the `SimpleModel` relies entirely on fixed-point math and custom trigonometric look-up tables for movement and collisions[cite: 1].
* **Easily Cloneable:** The Model contains the absolute minimum required data: positions, rotations, velocities, hit timers, and cooldowns[cite: 1]. Because it is purely data-driven, the entire game state can be instantly copied and overwritten (rolled back) without memory leaks or heavy processing[cite: 1].

### The View: The "Dumb" Renderer
The View's only job is to look at the Model and draw it. It is strictly read-only.
* **Visual Independence:** This separation is crucial for visual effects. For instance, in our `SimpleView`, the fading tracks left by a tank or the alpha transparency of an explosion are calculated locally using the standard visual delta-time[cite: 4]. Because these visual flourishes don't affect gameplay, the Model doesn't need to simulate or rewind them[cite: 4]. If visual logic leaked into the Model, rewinding the game would cause chaos.

### The Controller: The Conductor
The Controller sits between the network, the local inputs, the Model, and the View. It is the brain that orchestrates the simulation, deciding when to predict forward and when to rewind time[cite: 3].

---

## Part 2: The Engine - Rollback, Confirmed Frames, and Checksums

With a strictly deterministic MVC foundation in place, we can actually implement the rollback logic. Our `SimpleController` handles a complex dance between the past and the present.

### Local Prediction
When you press a button to shoot or move, you don't want to wait for the server to give you permission. The Controller immediately applies your input to your local Model for the current frame[cite: 3]. This is why rollback feels just like playing locally. 

### The "Confirmed Frame"
Because of ping, inputs from your opponents will always arrive late[cite: 3]. This introduces the concept of the **Confirmed Frame**. 
The Confirmed Frame is the most recent frame in the past where your client has successfully received the inputs from *all* players in the match[cite: 3]. Everything before this frame is immutable reality; everything after it is a prediction.

### Rewind and Resimulate
When an opponent's delayed input arrives over the network via our Photon client, the Controller realizes its prediction was wrong[cite: 2, 3]. Here is the rollback sequence that happens in a fraction of a millisecond:
1. **Flag as Dirty:** The input manager detects a change in the past[cite: 3].
2. **Rewind:** The Controller instantly loads the game state from the last Confirmed Frame[cite: 3].
3. **Resimulate:** The Controller fast-forwards the simulation back to the present frame, applying the newly received inputs along the way[cite: 3]. 

Because the Model is lightweight and strictly mathematical, this entire process happens invisibly between two rendered frames.

### Checksums: The Ultimate Safety Net
Even with fixed-point math, true determinism is hard. A single variable initialized incorrectly can cause a "desync," where Player 1 and Player 2 are playing two completely different realities.

To prevent silent failures, we use **Checksums**. 
Every time a frame becomes "Confirmed," our Model calculates an Adler-32 checksum—a digital fingerprint—of the entire game state (combining all player coordinates, rotations, and projectile data into a single number)[cite: 1]. 

Here is how the hash is built in our Model:

```cpp
common::Checksum<1> SimpleModelManager::checksums() const {
  common::Adler32 adler;
  
  // Hash every gameplay-relevant variable to ensure perfect sync
  for (const auto& m : models_) {
    adler.Add(m.position.x.value); // Fixed-point deterministic value
    adler.Add(m.position.y.value);
    adler.Add(m.rotation);
    adler.Add(m.turret_rotation);
    adler.Add(m.lives);
    adler.Add(m.is_dead);
    adler.Add(m.shoot_cooldown);
    adler.Add(m.hit_timer);
  }
  
  for (const auto& p : projectiles_) {
    adler.Add(p.position.x.value);
    adler.Add(p.position.y.value);
    adler.Add(p.rotation);
    adler.Add(p.active);
    adler.Add(p.owner_id);
  }
  return {adler.value()};
}

```


This checksum is sent over the network[cite: 3]. The Controller constantly compares its local checksum for a specific frame against the remote checksum received from the opponent[cite: 3]. If they match, the simulation is perfect. If they differ, the Controller immediately logs a fatal desync warning, letting us know our determinism has a flaw[cite: 3].

***

Developing a rollback game during my studies was a massive architectural challenge. It forced me to rethink how time and state exist within code. But by sticking strictly to MVC principles and rigorously verifying state through checksums, it's incredibly rewarding to deliver a multiplayer experience that feels absolutely flawless.