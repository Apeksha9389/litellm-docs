---
slug: rust-express
title: "When will LiteLLM get fast? When the stations are Rust, not just the track"
date: 2026-10-20T09:00:00
authors:
  - nate
description: "A request in LiteLLM is a round trip with stops at auth, routing, caching, secrets, logging and spend tracking. Rust is only as fast as the slowest stop, so the speedup arrives when the stations move, not just the main line."
keywords: [litellm, rust, rust migration, ai gateway performance, litellm roadmap]
tags: [litellm, rust, rust-migration, roadmap]
---

The question we hear most about the Rust migration is "when will LiteLLM get fast?" The honest answer is that it gets fast when a request can make its whole trip on Rust, stops included. Rust track on the main line alone isn't enough, and that is the part that took us longer than we said.

{/* truncate */}

## Every request is a round trip

The easiest way to see why is to follow one request. Think of LiteLLM as a little railway map.

![Every request is a round trip](./img/panel-1.svg)

Your request leaves your app and passes the **gate**, where we check your key, budget and rate limits. The **router** decides which deployment it's headed for. Then it stops at the **cache** ("have we answered this before?"), the **vault** (to fetch the provider's API key from your secret manager), **customs** (to sign the request with AWS, Azure or Google Cloud credentials), the **post office**, where logs and your callbacks (Langfuse, Datadog, and so on) go out, and the **ledger**, where we record what it cost.

Notice where the line splits. The router picks a destination early and tags each wagon with it (OCR, Messages, Responses or Chat), but every request, whatever API it's for, passes every station before it heads down its own line, through the **HTTP tunnel**, to the model provider. So a fast line to the provider doesn't help much if the stations before it are slow.

Today almost all of that trip is a Python wagon on a dirt road.

## The test track

In June we published a benchmark showing a Rust gateway adding about **0.05 ms** per request, against about **7.5 ms** for the Python path. That number is real, and you can reproduce it.

![The test track](./img/panel-2.svg)

But that benchmark ran on a test track: a straight line from your app to a mock provider, with every station closed. Our July benchmark said so plainly: no logging callbacks, no spend tracking, no persistence. That is the right way to measure the cost of the train itself. It is not the trip your production requests take.

## Two kinds of station

Laying Rust track isn't enough on its own, because of what happens when a Rust train reaches a Python station.

![Two kinds of station](./img/panel-3.svg)

Python stations sit off the line, at the end of a little road. At a Python station the train has to stop. Every crate comes off the train and gets repacked into a box Python can handle, and a wagon hauls it up the dirt road to the station. Then it waits, comes back, gets repacked into crates and goes back on the train. That isn't just a cartoon: it's roughly what happens to your request's data each time Rust hands it to a Python service and takes it back.

A Rust station sits right on the track. The train rolls through it and the work happens on the cargo where it is, with no unloading.

## Where the track is today

Here is the map as of October 2026.

![Where the track is today](./img/panel-4.svg)

- **OCR** runs on Rust track, and it is Rust only since `v1.104.0-rc.1`. You don't have to do anything.
- **Messages** has Rust track for Anthropic that you can switch on with `LITELLM_RUST=1`.
- **Responses** and **Chat completions** are still Python roads.
- **Customs** (cloud credentials for AWS, Azure and Google Cloud) and the **HTTP tunnel** are finished Rust.
- **Logging** at the post office is Rust if you opt in.
- **Cache** and **vault** have Rust versions being built. Until they open, a Rust train that needs them stops and unloads onto a wagon.
- The **gate**, **router** and **ledger** are Python.

OCR shows what this means in practice. On one CPU, the Rust path handled **1.5x to 1.7x** more OCR requests per second than Python. That is a solid gain, but it is not the 150x from the test track, because the express still stops at Python stations along the way. Fast on the track, slow at the stops.

## What we got wrong in June

In June we said Messages and Chat completions would be on Rust by September 1, and the router by September 15. We missed those dates.

The plan then was to lay Rust track only on the API lines after the split: Rust would translate requests and responses, and Python would keep running every station. Once we started, we found that even the smallest stretch of track needs the stations. One provider's translation for one route still needs the cache, the secret manager, logging and callbacks. So we went back and started rebuilding the stations first: cloud credentials and the HTTP client are done, and cache, secrets and logging are underway. That is slower to show up in a benchmark, but it is the work that makes real requests fast.

## Next: finish the lines and open the stations

![Next: finish the lines and open the stations](./img/panel-5.svg)

Our target for the **end of 2026** is Rust track on the three big lines, **Messages, Responses and Chat completions**, with the stations they stop at (cache, secrets, logging, callbacks and spend tracking) open as Rust. That is when a typical request spends most of its trip on Rust. Our target for **every other API** (embeddings, audio transcription, MCP and the rest) is **April 30, 2027**.

The gate and the router come after the lines. We haven't set a new date for them yet, and we would rather say that than give you another date we can't stand behind. You can follow every line and station on the [migration tracker](/rust-migration), which updates with each release.

## The Rust Express

![The Rust Express](./img/panel-6.svg)

When every station is Rust, the whole trip runs at test-track speed, and that is the gateway the June benchmark was measuring. One thing stays Python on purpose: if you wrote your own callback in Python, the express still stops and unloads at your station, because it is your code.

How much you will notice depends on how you use LiteLLM. For a single chat turn, the model's response time dwarfs anything the gateway adds. The difference shows up when you send a lot of requests (embeddings, classification, guardrails), run agents that make dozens of calls per task, or pay for the CPU and memory your gateway pods use.

## What you can do today

- **Use OCR.** It is already on Rust, with nothing to change.
- **Try Anthropic Messages on Rust** with `LITELLM_RUST=1`. Python stays as the fallback, and `LITELLM_RUST=0` turns it off.
- **Watch the [tracker](/rust-migration)** to see each line and station move from Python to Rust.

Nothing in your config changes along the way. We lay the track under you, and each route moves from Python, to opt-in, to Rust by default, to Rust only.
