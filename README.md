# Application Patterns

This repository describes a set of useful [patterns](#pattern) for building software [applications](#application). It contains descriptions in natural language along with some examples in actual or pseudo code. 

## Index <small>(repository)</small>
- [Overview (this document)](#overview)
- Concepts
  - [Applications Software](./concepts/applications-software.md)
- Data 
  - [Adapter Pattern](./data/adapter-pattern.md)
- Security
  - [Role Based Security](./security/role-based-security.md)

## License
The content in this repository is licensed under the [Creative Commons Attribution-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-sa/4.0/legalcode). As summarized in the [Commons Deed](https://creativecommons.org/licenses/by-sa/4.0/) and strictly defined in the [license](./LICENSE.md) itself: you may share or adapt this work, provided you give appropriate credit, provide a link to the license, and indicate if changes were made. In addition, you must preserve the original license. For the details and actual legal license, consult the [license](./LICENSE.md).

### Source Code
The [license](#license) of this work is designed for cultural works, [not source code](https://creativecommons.org/faq/#can-i-apply-a-creative-commons-license-to-software). Any source code intended for more than an illustration should be placed in a separate repository with an appropriate license. 

# Overview 

## Contents <small>(this document)</small>
- [Audience and purpose](#audience-and-purpose)
- [Terminology](#terminology)
- [Philosophy](#philosophy)
  - [Utility](#utility)
  - [Clear mechanics](#clear-mechanics)
  - [Mastery of fundamentals](#mastery-of-fundamentals)
  - [Building vs inventing](#building-is-not-inventing)
  - [Sufficient complexity](#sufficient-complexity)
  - [Modularity and restraint](#modularity-and-restraint)

## Audience and Purpose
This work is aimed at **professional software application engineers**: people who spend a substantial portion of their working days building software services and interfaces that other humans use to do work. See [terminology](#terminology) for detail.

## Terminology

### Application
*For a longer treatment of this topic, see [Applications Software](./concepts/applications-software.md)*

My general qualification for distinguishing software *applications* from other kinds of software is that applications software is generally aimed at helping **humans** do **work**. Often humans are represented directly in the form of `User` objects, or the software enables humans to interactively manipulate data. In other cases, humans may not be directly represented or even interacting with the software, but the software is still aimed at accomplishing a practical goal for humans: moving money, dispatching vehicles.

Like physical machines, applications are expected to **produce value** when they operate. Motors are incredibly useful, but pointless by themselves. They must be attached to some other thing -- even if just a simple drill bit -- to become a useful machine. Likewise building software applications generally involves selecting, assembling, and applying various mechanical software components (data stores, executing environments, shared caches, HTTP interfaces) to accomplish some practical goal.

We can clarify by identifying a few things that are not applications for our purposes:

- Systems software. An operating system kernel or device driver is mostly concerned with coordinating and controlling actions of hardware and other software. 
- Storage software. A database server or file system itself addresses the fundamental goal of serializing, structuring, and retrieving data. 
- Games. Some games do share many concerns with applications, but the goals of entertainment or art are very different than economic utility 

Obviously, "application" is not some Platonic form, perfect and mutually exclusive with all other kinds of software. The point is not to draw exact boundaries, but to aim in the right general direction.

### Pattern
A pattern in the scope of this work is a conceptual way of accomplishing a useful task. 
  - When constructing buildings, [log framing](https://en.wikipedia.org/wiki/Log_house), [balloon framing](https://en.wikipedia.org/wiki/Framing_(construction)#Balloon_framing), [pole framing](https://en.wikipedia.org/wiki/Pole_building_framing), and [stonemasonry](https://en.wikipedia.org/wiki/Stonemasonry) are all different conceptual ways of accomplishing the task of "make the basic structure of a building". 
  - When cooking, conceptual ways of "cooking rice" include steaming in a pan, using a bamboo steamer, soaking followed by boiling and passive steaming, or absorbing while stirring as in risotto and paella.

# Philosophy

Due to its focus on **professionals** engaged in building **applications software**, this work is written with certain value judgments and expectations:

## Utility
The goal of the software we create is to create value. Value is eroded when software cannot be understood, maintained, or modified over time. The following goals are all aimed at building software that produces more value than it consumes.

## Clear mechanics
We are ultimately responsible for the the function of the digital machines we create. This is not possible without understanding how those machines actually work -- their mechanics. We strongly discourage tools, patterns, and even languages that introduce mechanical complexity and obscurity in the name of automation, short-term productivity, or self-indulgent cleverness. We are professionals with both the time and the obligation to understand the things we build. The learning cost of understanding the fundamentals is dwarfed by the ever-compounding costs of *not* understanding. 

## Mastery of fundamentals
We believe professional software engineers should have a solid understanding of the fundamentals of our trade. This includes substantial understanding of the purpose, mechanics, and tools relating to:

- **File systems** 
- **Relational databases**
- **HTTP** 
- **Serialization** 
- **Cryptography**
- **Garbage collected runtimes** 
- **Statically typed languages** 
- **Scripting languages** 

And this list does not touch the other critical concerns of a software engineer, including **package management**, **testing**, and **deployment**. The scope of this work however is on design and construction itself.

## Building is not inventing
Understanding and implementing patterns is not reinventing. Builders build walls. They don't operate wall-building machines, but neither are they "reinventing walls" every time they build one. They are using their human brains to perpetually adapt the patterns used to build walls to each individual project. Electricians run wires, they don't operate electrical-installation machines. And they are not "reinventing electrical work" when they design the circuits and transformers of a building. They are using their human brains to perpetually adapt the patterns of planning and installing wires, outlets, and associated hardware to each individual project.

## Sufficient complexity
Tools, languages, patterns, abstractions...complexity and "managing" it is much of the substance of our work, as it has been for people building physical machines going back thousands of years. It is important to note that when the domain to model, the problem to solve, is itself irreducibly complex, as is generally the case with applications software, then good tools used to solve these problems also have a requisite complexity. Methods which try to force simplicity beyond the fundamental shape of the problem domain fail, because they simply displace that complexity. 

Trying to communicate with a vocabulary of 100 words, or no verb conjugation, is not in fact easier than with a full vocabulary of tens of thousands of words and sufficient grammatical complexity. The unique concepts we want to communicate about in natural language are so vast and varied -- and with many meaningful relationships between them -- that all natural languages evolve to acquire sufficient complexity to effectively handle this richness.

As an extreme counterexample, all known biology is described using a simple "language", in the form of linear strings of four nucleobase "symbols". The *emergent* mechanics that result from this are so mindbogglingly complex that I doubt the human species (or its supposed digital successors) will ever achieve a full analytical grasp of it all. A designed language for describing chemical systems that humans could actually cope with would require a far more complex syntax to achieve mechanics that are succinct and semantically clear.

And so, to build software applications, one must understand and use the requisite complexity of our available tools. All relational databases implement a wide but common set of concepts beyond "tables". Triggers, constraints, indexes, views, and cascades exist because they represent endemic mechanics and complexity in structure-validated information. Compiled object-oriented languages include "advanced" features like generics, interfaces, typed delegates, and overloadable operators because they too model fundamental and useful mechanics for representing and processing information. Trying to plaster over this complexity with magic and artificial simplicity is a fools errand.

## Modularity and restraint
A fundamental technique for maintaining clarity of mechanics is to draw boundaries across which knowledge may *not* flow. This often involves making actual choices to omit functionality, introduce abstraction, or add actual latency due to introducing communication channels between disconnected components. Abstraction is its own form of complexity, hence these are difficult professional choices that require understanding of the fundamentals and experience to understand viscerally the potential human costs of a given option.

As far as we can achieve modularity in practice, it also allows us to describe and use patterns such as the ones described here. Constraining a diesel engine's "interface" with the world to fuel and air in, exhaust and a single rotating shaft out allows the mix-and-match versatility of a modern tractor, or the entire business model of Cummins.

##### Attribution

Author: Joshua Honig. Copyright 2019 Mallowfields LLC.