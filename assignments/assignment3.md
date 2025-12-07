Assignment 3
An AI-Augmented Concept
Concept Selection and Problem Statement

Concept Selected:
Team Scheduling (When2Meet-style tool with AI parsing).

This extends the Team Scheduling Conflicts concept from Assignment 2, which allowed teams to select availability blocks on a shared grid and specify a minimum number of participants required for a viable time.
The limitation in that version was that users had to manually click or type every slot, making coordination tedious when teammates simply described their availability in natural language.

Problem Statement

Coordinating schedules among teammates is error-prone and slow.
People rarely provide structured data; instead, they write things like:

> “I’m free after lunch on Thursday,”
> “Can only do Tues afternoon,” or
> “Anytime before 3 on Fridays.”

These messages require someone to interpret and translate them into concrete blocks before overlaps can be found.
Misinterpretations or omissions cause confusion and extra communication rounds.

The goal of this project is to augment the scheduling concept with an AI component that can interpret free-form natural-language availability and summarize conflicts, reducing the manual overhead while preserving user control.

One-Sentence Pitch

An AI-assisted team scheduler that reads natural-language availability messages, converts them into precise time blocks, and highlights the best overlapping windows for the group.
