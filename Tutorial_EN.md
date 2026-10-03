# Two characters, from scratch

A nine-step course. By the end, two NPCs talk to each other, remember each other, give themselves a
goal and act on it — with no network, no key, and not one line of C++.

The examples are in French because that is where this system was built and measured. Nothing in them
is French-specific: swap the strings and the same nine steps give you two characters in any language
the model speaks. [Translating your NPCs](README.md#translating-your-npcs) lists every field that holds text.

---

## 1. How it works, in one page

The system is a loop with four beats. Understand this one and the rest follows.

**Perceive.** Everything that happens — a spoken line, a noise, someone walking in — is sent to the
*bus* (`NPC Perception`). The bus decides who heard it, from distances and hearing radii.

**Take the floor.** The bus lets only one character speak at a time. It gives the turn to whoever the
situation concerns most: someone spoke to them, someone named them, they are close by, they have not
spoken recently. That is the *arbitration*, and it is what stops three NPCs from answering at once.

**Think.** Whoever has the floor gets a prompt and answers in three lines:

```
INTENTION: I lean down to the bowl to drink
NOTE: this water smells wrong
SAYS: Have you seen the state of this water?
```

Only the `SAYS` line is spoken out loud. `INTENTION` is what they are about to do, `NOTE` is what
they keep. Those three words are the defaults; they are fields, and section 4 changes them.

**Act.** The `INTENT` goes to the *game master*, a second request to the model whose only job is to
translate it into verbs the body knows how to run. It decides nothing, it translates.

And alongside the loop, a **reflection** now and then: the character gives themselves a goal and
steps, which stay in their context until they change them.

### The three layers of the prompt

This is the key to all the configuration:

| layer | where | what it says |
|---|---|---|
| **common** | `Profile > Reply Format > Common Prompt` | the rules of being a character, the three-line format |
| **personality** | `NPC Brain > Persona`, on each actor | who *this* character is |
| **situation** | built every turn | what they perceive, what they remember, what their body tells them |

The first and the third come from the **profile**, a Data Asset. The second is plain text on the
actor. Nothing is hardcoded in the plugin: every sentence the model reads comes from there.

If you leave `Common Prompt` empty, the model's own `System Prompt` is used instead. That is handy
while you are still moving text around — one prompt for the whole scene, held in your GameMode — but
the profile is the right home for it, because that is what makes a second language a second asset
rather than a second build.

---

## 2. Setting up

**The model.** Any GGUF `llama.cpp` can load. Set `PathToModel` in the model parameters, and
`MMProjPath` if you want the characters to *see* — leave it empty and vision is simply off. Start
without vision: it is twice as fast and changes nothing else.

If you do turn vision on, look at `NPC Brain > Vision > Vision Resolution` before anything else. The
token cost follows the **area** of the picture, so doubling the side quadruples the price of every
single reply. On our scene, 256 px put the prompt at 2,359 tokens and 128 px at 1,536 — while no
picture at all only got to 1,635. In other words 128 px buys back everything cutting vision would
have bought, and the characters still make out who is in front of them and what is lying on the
ground. Raise it only for a model that has to read something written in the world.

**Project settings.** `Project Settings > Plugins > NPC Perception`. You only need two fields for
now:

- `Default Profile`: the profile used by any NPC that does not bring its own;
- `Player Display Name`: what the NPCs call the player (`A stranger`).

---

## 3. The first character

Create a Blueprint deriving from **Character**, call it `BP_Villager`, and add two components:

- **NPC Brain**
- **NPC Agent**

That is all. **Zero nodes.** The brain registers itself with the bus at `BeginPlay`, binds to the
speaking turn, and the loop runs. An actor with those two components is a living character.

On the pawn, check `Auto Possess AI = Placed in World`, otherwise it will think and speak but will
not be able to move — the plugin tells you so in the log, but you may as well know now.

For it to walk you also need a **NavMeshBoundsVolume** in your scene, set to **dynamic** generation.
With no navigation, the movement actions are dropped from the catalogue and the model is never even
offered them.

To hear what it says: on the `NPC Brain`, the **On Sentence** event fires for each finished sentence.
Wire it to a `Print String`, or to your speech synthesis.

---

## 4. The profile

This is the step that matters. Look in `NPCExamples/Languages` first: each folder there is a language
already translated (`EN`, `FR`), with a profile and an action catalogue per species, ready to assign.
If yours is missing, duplicate `NPCExamples/Languages/EN/DA_Profile_Human_EN` into a folder of its
own and name it after the language (`DA_Profile_Human_IT`).

### The rule not to miss

The model answers with labels, and the code finds its answer through those labels. **Change a label
on one side and not the other, and the NPC thinks but never speaks — and nothing in the log says
why.**

The labels live in `Reply Format`:

| field | French value |
|---|---|
| `Intention Label` | `INTENTION` |
| `Note Label` | `NOTE` |
| `Speech Label` | `DIT` |
| `Silence Words` | `rien`, `silence`, `-` |

And the `Common Prompt`, in the same group, must use **exactly** those words:

```
Tu es une personne qui vit dans ce monde, avec un nom, un corps et une histoire a toi.
Tout ce que tu ecris sort de cette personne-la.

Tu reponds en trois lignes, dans cet ordre, et rien d'autre :
INTENTION: ce que tu fais maintenant, en une phrase.
NOTE: ce que tu retiens de ce moment, pour toi seul.
DIT: ce que tu dis a voix haute. Ecris "rien" si tu preferes te taire.
```

Same for the reflection: `Reflection > Goal Label` = `BUT`, `Steps Labels` = `ETAPES`, and the
`Instructions` have to use those words.

### A creature that cannot talk

Animals go through the same loop, with one field changed: leave `Speech Label` **empty** and the
character never speaks. Give it `Sound Label` instead and it makes a sound the others hear —
`SOUND: barks at the door` reaches the bus as a perceived noise, not as words.

Give it its own profile too, not the human one with the labels swapped: its `Common Prompt` is
where you tell it what kind of creature it is. Ours is 1,301 characters against 3,854 for the
humans — a third — and the dog costs half as much per reply because of it.

### The prefill, the only thing that holds a format

`Reply Format > Prefill` holds the beginning of the answer, written in advance in the model's place.
Leave it **empty** and the plugin writes the first label followed by a colon, which is what you want
in almost every case. Write your own only for something else — and then **never end it with a
space**: a trailing space makes the model emit bare numbers.

An instruction always gives way after fifty turns. A beginning of answer written in advance never
gives way. That is the difference between a format that holds for a minute and one that holds for an
hour.

### Keep the common prompt short

Every character pays for the common prompt on **every single reply**, and it is by far the biggest
item: on our scene it was two thirds of one character's whole prompt — more than the persona, the
memories and the picture put together. Ours went from 4,642 to 3,854 characters with no rule removed,
only repetition taken out: bullets explaining to the model what each line of perception already says
by itself, and a permission that was granted twice.

A useful ratio, if you want to price a block without running anything: **about 3.5 characters per
token** for French on a Qwen-class model. Measure it once on your own language and keep the number.

### The rest

Everything else in the profile is text to translate: the bus sentences ("You hear…", "X is in front
of you…"), the action reasons, the memory headers. [Translating your NPCs](README.md#translating-your-npcs) gives the complete
list, field by field.

Keep every `{placeholder}` as it is — `{name}`, `{target}`, `{goal}` — the code fills them. Move them
around inside the sentence to suit your grammar, but do not rename them.

Finally, point `Project Settings > Plugins > NPC Perception > Default Profile` at your new profile.

---

## 5. Two characters, not two copies

Drop `BP_Villager` twice into your scene. They share the profile — so the language, the format, the
way they remember — and differ in three things only.

**The name.** `NPC Agent > NPC Name`. That is the name the others know them by and call them by.

**The personality.** `NPC Brain > Persona`. Write it **in prose, and in the positive**:

> Odile, cook at the relay for eleven years. She feeds people and that is enough for her. She speaks
> plainly, without detour, and does not like people hanging around her kitchen.

> Garran, guard at the north gate. Gruff, sparing with words, he does you a favour when it suits him
> and says so when it does not.

**Never list what they must not do.** "You are not an assistant" produces "I cannot help you with
that". Name a limit and the model installs it. Describe who they are, not what they are not. And do
not quote example lines: they will be recited word for word.

**The appearance.** `NPC Agent > Appearance` describes **their own** look, never anyone else's — what
they know of themselves from looking down. What the others look like comes through vision, if they
have it.

At this point, hit play and watch: they talk to each other, one at a time, and remember what was
said.

---

## 6. Giving them something to talk about

A character whose only subject is hunger will talk about hunger. That is the cause of most of the
behaviour people take for a flaw in the model.

`NPC Brain > Add Variable` takes **any value** from your game — a number, a boolean, a string, an
array, a struct — and hands it to the character in your own words.

| parameter | what it is for |
|---|---|
| `Value` | the value. Wire a **variable of this Blueprint** and it stays live; wire anything else and it is a copy, to be refreshed with `Update Variable`. |
| `Description` | what it means. May contain `{value}`. |
| `Steps` | turns a number into a sentence: *up to 60 → "You are a little thirsty."* An empty sentence says nothing at all. |
| `Show To Game Master` | shows it to the action translator too — useful for what they are carrying. |

Example, in `BeginPlay`:

```
Add Variable  Value = Thirst (float variable of the Blueprint)
              Description = "Your thirst"
              Steps = [ 30 -> ""                                        ]
                      [ 60 -> "You are a little thirsty."               ]
                      [ 85 -> "You are thirsty, your mouth is dry."     ]
                      [100 -> "Your throat burns and your mouth is pasty." ]
```

**Describe the body, do not dictate the thought.** "Your throat burns" is a state. "All you can think
about is drinking" is an order, and the model obeys: in our measurements that one sentence made the
characters go for water 485 times against 15 for food, while they were starving to death.

Give them the world too: `Register Named Thing` on the bus names an object, and it shows up in what
they perceive — and becomes a possible target for their actions.

---

## 7. Giving them something to do

Create an `NPC Action Catalog` and assign it to the `NPC Agent`. **There is no built-in action**: an
empty catalogue is a character who perceives and speaks but never moves.

Six capabilities are provided by the engine, picked with `Native Handler`:

| Native Handler | what it does |
|---|---|
| `MoveToActor` | walks to someone, an object, or a place named for the first time |
| `Follow` | follows someone |
| `MoveAway` | backs off |
| `LookAt` | sets eyes on |
| `TurnToward` | turns towards |
| `Wait` | waits |

The `Verb` is yours, in your language: `ALLER_VERS` with `Native Handler = MoveToActor` works
perfectly well. Put the spellings the model drifts towards into `Aliases`.

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

Write both traces, and write the failure one as a **reason**, not as a repeat of the success line.
"You lost sight of {target}" tells the character something; "you followed {target}" after a failure
tells them a lie they then have to justify out loud.

For your own gestures: `Impl = Class`, and `Action Class` points at a child of `NPC Action` where you
override `On Execute` and call `Finish`.

### Taking an action away

Two ways, for two different needs.

`Enabled` on a catalogue row, for a whole species: the row stays, with its summary, its example and
its two traces, but the verb disappears. Use it while iterating, so you do not have to delete text
you will want back.

`Set Action Enabled (Verb, false)` on the `NPC Agent`, for one character at a time: a broken arm, a
rope, a quest that unlocks a gesture, a place where one does not run. The verb leaves that
character's game-master prompt, so the model is not offered it and does not waste a turn being
refused. An action already running is left to finish rather than cut off mid-step.

To try it with nothing wired up, the console does the same while the game runs — `NPCAgency.Action
Garran LOOK_AT off`, and `on` to give it back.

Before writing an action, read [Writing actions](README.md#writing-actions). There are three traps in there, and one of
them costs a week to find on your own.

---

## 8. Checking that it works

Open the log on `LogNPCPerception` and `LogNPCAgency`. You should see this scrolling past:

```
Floor to Odile | Garran : Have you seen the water? / What you have in mind: ...
Odile sees | - Garran is in front of you, a few steps away. ...
Odile: plan requested for "I lean down to the bowl" (said: "...")
Odile: the game master wrote | 1. GO_TO a water bowl / 2. DRINK a water bowl
Odile: plan received, 2 action(s), 0 line(s) rejected.
Odile: GO_TO -> succeeded (arrived)
Odile: plan finished (ok).
```

**The `Floor to` line is your best tool.** It is the exact text the character receives. Anything
wrong with their behaviour shows up there first — a sentence left in the wrong language, a need they
cannot satisfy, an absurd goal they have been dragging around for ten minutes.

A few symptoms and their cause:

| what you see | what it is |
|---|---|
| thinks but never speaks | `Speech Label` or `Silence Words` do not match the common prompt |
| `reflection without a readable goal` | `Goal Label` does not match the `Instructions` |
| many `line(s) rejected` | verbs or targets the parser does not know: look at `Aliases` |
| does not move at all | no `AIController`, or no navmesh — the log says so at `BeginPlay` |
| repeats the same line | normal up to a point: the guard cuts a reply that repeats itself |

And a reading trap: **searching for `-> failed` finds nothing.** Failures are logged with their
reason — `no path`, `target gone`, `timed out`, `refused`.

---

## 9. The four traps that cost real time

**A need that takes two actions is almost never satisfied.** If drinking requires `PICK_UP` then
`CONSUME`, the plan restarts from scratch between the two and the sequence never completes. Give a
verb that gets there in one go.

**An action that succeeds without changing anything is worse than a failure.** The character
remembers having drunk while their body says they are dying of thirst: they have to explain the
contradiction, and they explain it out loud. Check before you return `true`.

**The memory cues decide what they will remember.** `Memory > Importance Cues` weights words. Put
only words of violence in there and memory becomes an archive of disasters: everything peaceful
evaporates, everything alarming stays for hours, and the conversation drifts towards emergency all by
itself. Ours was 18 words of violence out of 20, with a four-hour half-life, and danger talk ratcheted
from 3 % of lines at the start of a run to 26 % at the end. Put the words of ordinary life in there
too.

**What the character says is not evidence.** Their own speech and private notes are memorised lightly
on purpose (`Importance Said`, `Importance Note`). Raise those and they will take themselves for
whatever they once said at random.

---

## And then

- [Writing actions](README.md#writing-actions): writing your own actions without falling into the traps.
- [Translating your NPCs](README.md#translating-your-npcs): the exhaustive list of fields that hold text.
- [Running NPCs on an online model](README.md#running-npcs-on-an-online-model): running a character on an online model instead of a local one.
