<a id="readme-top"></a>

[![Unreal Engine][unreal-shield]][unreal-url]
[![Platforms][platform-shield]](#prerequisites)
[![Version][version-shield]](#roadmap)
[![Discord][discord-shield]][discord-url]

<br />
<div align="center">
  <img src="images/icon.png" alt="Autonomous NPC AI" width="96" height="96">

  <h3 align="center">Autonomous NPC AI</h3>

  <p align="center">
    Characters for Unreal Engine that listen, take turns, remember and act, on a language model that runs on the player's own GPU.
    <br />
    <a href="Tutorial_EN.md"><strong>Start with the tutorial »</strong></a>
    <br />
    <br />
    <a href="#usage">Reference</a>
    &middot;
    <a href="https://github.com/Nat11-n1/AutonomousNPCIA/issues">Report a bug</a>
    &middot;
    <a href="https://discord.gg/CspDUgzkq3">Discord</a>
  </p>
</div>

<details>
  <summary>Table of contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About the project</a>
      <ul>
        <li><a href="#what-is-in-the-pack">What is in the pack</a></li>
        <li><a href="#built-with">Built with</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
      </ul>
    </li>
    <li>
      <a href="#usage">Usage</a>
      <ul>
        <li><a href="#writing-actions">Writing actions</a></li>
        <li><a href="#translating-your-npcs">Translating your NPCs</a></li>
        <li><a href="#running-npcs-on-an-online-model">Running NPCs on an online model</a></li>
      </ul>
    </li>
    <li><a href="#known-limits">Known limits</a></li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li><a href="#support">Support</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>

## About the project

Autonomous NPC AI is a pack of Unreal Engine plugins for characters driven by a language model.
A character hears what happens around it, waits for its turn to speak, remembers what was said,
gives itself a goal and acts on it. The model runs on the player's GPU through llama.cpp, with no
account and no network. On a machine that cannot carry it, the same characters run on an online
provider instead.

A Character with two components on it is a living NPC, with no node to wire. What it says, what
it is able to do and what it knows about your game are data: one profile asset per species and
language, one catalogue of actions, and the variables you decide to show it.

Three things were left out on purpose:

- **No game system.** No needs, no inventory, no quests. You already have yours, so the pack only
  carries what you hand it.
- **No built-in action.** A character does what its catalogue lists and nothing else. Six ways of
  moving and looking are provided, the rest you write in Blueprint.
- **No voice.** Each finished sentence comes out as text on an event. Wire it to the speech
  synthesis you use.

### What is in the pack

| Plugin | What it does | License |
|---|---|---|
| NPC Perception | The shared bus. What happens is reported once, every character in range perceives it, and one of them is given the floor. | Fab Standard License |
| NPC Agency | The character: the `NPC Brain` and `NPC Agent` components, the profile, the action catalogue and the queue that runs it. | Fab Standard License |
| NPC Examples | Content only. An example NPC and item, four actions, profiles and catalogues for a human and a dog in English and French, and the `NPCDemo` map. | Fab Standard License |
| Llama | [Llama-Unreal][llama-unreal-url] by Getnamo and Mika Pi, modified for this pack. Loads the model and talks to it. | MIT |

### Built with

- [Unreal Engine 5.8][unreal-url]
- [Llama-Unreal][llama-unreal-url]
- [llama.cpp][llamacpp-url], build b9404

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Getting started

### Prerequisites

- Unreal Engine 5.8.
- Windows 64-bit or Linux. Nothing else is built.
- For a local model: a GPU with Vulkan, and a model file in GGUF format. **No model is included.**
  Vision is optional and needs the matching `mmproj` file.
- For an online model instead: an API key from one of the
  [providers listed below](#running-npcs-on-an-online-model). The model then uses no VRAM.
- For characters that walk: a `NavMeshBoundsVolume` set to **dynamic** generation.

To give an idea of size: on an RTX 3060 with 12 GB, Qwen3-VL-30B-A3B in IQ4_XS with vision on took
5.6 GB of VRAM and wrote about 18 tokens per second with its expert weights left in system RAM.
With sixteen layers of them on the card it took 10.6 GB and wrote about 24.

### Installation

1. Add the pack to your Fab library and install it to Unreal Engine 5.8 from the Epic Games
   Launcher.
2. In your project, open **Edit > Plugins**, enable **Llama**, **NPC Perception**, **NPC Agency**
   and **NPC Examples**, and restart the editor.
3. Put your model in `<YourProject>/Saved/Models/` and name it `model.gguf`. That is the path the
   demo loads. For another name, or an absolute path, change `Path To Model` in
   `BP_NPCDemoGameMode`.
4. In the Content Browser settings, tick **Show Plugin Content** and **Show Engine Content**, then
   open `NPCExamples/Maps/NPCDemo` and press Play.
5. Open the Output Log and filter on `LogNPCPerception`. Each `Floor to <name>` line is a
   character taking its turn.

No GPU to spare? Skip step 3 and follow
[Running NPCs on an online model](#running-npcs-on-an-online-model).

From there, [the tutorial](Tutorial_EN.md) builds two characters starting from an empty Blueprint.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Usage

The tutorial is the way in. What follows is the reference you come back to: how to write an action
that works, every field to translate for another language, and the online backend.

### Writing actions

An NPC can do exactly what its `NPC Action Catalog` lists, and nothing else. There are no built-in
actions: an empty catalogue is a character that perceives and speaks but never moves, and it says so
in the log at BeginPlay.

Each row is one verb. The plugin provides six capabilities of its own, chosen by `Native Handler`:

| Native Handler | what it does | instant? |
|---|---|---|
| `MoveToActor` | walks to somebody, to an object, or to a place named for the first time | no, needs a timeout |
| `Follow` | keeps up with somebody until something else is asked | yes, sets a mode |
| `MoveAway` | backs off, without leaving its own perception (`Retreat Radius Factor`) | no |
| `LookAt` | puts its eyes on something and keeps them there | yes |
| `TurnToward` | pivots to face somebody | yes |
| `Wait` | stands there for a number of seconds | no |

Anything else is a Blueprint: set `Impl` to `Class` and point `Action Class` at a child of
`NPC Action`. Override `On Execute`, and call `Finish` when it is over.

The `Verb` is yours. It can be in your language: the plugin never reads it, only `Native Handler`.
Put the spellings the model drifts into in `Aliases`.

#### The two-step trap

**An action that takes two verbs to satisfy a need almost never finishes.** This is the single most
expensive thing to learn the hard way, so it comes first.

A plan comes from what the character has just said or decided, and a new one replaces it at the
next turn. Give a character `PICK_UP` and `CONSUME` as the only way to drink, and this happens:

1. it is thirsty, it decides to drink, the game master writes `PICK_UP flask`;
2. the flask is in its hands, the plan is finished;
3. the next turn makes a new plan from what it says now, and it says it is thirsty again.

Measured over eleven sessions of twenty minutes: a cook picked up a flask and put it down again,
never once drank, and spent every turn explaining out loud that she could not. The refusals people
read as a prompt problem were the truth about her body.

Three ways out, best first:

- **Give a verb that finishes the job in one go.** One row, `EAT_OR_DRINK`, that consumes what is
  within reach without picking it up. This is what fixed it: object actions per session went from 0
  to 11, defective lines from 38 % to 14 %.
- **Let the character run its own steps.** `Use Own Step When Nothing To Do` (on by default) makes it
  carry on with the next step of the goal it wrote for itself instead of waiting to win the floor
  again. Worth about a fifth more plans per session.
- **Raise `Max Actions Per Plan`** so a single plan can hold the whole sequence. A plan costs one
  model call whatever its length, so this is nearly free. See *What it costs* below.

#### Taking an action away

Three levels, from the widest to the narrowest:

- **Delete the row.** The verb does not exist for that species at all.
- **Untick `Enabled` on the row.** The row is skipped as if deleted, but keeps its wording. For an
  action you are still working on, or one a creature should not have yet.
- **`Set Action Enabled(Verb, false)` on the `NPC Agent`.** Takes the verb away from *one character*,
  at runtime, without touching the catalogue its species shares. A broken arm, a rope, a quest that
  unlocks a gesture. `Set Action Enabled(Verb, true)` gives it back, and `Is Action Enabled` reads it.

In all three cases **the verb is left out of what the game master is shown**. That matters: a verb
that is visible but refused costs the character a whole turn reaching for something impossible, which
is exactly how the two-step trap above wastes turns. An action already under way is left to finish
rather than cut off in mid-step.

To try it without wiring anything, the console does the same thing while the game runs:

```
NPCAgency.Action Garran LOOK_AT off
NPCAgency.Action Garran LOOK_AT        (prints whether it is on)
NPCAgency.Action Garran LOOK_AT on
```

#### Saying what an action can be aimed at

To the game master, "pick up the house" reads as well as "pick up the canteen". Nothing in a name
says what the thing is, so the catalogue has to.

1. **Tag the actors.** Add an `NPC Target Tags` component and fill `Tags` with what the actor *is*:
   `Thing.Portable`, `Thing.Drinkable`, `Thing.Building`. The tags are ordinary Gameplay Tags and they
   are yours, the plugin ships none. An actor that already carries Gameplay Tags of its own is read
   as it is.
   Declare them in Project Settings > Gameplay Tags as usual. A content-only plugin that depends on
   NPC Agency can ship its own in `Config/Tags/*.ini`: that folder is read for it, which the engine
   does not do by itself. The examples do this with `NPCExample.Portable` and `NPCExample.Consumable`.
2. **Say what each action wants.** On the catalogue row, `Target Required Tags` lists what the target
   must carry (all of them; a child tag counts as its parent), `Target Blocked Tags` what it must not.
   `PICK_UP` requires `Thing.Portable`.

Left empty, a row accepts any target, so nothing changes until you fill one in. An actor with no
tags at all fails every requirement: you tag what can be picked up, not everything that cannot.

What the game master then reads:

```
People present:
- Merle (close, it sees them)
- the canteen (very close, it sees them) - allows: PICK_UP, EAT_OR_DRINK
- the bowl of water (close, it sees them) - allows: EAT_OR_DRINK
- the house (far away, it sees them)
PICK_UP, EAT_OR_DRINK: only on a target whose line allows it.
```

Only the actions that ask something of their target are named, on the targets that allow them.
`GO_TO` asks nothing and goes anywhere, so it is never listed. A target that no action at all applies
to is left out. If a refused line is written anyway, because the character said it would, the line
is dropped and the log says `PICK_UP does not apply to 'the house'`. The two sentences are
`Target Verbs Suffix` and `Choosy Verbs Rule` in the profile, to translate along with the rest.

`NPCAgency.Dump <name>` prints this list for a character as it stands, with no model involved.

Tags are read each time a plan is asked for, so they can change while the game runs: a chest that
becomes `Thing.Open`. `Can Act On(Verb, Actor)` on the `NPC Agent` answers the same question from a
Blueprint. On an action with two targets (`GIVE the canteen to Merle`), the tags are about the first.

#### Keeping the character's plan on the facts

A character writes its own plan, in steps, and is shown the step it is on. Two profile switches keep
that step from drifting away from what really happened. A character reading *"you are at: put the
canteen down"* puts down a canteen it never held.

- **`Mind > Steps Follow Facts`**: a step that names a thing or a person present only moves on once
  an action has succeeded on that thing; a step the game master finds nothing to do in is skipped; a
  plan confirmed word for word keeps its progress. With it, `Reflection > Grounding` is added to the
  reflection's instructions, and `Reflection > Progress` tells it how far the plan really got.
- **`Mind > Facts Beside Step`**: whatever your game shows to the game master (what the character
  carries, typically) is repeated right under the current step. `Perception > Surroundings Closing`
  ends the list of what is around with a sentence saying that the list is all there is.

Both are off by default and on in the example profiles. The log line `step 2 -> 3 of 5` shows the
first one at work.

#### An action must be able to fail

`Finish` takes a success flag. Returning `true` when nothing happened is worse than failing:

> The example's own `Consume In Place` used to return `true` unconditionally. Out of 470 meals in a
> test session, **197 were eaten by characters who were neither hungry nor thirsty**, and each one
> was written into memory as *"you ate or drank"*. A body that remembers drinking while it is dying of
> thirst has to explain the contradiction, and it explains it out loud: the water must be unsafe,
> someone must be forbidding it, it cannot help you.

So check before you succeed, and say why when you do not. The example now refuses when the need it
would answer is already at zero.

#### An action must be instant, or time-limited

An action that can neither finish on its own nor time out **stops the queue for ever**: the finished
event never fires, and the think-act-think loop dies without an error. Either set `Instant` (it
completes the moment it starts, like gaze or setting a mode) or give it `Timeout Seconds`. The
catalogue warns about a row that is neither, at BeginPlay.

#### Language: `Trace Failure`, not `Reason`

Each row carries the sentences the character reads afterwards, `Trace Success` and `Trace Failure`,
with `{target}`, `{reason}` and `{number}`.

**`Trace Failure` is what the character is told when a step fails.** The `Reason` string your
Blueprint passes to `Finish` goes to the log only. This is deliberate: an action written in Blueprint
reports in whatever language its author typed, and in a French project the example actions were
telling characters *"you do not have that on you"* eighty times a session. Write the sentence once, in
the catalogue, in the language of the profile.

Watch out for rows whose `Trace Failure` is a copy of `Trace Success`. A failure then reads as a
success, and nothing in the log says so.

#### What it costs

The model is the bottleneck. On one local GPU, a session spends about **94 % of the wall clock
inside the model, and two thirds of that reading prompts rather than writing replies**. Every plan
is one call; every step the character chains onto is another.

That has two consequences worth designing around:

- **A longer plan is nearly free, a second plan is not.** Raising `Max Actions Per Plan` from 3 to 6
  took actions per plan from 2.2 to 3.9 and total actions up by 42 %, for the same number of calls.
  But the game master writes more, so there were fewer speaking turns. You are choosing between
  characters that act and characters that talk.
- **Instant actions do not fill time.** In a small room, `MoveToActor` completes in less than half a
  second, and gaze and consumption are instantaneous. Measured across a session, the body was busy
  5 % of the time no matter how the plans were arranged. If you want characters that look alive
  between replies, they need somewhere to walk to. That is level design, not tuning.

#### Console commands

- `NPCAgency.Think <name> <intention>` hands an intention to the game master, as if the character
  had just decided it.
- `NPCAgency.Cancel <name>` empties the queue.
- `NPCAgency.Dump [name]` prints the catalogue and the queue of each NPC, then what is around it as
  the game master reads it, with the verbs each target accepts.
- `NPCAgency.Action <name> <verb> [on|off]` switches a verb for one character, or prints its state.

#### Reading the log

```
Odile: plan requested for "..." (said: "...")
Odile: the game master wrote | 1. GO_TO a water bowl / 2. EAT_OR_DRINK a water bowl
Odile: plan received, 2 action(s), 0 line(s) rejected.
Odile: GO_TO -> succeeded (arrived)
Odile: plan finished (ok).
```

- `line(s) rejected` above zero: a verb or a target that was not recognised. Check `Aliases`.
- **Failures are not logged as `failed`.** The word comes from the result: `no path`, `target gone`,
  `timed out`, `refused`. Grepping for `-> failed` finds nothing and proves nothing.
- `N step(s) dropped, nothing could be seen doing them`: the reflection wrote steps that refuse
  rather than act ("do not move"). They were removed from the goal instead of the goal being thrown
  away. Listed in the profile under `Reflection > Refusal Openings`.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

### Translating your NPCs

Out of the box, NPCs think, speak and remember in English. Nothing in the plugin depends on a
language: every sentence the model reads, and every word the plugin reads back, is a property of the
**NPC profile** (a `NPC Agent Profile` Data Asset).

To make NPCs speak another language, you create a profile in that language. No C++, no recompiling.

#### Setup

1. Look in `NPCExamples/Languages`: one folder per language already translated (`EN`, `FR`), each
   with a profile and an action catalogue per species. If yours is there, skip to step 3.
   Otherwise duplicate `NPCExamples/Languages/EN/DA_Profile_Human_EN` (or create a new
   `NPC Agent Profile`) into a folder named after the language, for example
   `Languages/IT/DA_Profile_Human_IT`. Duplicate the catalogue the same way (`DA_Actions_Human_IT`)
   and point the new profile at it: the summaries the game master reads are in the catalogue.
2. Translate its fields, in the order below. Sections 1 and 2 are enough for NPCs that work, the rest
   polishes them.
3. Use it:
   - for every NPC: **Project Settings > Plugins > NPC Perception > Default Profile**;
   - for one NPC: the **Profile** field of its `NPC Agent` component.

Do the same for each species. A dog profile has its own texts: see `DA_Profile_Dog_EN`.

Keep every `{placeholder}` exactly as it is (`{name}`, `{target}`, `{goal}`...): the plugin fills
them in. Move them around the sentence as your grammar needs.

#### 1. The labels: translate them together, or the NPC breaks

The model answers in labelled lines, and the plugin finds its answer by these labels. A label changed
in one place and not in the other gives an NPC that thinks but never speaks, or never acts. There is
no error message: it silently does nothing.

| Field | Default | Must match |
|---|---|---|
| `Reply Format > Intention Label` | `INTENTION` | the common prompt (section 2) |
| `Reply Format > Note Label` | `NOTE` | the common prompt |
| `Reply Format > Speech Label` | `SAYS` | the common prompt |
| `Reply Format > Sound Label` (animals) | empty (dog: `SOUND`) | the dog's common prompt |
| `Reply Format > Silence Words` | `nothing`, `silence`, `-` | the common prompt: the first word is what the NPC is told to write when it has nothing to say |
| `Reflection > Goal Label` | `GOAL` | `Reflection > Instructions` and `Revision` |
| `Reflection > Steps Labels` | `STEPS`, `STEP` | `Reflection > Instructions` |
| `Reflection > Abandon Word` | `nothing` | `Reflection > Revision` ("write GOAL: nothing if...") |
| `Game Master > Reply Label` | `ACTIONS` | `Game Master > Prefill` (`ACTIONS:\n1.`) |
| `Game Master > End Marker` | `END` | inserted into the rules through `{end}` |

Labels can stay in English even when everything else is translated. The model follows them just as
well. If you are unsure, leave them as they are and only translate the sentences.

#### 2. What the model reads most

These texts set how the NPC thinks. Translate them carefully, and say in them which language it thinks
and speaks in: "You think in Italian" does more than any other sentence.

- **The common prompt**: the rules of being a person in your world, and the three lines to answer in.
  It lives in `Reply Format > Common Prompt`. When that field is empty, the one in the model parameters
  is used instead (`Model Params > System Prompt`, set in the demo by `BP_NPCDemoGameMode`).
- `Reply Format > Persona Balance`: one sentence put after every persona.
- `Reflection > Instructions`, `Revision`, `Plan Format`, and the four `Reason...` fields: the moment the
  NPC stops to choose a goal.
- `Game Master`: `Intro`, `Rules`, and the other lines. This is the call that turns what the NPC wants
  into actions.
- `Brain > Context Preamble`: opens the private part of every request.
- On each NPC: the **Persona** (`NPC Brain` component) and the **Appearance** (`NPC Agent` component).

#### 3. Everything the NPC is told

These are short sentences, and there are many of them. Untranslated ones still work, but the model
then reads two languages mixed together.

- `Perception`:
  - where things are: `Direction...`, `Distance...`;
  - the surroundings list: `Surroundings Header`, `Person Line`, `Item Line`...;
  - what happened: `Speech Line`, `Noise...`, `Arrival Line`, `Repetition Notice`...;
  - moments of calm: `Company Lull`, `Alone`...;
  - memories: `Memory...`, `When...`.
- `Mind`: the goal, the current step, what it just did, how it went.
- `Reasons`: why a built-in capability failed ("no path", "you were cornered"...).
- `Sound Heard`, `Sound Made`, `Appearance Title`.
- `Game Master > Distance...`, `Sees It`, `Does Not See It`: the words the game master reads about the
  people around.

#### 4. Language rules

These are not sentences, but they are specific to each language.

- `Perception > Stop Words`: common words ignored when checking whether two sentences say the same thing
  (repetition, memory). List the frequent words of four letters or more in your language.
- `Memory > Importance Cues`: word starts that make a memory weigh more ("hurt", "blood", "promis"...).
  They are matched on the start of words, without accents or case.
- `Perception > Failure Marker`: a few words present in every failure account (`could not`). Put the same
  words in your failure traces (section 5) and in `Mind > Outcome Failure`.
- `Mind > Contractions`: replacements applied to accounts, for languages that contract words. French:
  "jusqu'a le" becomes "jusqu'au". Italian: "di il" becomes "del", "a il" becomes "al". English needs none.
- `Mind > Articles`: a name starting with one of these gets a lowercase first letter in accounts
  (`A stranger` then reads "a stranger").
- `Game Master > Give Joints`: the words between the thing given and who gets it (`to`, `for`).
- `Perception > Names Joiner`, `Be Singular` / `Be Plural`, `Them Singular` / `Them Plural`: the small
  grammar used to name the people around.
- `Reply Format > Latin Script Only`: drops lines written in another script. It is on by default. Turn
  it **off** for a language not written in the Latin alphabet (Russian, Greek, Japanese...), or every
  line will be dropped.

#### 5. Actions

Every word an action shows the model comes from its catalogue row, so all of it is yours to
translate:

- `Verb` can be in your language (`ALLER_VERS`) or stay in English, with the spellings the model
  drifts into in `Aliases`.
- `Summary` and `Example` are what the game master reads to choose.
- `Trace Success` and `Trace Failure` are how the character is told afterwards what it did. The
  failure sentence comes from here and not from the Blueprint, as explained under
  [Language: `Trace Failure`, not `Reason`](#language-trace-failure-not-reason).
- Copies of the example actions (`BPA_PickUp`, `BPA_PutDown`, `BPA_Consume`, `BPA_EatOrDrink`) report
  in English. Their `Reason` does not reach the model, but it is still what you read in the log.
- `NPC Montage Action` has one string of its own: `Cannot Play Reason`
  (`your body cannot do that`), shown when there is no montage or no mesh.

#### 6. Your game's data

Everything your game hands to the NPC is yours to translate: the descriptions and steps given to
`Add Variable`, the texts of `Request Reflection`, `Report Event`, `Remember`, and the names of things
passed to `Register Named Thing`.

In the example, `BP_NPC` keeps these texts in variables (`Thirst Description`, `Thirst Steps`,
`Thirst Reason`...). A child Blueprint in another language only has to change their default values.

#### 7. Project settings

- **Project Settings > Plugins > NPC Perception > Player Display Name**: what NPCs call the player
  (`A stranger`).
- **Project Settings > Plugins > Llama Cloud API > Prefill Instruction**: only for NPCs running on an
  online model (see [Running NPCs on an online model](#running-npcs-on-an-online-model)). It asks the
  model to start its reply with the right label.

#### Checking your translation

Play the demo map with the `LogNPCPerception` and `LogNPCAgency` logs open:

- `Floor to <name> | ...` shows the exact text an NPC receives. Any line still in English is visible
  there. Search the whole log for a word your language never uses (`you`, `your`, `not`): that is how
  the English reasons above were found, after six days of blaming the prompt for them.
- `plan requested for "..."` then `plan received, N action(s)` shows that the game master understood
  the intention. Many `line(s) rejected`: check the verbs and aliases.
- An NPC that thinks but never speaks: a speech label or silence word that does not match the common
  prompt (section 1).
- `reflection without a readable goal`: `Goal Label` does not match `Reflection > Instructions`.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

### Running NPCs on an online model

By default NPCs run on a model loaded on the player's GPU. On a machine without the VRAM for it, or
with the GPU already busy with the game, they can run on an online model instead. Nothing else changes:
perception, memory, actions, the game master and reflection all work the same way.

#### Providers

| Provider | Format | Key | Notes |
|---|---|---|---|
| OpenRouter | OpenAI | yes | one key for most models of every lab |
| OpenAI | OpenAI | yes | |
| Anthropic | **Anthropic Messages (native)** | yes | Claude |
| Gemini | OpenAI (Google's compatible endpoint) | yes (AI Studio) | |
| Mistral | OpenAI | yes | |
| xAI | OpenAI | yes | Grok |
| DeepSeek | OpenAI | yes | |
| Groq, Cerebras | OpenAI | yes | very fast inference of open models |
| Together AI, Fireworks, DeepInfra | OpenAI | yes | hosted open models |
| Perplexity | OpenAI | yes | |
| Cohere | OpenAI (compatibility endpoint) | yes | |
| Azure OpenAI | OpenAI | yes | `Base Url` = `https://<resource>.openai.azure.com/openai/v1`, `Model` = your deployment |
| Ollama, LM Studio | OpenAI | no | a local server, on this machine or another one of yours |
| Custom | OpenAI or Anthropic (`Api Format`) | optional | vLLM, llama-server, your own proxy... |

#### Settings

**Project Settings > Plugins > Llama Cloud API**

| Setting | What to put |
|---|---|
| Backend | `Cloud API` |
| Provider | your provider. The address fills itself in; `Custom` takes any OpenAI-compatible `Base Url` |
| Model | the model's name at that provider: `google/gemini-2.5-flash` (OpenRouter), `gpt-4o-mini` (OpenAI), `claude-haiku-4-5` (Anthropic), `gemini-2.5-flash` (Gemini)... |
| Light Model | optional: a cheaper, faster model for the game master and reflection calls, which are short |
| Send Images | sends what each NPC sees along with its turn. Needs a model that reads images; costs tokens on every turn |
| Max Concurrent Requests | how many requests run at once. Keep it under your provider's rate limit |

With `Cloud API`, `Load Model` loads nothing and uses no VRAM. It still fires `On Model Loaded`, so
existing Blueprints keep working.

#### The API key

The key is **never** a project setting: project settings are packaged with the game, and anyone could
read a key from them. It is read from, in this order:

1. **`Set Cloud API Key`** (Blueprint), for this session only, never saved. Use it to let players enter
   their own key in your options menu, or to hand over a key from your own backend.
2. **An environment variable**, `NPC_API_KEY` by default (setting `Api Key Environment Variable`).
3. **In the editor only**: **Editor Preferences > Plugins > Llama Cloud API Key (this machine)**.
   Editor Preferences, not Project Settings: the key belongs to your machine, not to the project. It
   is stored in `Saved/Config`, which is not shared with the project and not packaged. Type the key
   in the field and **press Enter**: an Unreal text field that loses focus without Enter keeps
   nothing.

Shipping a game with your own key inside means every player spends your credit, and anyone can take
the key out. If you want players to use the online backend without a key of their own, put a small
server of yours between the game and the provider, and point a `Custom` provider at it.

#### Holding the reply format

NPCs answer in labelled lines (`INTENTION:`, `NOTE:`, `SAYS:`). On the local model, the plugin writes
the first label itself, so the format always holds. Online, it depends on the provider (setting
`Prefill`):

- `Assistant Message`: the start of the reply is sent for the model to continue. It holds best.
  `Automatic` uses it with OpenRouter, Anthropic, Mistral and Ollama.
- `Instruction`: the start of the reply is asked for in the prompt (`Prefill Instruction`, to translate
  along with your other texts). It works everywhere. If the model forgets, the label is added back to
  its reply.

Some models refuse options that others accept:
- Claude 4.6 and later refuse a prefilled reply.
- Claude 4.7 and later, and OpenAI's reasoning models, refuse a temperature.

You do not need to know which: the first refusal (HTTP 400) is caught, the request is sent again
without the option, and that model is not asked again for the rest of the session. The log says so
once: `Cloud: <model> refuses a prefilled reply...`.

#### Other Blueprint nodes

- `Is Using Cloud API`
- `Has Cloud API Key`
- `Set Use Cloud API`: switches between the local and the online model for the session, for example
  from a graphics/performance menu. Call it before `Load Model`.

#### When it fails

Failures are logged under `LlamaLog` as `Cloud request failed: ...`, with the provider's own message:

| Message | Meaning |
|---|---|
| `HTTP 401` | wrong key |
| `HTTP 404` | wrong model name or address |
| `HTTP 429` | rate limit. The request is sent again after a pause, up to `Rate Limit Retries` times (the log says `trying again in`). If it still fails, lower `Max Concurrent Requests` |
| `no API key` / `no Model set` | a setting is missing |

A reasoning model that returns empty replies is spending the whole token budget thinking: set
`Reasoning Effort` to `minimal` or `low`, or pick a model without reasoning.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Known limits

- It is a beta, version 0.1.0. Field names can still move between versions.
- Windows 64-bit and Linux only.
- No speech synthesis and no speech recognition wired in. Sentences come out as text.
- A local model answers one request at a time, so characters take turns and a turn takes seconds,
  not frames. On the RTX 3060 above, the first word of a reply came after 4 to 5.5 seconds.
- Vision is paid on every reply, and its cost follows the area of the picture. The tutorial has the
  numbers.
- What a character says depends on the model you load. Test with the one you intend to ship.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Roadmap

- [x] Switch an action off per catalogue row, and per character at runtime
- [x] English and French language packs, for a human and a dog
- [x] Online backend, with the local model as the default
- [ ] One model per character, so that two languages can share a scene with a model each. The texts
      already follow the character through its profile; the model is still shared by everyone.
- [ ] Speech synthesis. It exists and is not shipped: it is not good enough yet.

Requests go to the [issue tracker][issues-url].

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Support

The source of the three NPC plugins is not public, so there is nothing to fork. Bugs go to the
[issue tracker][issues-url] of this documentation repository. A report that can be acted on has:

1. the engine version and the platform;
2. the backend: local model (which GGUF, which quantisation) or online (provider and model);
3. the `Floor to <name>` line of the turn that went wrong, and the lines after it down to
   `plan finished`.

That third item is the exact text the character received. What went wrong usually shows there first.

Questions and show-and-tell are better on [Discord][discord-url]. Corrections to these docs are
welcome as pull requests here.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## License

The pack mixes two licenses. Which one applies depends on the folder.

**NPC Perception, NPC Agency, NPC Examples.** Copyright (c) 2026 Nat11. All rights reserved.
Distributed through Fab under the [Fab Standard License][fab-eula-url]. You may use and modify them
in your projects and ship those projects commercially. You may not resell them or redistribute
them on their own. Each plugin folder has its `LICENSE` file.

**Llama.** [Llama-Unreal][llama-unreal-url], copyright (c) 2023-current Jan Kaniewski (Getnamo),
Mika Pi and contributors, under the MIT license. Its `LICENSE` file is in its folder, unchanged.
The copy in this pack is based on version 1.1.0 and is not the original. The changes are released
under the same MIT license.

Added:

- an online backend: the same requests can go to an API provider instead of the local model, with
  its settings page, key handling and retries;
- the NPC request node: one self-contained request with a system prompt, replayed history, an
  optional picture, a prefilled start of reply and a ceiling on reply length, read back as
  labelled lines;
- prompts formatted by the model's own chat template, through llama.cpp's Jinja engine;
- a model parameter, `MoE Expert Layers On CPU`, that keeps the expert weights of a
  mixture-of-experts model in system RAM.

Changed:

- repeat penalty and temperature are applied in every sampling mode (they were ignored in one);
- the reasoning markers of Gemma-style models are recognised;
- reloading a model clears the previous conversation, and requests left behind by a level that
  ends are cancelled;
- the plugin declares Windows 64-bit and Linux only, and builds on its own for packaging.

**Third-party software inside the Llama plugin.** Full license texts are in
`ThirdPartyNotices.txt`, in the Llama plugin folder.

| Component | License | Copyright | Shipped as |
|---|---|---|---|
| [llama.cpp][llamacpp-url] b9404 (with ggml, mtmd and the Jinja engine) | MIT | The ggml authors | binaries, headers, sources |
| [whisper.cpp](https://github.com/ggml-org/whisper.cpp) | MIT | The ggml authors | sources |
| [JSON for Modern C++](https://github.com/nlohmann/json) | MIT | Niels Lohmann | header, linked into llama.cpp |
| [cpp-httplib](https://github.com/yhirose/cpp-httplib) | MIT | yhirose | linked into llama.cpp |
| [stb_image](https://github.com/nothings/stb) | MIT or public domain | Sean Barrett | linked into llama.cpp |
| [miniaudio](https://github.com/mackron/miniaudio) | public domain or MIT-0 | David Reid | linked into llama.cpp |
| [hnswlib](https://github.com/nmslib/hnswlib) | Apache-2.0 | the hnswlib authors | headers |

**Models.** No model is distributed with the pack. Every GGUF file comes with the license of
whoever published it, and some of them forbid commercial use. Read the model card before you ship a
game with it.

**Online providers.** With the online backend, what a character is told, and the picture of what
it sees if `Send Images` is on, is sent to the provider you chose and handled under that provider's
terms and privacy policy. Telling your players, and getting their consent where the law asks for it,
is your responsibility as the publisher of the game. The pack sends nothing anywhere when the local
model is used.

**What a model writes.** The pack passes text to a model and reads its answer. It does not control
that answer and does not filter it. No warranty is given on what a character says or does.

**Trademarks.** Unreal Engine and Fab are trademarks of Epic Games, Inc. Vulkan is a trademark of
the Khronos Group. OpenAI, Anthropic, Claude, Google, Gemini, Mistral, xAI, Grok, DeepSeek, Groq,
Cerebras, Together AI, Fireworks, DeepInfra, Perplexity, Cohere, Azure, OpenRouter, Ollama, LM Studio
and Qwen belong to their respective owners. This pack is not affiliated with any of them, nor
endorsed by them.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Contact

Nat11, on [Discord][discord-url].

Documentation and issues: [https://github.com/Nat11-n1/AutonomousNPCIA][repo-url]

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Acknowledgments

- [Llama-Unreal][llama-unreal-url] by Jan Kaniewski (Getnamo) and Mika Pi. This pack stands on it.
- [llama.cpp][llamacpp-url] and whisper.cpp by the ggml authors.
- [Best-README-Template](https://github.com/othneildrew/Best-README-Template) for the skeleton of
  this page.
- [Shields.io](https://shields.io) for the badges.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- LINKS -->
[repo-url]: https://github.com/Nat11-n1/AutonomousNPCIA
[issues-url]: https://github.com/Nat11-n1/AutonomousNPCIA/issues
[discord-url]: https://discord.gg/CspDUgzkq3
[fab-eula-url]: https://www.fab.com/eula
[unreal-url]: https://www.unrealengine.com
[llama-unreal-url]: https://github.com/getnamo/Llama-Unreal
[llamacpp-url]: https://github.com/ggml-org/llama.cpp
[unreal-shield]: https://img.shields.io/badge/Unreal%20Engine-5.8-0E1128?style=for-the-badge&logo=unrealengine&logoColor=white
[platform-shield]: https://img.shields.io/badge/Platforms-Win64%20%7C%20Linux-555555?style=for-the-badge
[version-shield]: https://img.shields.io/badge/Version-0.1.0%20beta-orange?style=for-the-badge
[discord-shield]: https://img.shields.io/badge/Discord-Join-5865F2?style=for-the-badge&logo=discord&logoColor=white
