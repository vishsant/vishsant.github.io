+++
date = "2026-03-25"
draft = false
title = "PagedAttention: How LLM Inference Just Reinvented Unix’s Virtual Memory for GPUs"
+++

A single conversation with an LLM can eat gigabytes of GPU memory. Not for math, not for model weights. Just for remembering what it already said.

In 1962, a computer at the University of Manchester solved a similar problem. The machine was called **Atlas**, and the fix it introduced was **virtual memory**. Before Atlas, programmers moved data between fast memory and slow storage by hand. Programs couldn't exceed physical memory, and the process broke often. **Atlas added an indirection layer**: fixed-size pages, page tables, and on-demand movement between memory tiers. The programmer saw one large memory. The system handled the rest.

That idea stuck. Multics adopted it. Unix made it practical. Linux made it universal. For sixty years, every mainstream OS has relied on it. Then, in 2023, the same idea had to come back - not for CPUs, but for LLM inference on GPUs.

## The Memory Nobody Sees

When you send a message to an LLM, something called the **KV cache** starts growing in GPU memory. Each token the model processes creates two vectors: **a Key (how to find this token later)** and a **Value (what this token contributes)**. The model stores these so it can look back at earlier tokens without recomputing them from scratch, which would be far too expensive. Think of it as the model's working set, similar to a process's resident memory in Linux, except it only ever grows.

Here's a concrete example.

You open ChatGPT and type: "*I'm learning Linux memory management. Can you explain virtual memory?*" The model reads your sentence token by token. For each token, it stores a Key and a Value. The KV cache now holds meaning for "learning," "Linux," "memory," "virtual memory," and "explain."

Now you ask a follow-up: "*How does that relate to what GPUs do?*" The model doesn't reread your first message. Instead, it builds a Query from your new question, compares it against all stored Keys, finds the best matches, and pulls their Values. When it sees "similar to what GPUs do," it matches against Keys like "Linux" and "virtual memory," retrieves their context, and uses that to build the answer.

So **what does the cache actually hold**? Not sentences. **It holds how to find meaning (Keys) and the meaning itself (Values).** This memory **never shrinks** during output. It **only grows.** Thousands of tokens across many layers and many sequences add up to tens of gigabytes. On a GPU, memory becomes the bottleneck - not compute, not bandwidth.

## The Waste Problem

Uncertainty drives the waste. When a request starts, the system doesn't know the response length. Twenty tokens? Two thousand? Most systems assume the worst case and grab one large contiguous block per request.

This creates three familiar problems. **Internal fragmentation**: a 2,048-token block used for 200 tokens leaves 90% empty. **Reservation waste**: future tokens claim memory before they exist. **External fragmentation**: free memory scatters across the GPU, and even though enough total space exists, none of it sits in one usable block. If you've used malloc or heap allocators, this should feel familiar.

In real workloads, only 20-40% of allocated KV memory does useful work. The other 60-80% goes to waste. This is the exact problem operating systems faced before virtual memory.

## The Manchester Insight

In the late 1950s, Atlas solved this with one idea: drop the requirement for contiguous physical memory. Divide memory into fixed-size pages, map virtual addresses to physical ones via a table, and allocate pages only when needed. This killed fragmentation. Programs stopped caring about physical layout.

The idea spread to Multics, Unix, Linux, and Windows. For decades, everyone treated this problem as solved. Then GPUs took over compute, and the problem returned.

## Pages for Tokens

In 2023, UC Berkeley researchers introduced PagedAttention. They published it at **SOSP**, the top operating systems conference - not ICML, not NeurIPS - because this is a systems problem.

The **core idea: stop giving each sequence one big block. Break memory into small, fixed-size blocks instead. **Allocate them on demand, allow them to sit apart in physical memory, and pull them from a shared pool. Each sequence keeps a small map from logical position to physical block. Strip away the names and this is virtual memory: blocks are pages, the block table is a page table, and on-demand allocation is demand paging.

**PagedAttention also supports copy-on-write**. When many sequences share a prefix, like in beam search, they share memory and only split when one writes new data. **Same trick as fork() in Unix.**

The results: memory waste dropped from 60-80% to under 4%, throughput jumped 2-4x, and GPU costs fell hard. The fix didn't come from new research. It came from ideas that had been sitting in OS textbooks for sixty years.

## The Missing Hardware

A fair question: *why didn't GPU hardware handle this already?* CPUs solved it decades ago with MMUs, TLBs, page faults, and demand paging. GPUs didn't. Modern GPUs have a memory management unit (GMMU), but it can't do fine-grained allocation, per-request dynamic memory, or sub-millisecond fault handling. A GPU page fault takes hundreds of microseconds, which is far too slow for per-token work.

So PagedAttention runs it all in software. The block table acts as a software page table, and the kernel reads it directly. No faults, no OS calls, no latency spikes. The irony: the processor running the most memory-hungry workloads in computing still lacks the memory abstractions that CPUs gained decades ago.

## The Hardware Catches Up

New chips are closing the gap. **NVIDIA's Grace Hopper (GH200)** brings coherent CPU-GPU memory with shared page tables, faster address translation through ATS, and memory tiering across HBM and system RAM. On the software side, systems like **vAttention from Microsoft** Research push paging into the GPU memory API, splitting virtual and physical memory at the driver level.

The path forward looks the same as before: **hardware will absorb what software built first. **Atlas led to hardware MMUs, which led to modern CPUs. The same arc is playing out for GPUs now. Sixty years later, the problems haven't changed. Only the hardware has.
