# Two characters, from scratch

A course in nine steps. At the end, two NPCs talk to each other, remember each other, give
themselves a goal and act on it, with no network, no key and no C++.

You need the four plugins enabled and a model in place, as described in
[Getting started](README.md#getting-started).

The numbers quoted along the way were measured on one scene, in French, on an RTX 3060. Nothing in
the method depends on the language: the pack ships an English and a French profile, and
[Translating your NPCs](README.md#translating-your-npcs) lists every field that holds text if you
need a third.

---

## 1. How it works, in one page

The system is a loop with four beats. Understand this one and the rest follows.

**Perceive.** Everything that happens (a spoken line, a noise, someone walking in) is sent to the
*bus*, `NPC Perception`. The bus decides who heard it, from distances and hearing radii.

**Take the floor.** The bus lets only one character speak at a time. It gives the turn to whoever
the situation concerns most: someone spoke to them, someone named them, they are close by, they have
not spoken recently. This is what stops three NPCs from answering at once.

**Think.** Whoever has the floor gets a prompt and answers in three lines:

```
INTENTION: I lean down to the bowl to drink
NOTE: this water smells wrong
SAYS: Have you seen the state of this water?
```

Only the `SAYS` line is spoken out loud. `INTENTION` is what they are about to do, `NOTE` is what
they keep for themselves. Those three words are the defaults. They are fields, and step 4 shows
where to change them.

**Act.** The `INTENTION` goes to the *game master*, a second request to the model whose only job is
to turn it into verbs the body knows how to run. It decides nothing, it translates.

Alongside the loop there is a **reflection** now and then: the character gives itself a goal and
steps, which stay in its context until it changes them.

### The three layers of the prompt

This is the key to all the configuration:

| layer | where | what it says |
|---|---|---|
| **common** | `Profile > Reply Format > Common Prompt` | the rules of being a character, the three-line format |
| **personality** | `NPC Brain > Persona`, on each actor | who *this* character is |
| **situation** | built every turn | what they perceive, what they remember, what their body tells them |

The first and the third come from the **profile**, a Data Asset. The second is plain text on the
actor. Every sentence the model reads comes from one of the three, and all of it is yours to edit.

If you leave `Common Prompt` empty, the `System Prompt` of the model parameters is used instead.
That is handy while you are still moving text around, with one prompt for the whole scene held in
your GameMode. The profile is still the right home for it: it is what makes a second language a
second asset.

---

## 2. Setting up

**The model.** Any GGUF that llama.cpp can load. The demo's game mode, `BP_NPCDemoGameMode`, sets
the model parameters and loads the model when the game starts. Use it as the game mode of your own
map to begin with, or copy what it does into yours. `Path To Model` is the file; `Mmproj Path` is
only needed if you want the characters to *see*. Leave it empty and vision is off. Start without
vision: it is twice as fast and changes nothing else.

If you do turn vision on, look at `NPC Brain > Vision > Vision Resolution` before anything else. The
token cost follows the **area** of the picture, so doubling the side quadruples the price of every
reply. On our scene, 256 px put the prompt at 2,359 tokens and 128 px at 1,536, which is no more
than the same prompt with no picture at all. At 128 px the characters still make out who is in front
of them and what is lying on the ground. Raise it only for a model that has to read something
written in the world.

**Project settings.** `Project Settings > Plugins > NPC Perception`. Two fields are enough for now:

- `Default Profile`: the profile used by any NPC that does not bring its own;
- `Player Display Name`: what the NPCs call the player (`A stranger`).

---

## 3. The first character

Create a Blueprint deriving from **Character**, call it `BP_Villager`, and add two components:

- **NPC Brain**
- **NPC Agent**

That is all, and there is no node to place. The brain registers itself with the bus when the game
starts, waits for its turn, and the loop runs. An actor with those two components is a living
character.

On the pawn, set `Auto Possess AI` to `Placed in World`. Without it the character thinks and speaks
but cannot move. The log tells you so, but you may as well know now.

For it to walk you also need a **NavMeshBoundsVolume** in your level, set to **dynamic** generation.
With no navigation, the movement actions are removed from the catalogue and the model is never
offered them.

To hear what it says, use the **On Sentence** event of the `NPC Brain`. It fires for each finished
sentence. Wire it to a `Print String`, or to your speech synthesis.

---

## 4. The profile

This is the step that matters. Look in `NPCExamples/Languages` first: each folder there is a
language already done (`EN`, `FR`), with a profile and an action catalogue per species, ready to
assign. If yours is missing, duplicate `NPCExamples/Languages/EN/DA_Profile_Human_EN` into a folder
of its own and name it after the language (`DA_Profile_Human_IT`).

### The rule not to miss

The model answers with labels, and the plugin finds its answer through those labels. **Change a
label on one side and not the other, and the NPC thinks but never speaks. Nothing in the log says
why.**

The labels live in `Reply Format`:

| field | default |
|---|---|
| `Intention Label` | `INTENTION` |
| `Note Label` | `NOTE` |
| `Speech Label` | `SAYS` |
| `Silence Words` | `nothing`, `silence`, `-` |

The `Common Prompt`, in the same group, must use **exactly** those words:

```
You are a person who lives in this world, with a name, a body and a story of your own.
Everything you write comes from that person.

You answer in three lines, in this order, and nothing else:
INTENTION: what you are doing now, in one sentence.
NOTE: what you keep from this moment, for yourself only.
SAYS: what you say out loud. Write "nothing" if you would rather stay silent.
```

The French profile does the same thing with `DIT` and `rien`. The reflection follows the same rule:
`Reflection > Goal Label` is `GOAL`, `Steps Labels` is `STEPS`, and the `Instructions` have to use
those words.

### A creature that cannot talk

Animals go through the same loop with one field changed. Leave `Speech Label` **empty** and the
character never speaks. Give it a `Sound Label` instead and it makes a sound the others hear:
`SOUND: barks at the door` reaches the bus as a noise, not as words.

Give it its own profile too, not the human one with the labels swapped. Its `Common Prompt` is where
you tell it what kind of creature it is. Ours is 1,301 characters against 3,854 for the humans, and
the dog costs half as much per reply because of it.

### The prefill, the only thing that holds a format

`Reply Format > Prefill` holds the beginning of the answer, written in advance in the model's place.
Leave it **empty** and the plugin writes the first label followed by a colon, which is what you want
in almost every case. If you write your own, **never end it with a space**: a trailing space makes
the model write bare numbers.

An instruction gives way after fifty turns. A beginning of answer written in advance does not. That
is the difference between a format that holds for a minute and one that holds for an hour.

### Keep the common prompt short

Every character pays for the common prompt on **every reply**, and it is by far the biggest item.
On our scene it was two thirds of one character's whole prompt, more than the persona, the memories
and the picture put together. Ours went from 4,642 to 3,854 characters with no rule removed, only
repetition: bullets explaining what each line of perception already says by itself, and a permission
that was granted twice.

To price a block of text without running anything, count **about 3.5 characters per token**. That is
for French on a Qwen-class model. Measure it once on your own language and keep the number.

### The rest

Everything else in the profile is text to translate: the bus sentences ("You hear…", "X is in front
of you…"), the action reasons, the memory headers.
[Translating your NPCs](README.md#translating-your-npcs) goes through them field by field.

Keep every `{placeholder}` as it is (`{name}`, `{target}`, `{goal}`): the plugin fills them. Move
them around inside the sentence to suit your grammar, but do not rename them.

Last, point `Project Settings > Plugins > NPC Perception > Default Profile` at your profile.

---

## 5. Two characters, not two copies

Drop `BP_Villager` twice into your level. They share the profile, so the language, the format and
the way they remember. They differ in three things only.

**The name.** `NPC Agent > NPC Name`. It is the name the others know them by and call them by.

**The personality.** `NPC Brain > Persona`. Write it **in prose, and in the positive**:

> Odile, cook at the relay for eleven years. She feeds people and that is enough for her. She speaks
> plainly, without detour, and does not like people hanging around her kitchen.

> Garran, guard at the north gate. Gruff, sparing with words, he does you a favour when it suits him
> and says so when it does not.

**Never list what they must not do.** "You are not an assistant" produces "I cannot help you with
that". Name a limit and the model installs it. Describe who they are. And do not quote example
lines: they will be recited word for word.

**The appearance.** `NPC Agent > Appearance` describes **their own** look and nobody else's: what
they know of themselves from looking down. What the others look like comes through vision, if they
have it.

Now press Play and watch. They talk to each other, one at a time, and remember what was said.

---

## 6. Giving them something to talk about

A character whose only subject is hunger will talk about hunger. That is the cause of most of the
behaviour people take for a flaw in the model.

`NPC Brain > Add Variable` takes **any value** from your game (a number, a boolean, a string, an
array, a struct) and hands it to the character in your own words.

| parameter | what it is for |
|---|---|
| `Value` | the value. Wire a **variable of this Blueprint** and it stays live. Wire anything else and it is a copy, to be refreshed with `Update Variable`. |
| `Description` | what it means. May contain `{value}`. |
| `Steps` | turns a number into a sentence: *up to 60, "You are a little thirsty."* An empty sentence says nothing at all. |
| `Show To Game Master` | shows it to the game master too. Useful for what they are carrying. |

Example, in `BeginPlay`:

```
Add Variable  Value = Thirst (float variable of the Blueprint)
              Description = "Your thirst"
              Steps = [ 30 -> ""                                        ]
                      [ 60 -> "You are a little thirsty."               ]
                      [ 85 -> "You are thirsty, your mouth is dry."     ]
                      [100 -> "Your throat burns and your mouth is pasty." ]
```

**Describe the body, do not dictate the thought.** "Your throat burns" is a state. "All you can
think about is drinking" is an order, and the model obeys. In our measurements that one sentence
made the characters go for water 485 times against 15 for food, while they were starving.

Give them the world too. `Register Named Thing` on the bus names an object. It then shows up in
what they perceive, and becomes a possible target for their actions.

---

## 7. Giving them something to do

Create an `NPC Action Catalog` and assign it to the `NPC Agent`. **There is no built-in action**: an
empty catalogue is a character who perceives and speaks but never moves.

Six capabilities are provided, picked with `Native Handler`:

| Native Handler | what it does |
|---|---|
| `MoveToActor` | walks to someone, an object, or a place named for the first time |
| `Follow` | follows someone |
| `MoveAway` | backs off |
| `LookAt` | sets eyes on |
| `TurnToward` | turns towards |
| `Wait` | waits |

The `Verb` is yours, in your language: `ALLER_VERS` with `Native Handler = MoveToActor` works just
as well as `GO_TO`. Put the spellings the model drifts towards into `Aliases`.

A catalogue row looks like this:

```
Verb          GO_TO
Summary       walk up to someone and stop next to them
Args          Target (required)
Example       GO_TO followed by the name of a person or an object
Impl          Native
Native Handler MoveToActor
Timeout       15 s
Requires NavMesh  true
Trace Success you walked up to {target}
Trace Failure you tried to reach {target} and did not get there
```

Write both traces, and write the failure one as a **reason**, not as a copy of the success line.
"You lost sight of {target}" tells the character something. "You followed {target}" after a failure
tells them a lie they then have to justify out loud.

For your own gestures, set `Impl` to `Class` and point `Action Class` at a child of `NPC Action`
where you override `On Execute` and call `Finish`.

### Taking an action away

Deleting the row removes the verb for good. There are two gentler ways, for two different needs.

`Enabled` on a catalogue row, for a whole species: the row stays, with its summary, its example and
its two traces, but the verb disappears. Use it while iterating, so you do not have to delete text
you will want back.

`Set Action Enabled (Verb, false)` on the `NPC Agent`, for one character at a time: a broken arm, a
rope, a quest that unlocks a gesture, a place where one does not run. The verb leaves what that
character's game master is shown, so the model is not offered it and does not waste a turn being
refused. An action already running is left to finish.

To try it with nothing wired up, the console does the same while the game runs:
`NPCAgency.Action Garran LOOK_AT off`, and `on` to give it back.

Before writing an action of your own, read [Writing actions](README.md#writing-actions). There are
three traps in there, and one of them costs a week to find alone.

---

## 8. Checking that it works

Open the log on `LogNPCPerception` and `LogNPCAgency`. You should see this scrolling past:

```
Floor to Odile | Garran : Have you seen the water? / What you have in mind: ...
Odile sees | - Garran is in front of you, a few steps away. ...
Odile: plan requested for "I lean down to the bowl" (said: "...")
Odile: the game master wrote | 1. GO_TO a water bowl / 2. EAT_OR_DRINK a water bowl
Odile: plan received, 2 action(s), 0 line(s) rejected.
Odile: GO_TO -> succeeded (arrived)
Odile: plan finished (ok).
```

**The `Floor to` line is your best tool.** It is the exact text the character receives. Anything
wrong with their behaviour shows up there first: a sentence left in the wrong language, a need they
cannot satisfy, an absurd goal they have been dragging around for ten minutes.

A few symptoms and their cause:

| what you see | what it is |
|---|---|
| thinks but never speaks | `Speech Label` or `Silence Words` do not match the common prompt |
| `reflection without a readable goal` | `Goal Label` does not match the `Instructions` |
| many `line(s) rejected` | verbs or targets that were not recognised: look at `Aliases` |
| does not move at all | no AI controller, or no navmesh. The log says so when the game starts |
| repeats the same line | normal up to a point: a reply that repeats itself is cut |

One reading trap: **searching for `-> failed` finds nothing.** Failures are logged with their
reason: `no path`, `target gone`, `timed out`, `refused`.

---

## 9. The four traps that cost real time

**A need that takes two actions is almost never satisfied.** If drinking requires `PICK_UP` then
`CONSUME`, the plan starts again from nothing between the two and the sequence never completes. Give
a verb that gets there in one go.

**An action that succeeds without changing anything is worse than a failure.** The character
remembers having drunk while their body says they are dying of thirst. They have to explain the
contradiction, and they explain it out loud. Check before you return `true`.

**The memory cues decide what they will remember.** `Memory > Importance Cues` weights words. Put
only words of violence in there and memory becomes an archive of disasters: everything peaceful
evaporates, everything alarming stays for hours, and the conversation drifts towards emergency by
itself. Ours was 18 words of violence out of 20, with a four-hour half-life, and talk of danger went
from 3 % of lines at the start of a run to 26 % at the end. Put the words of ordinary life in there
too.

**What the character says is not evidence.** Their own speech and private notes are memorised
lightly on purpose (`Importance Said`, `Importance Note`). Raise those and they will take themselves
for whatever they once said at random.

---

## And then

- [Writing actions](README.md#writing-actions): your own actions, without falling into the traps.
- [Translating your NPCs](README.md#translating-your-npcs): every field that holds text.
- [Running NPCs on an online model](README.md#running-npcs-on-an-online-model): the same characters
  on an online model instead of a local one.
