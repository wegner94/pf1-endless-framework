# PF1 Endless Framework: concept

*A plan for a set of connected Foundry VTT modules for Pathfinder 1e. Each module works on its own, and does more when the others are installed. Nothing here is built yet. Status: concept, 2026-11-17.*

Contents:
- [1. The idea](#1-the-idea)
- [2. Principles](#2-principles)
- [3. The core](#3-the-core)
- [4. The modules](#4-the-modules): the [Check Requester](#41-check-requester-exists), [Items and Merchants](#42-items-and-merchants), [Settlements](#43-settlements), [NPCs and Relationships](#44-npcs-and-relationships), [Factions](#45-factions), [Research](#46-research)
- [5. More module ideas](#5-more-module-ideas)
- [6. What to take from the Dream Danger Zones tool](#6-what-to-take-from-the-dream-danger-zones-tool)
- [7. How the modules connect](#7-how-the-modules-connect)
- [8. A possible build order](#8-a-possible-build-order)
- [9. Open questions](#9-open-questions)

---

## 1. The idea

PF1 has many campaign systems beyond combat: settlements with their modifiers, contacts and relationships, organisational influence, research, downtime, kingdoms. Most tables run them on paper, or not at all, because tracking them by hand is tedious and the numbers they produce (a +3 Lore here, a Trust 4 contact there) never reach the dice.

The framework puts each system in its own Foundry module, and they share one simple contract:

- **The GM never re-types a number Foundry already knows.** Bonuses are read from the sheet when they're needed. Situational modifiers (a settlement's Lore, a contact's help, a faction's boon) travel with the check request, so the player's roll already includes them.
- **Each module stands alone.** Settlements is useful without Factions. When both are active, a faction can own a settlement, and the faction's standing can change that settlement's prices.
- **Rules are data.** The PF1 rules are one content pack and homebrew is another. The Dimension of Dreams setting, for example, would be a pack of currencies, merchants and item templates.

## 2. Principles

These are lessons from the Dream Danger Zones tool, where they were tested at the table.

1. **Read stats, don't store them.** Records hold links to Foundry actors and items (uuids), not copies of their bonuses (the Dream tool's D8 and D32). A level-up never leaves a module out of date.
2. **Known modifiers go with the roll.** The modules work out the situational modifiers they know about, show each with its source, and pass them to the Check Requester as the player's situational modifier. That way nothing is added twice, and nothing is added by hand (the Dream tool's D9 and `uncoveredModifiers`).
3. **Never roll a player's check for them.** Modules ask for checks through the Check Requester. They may roll tables and amounts that don't depend on a character (a restock, a random event), always behind a visible 🎲 button or a setting (the Dream tool's D4 and D17).
4. **History is a log of events.** Anything that changes over a campaign (influence points, Trust, a settlement's events, a merchant's stock) is a list of events, and its current state is replayed from them. That gives undo, an audit trail ("why is our standing 12?"), and an end-of-session summary for free (the Dream tool's §5.2).
5. **Setup is not the story.** Editing a faction's description or a merchant's template is setup: it isn't logged and isn't undone. A boon earned or a task failed is story (the Dream tool's D33).
6. **Automation asks first.** Costs and gains (a contact's fee, an upkeep, a tithe) are applied automatically, after asking (the default), or only noted, set per kind. "Ask" means a popup with *No* as the default (the Dream tool's D30).
7. **Places remember.** Per-place state (standing in *this* court, Trust with *this* contact) is stored with the place and comes back when the party returns (the Dream tool's D24).
8. **Journals are first-class.** Content can live in ordinary journal entries, written by hand. Modules can read a stat block from a journal page and write one back. They don't hide data in places the GM can't see or edit.
9. **One look.** Dark mode, one shared style and one set of controls across modules. Anything that only works in Foundry is marked the same way everywhere.
10. **Players see the fiction, the GM sees the rule.** A request tells the player what's happening and where. The GM's note says which rule asked for it, the DC, and where the DC comes from (the Dream tool's `checkFluff` and `gmWhereText`).

## 3. The core

`pf1-endless-core` is a small library module that the others depend on. It holds only what at least two modules need.

| Part | What it does |
|---|---|
| **API registry** | `game.endless.register(id, api)` and `game.endless.get(id)`. Modules find each other at runtime, and a feature lights up only when the module it needs is active (soft dependencies). |
| **Event log** | A small event-sourcing kit: `dispatch(command) → events`, replay, undo by command group, setup events kept out of undo, and readable log lines. |
| **Storage** | Records kept as JSON files in the world's folder (`worlds/<world>/endless/<module>/<id>.json` with an index), written by the GM's client. Files are used rather than documents because every document change is sent to every client (the Dream tool's §11.6). Small per-actor and per-item data goes in flags under one namespace, `flags.endless.<module>`. |
| **Schemas and migrations** | Every record carries `schema: "endless.<kind>/<n>"`. Loading an older one runs its migrations. |
| **Modifier model** | `{ value, type, source, ref, appliesTo }`, PF1's stacking rules shown (not enforced), a sum with a breakdown, and the conversion to a check request's situational modifier. |
| **Check bridge** | A thin wrapper over the Check Requester: a check spec (skill, save or ability with alternatives; DC with its breakdown; known modifiers; player and GM text) → request → result (total, natural die, declined) → an event. |
| **Pickers** | An actor picker (filter as you type, folders, a tick for "and its subfolders"), an item picker, a journal-page picker. |
| **Time** | One clock for everything: Foundry's world time plus scheduled rules (`every 1 day`, `monthly`, `at <date>`), so restocks, upkeep and Trust decay come due by themselves. It works with Simple Calendar or Calendaria if one is installed (the Dream tool's scheduler, generalised). |
| **Journal sections** | A parser for stat blocks written in journal pages: a heading, then `Label value; Label value` lines. Also a writer that produces the same layout, so the GM can edit either way. |
| **Rules links** | Text such as "Lore (GMG p. 205)" or a `[ID]` links to a journal page or a content-pack entry, shown in a popup with *Back* (the Dream tool's D18). |
| **UI kit** | CSS variables, panels, gauges and tracks (bands, two-sided bars), a resizable side panel layout, keyboard shortcuts. |

## 4. The modules

### 4.1 Check Requester (exists)

`pf1-check-requester`. The GM asks for skill checks, saves, ability checks, CL, concentration and CMB, and the player rolls from a popup. It also has a history, a windowless API (`send`, `results`, `cancel`, `checksFor`, `roll`, `bonusOf`), alternatives the player picks from, a filled-in situational modifier, and the natural d20.

What the framework would add:
- **Context from other modules**: a request can carry its source ("Lore +3, Absalom") as a labelled modifier the player sees, rather than a bare number.
- **Group and opposed checks**: aid another (the helpers' results feed the main roll), one roll for the party (the best result, or everyone's), and opposed rolls (Bluff against Sense Motive, both requested at once).
- **Repeated checks**: a check made every 8 hours until a threshold, for Research and downtime, with a running total.
- **Hidden DC and secret checks**: the GM rolls for the player (Perception for a trap), and the player sees nothing or only the fiction.
- **Request templates**: saved checks such as "Gather Information in town" (Diplomacy, 1d4 hours, settlement Lore) that other modules can trigger.

### 4.2 Items and Merchants

`pf1-endless-items`. A general version of what the Dream tool's item library does, working with **Item Piles**.

**Catalogue**
- Every item from the PF1 compendiums, homebrew items, and *item templates*.
- Filters by type, slot, aura school, CL, price, source and tags.
- An editor for your own items and for overrides of compendium items.

**Item templates (transformations)**
- A rule that turns any item into a variant, with a price factor, extra tags, a description line, and upkeep or fuel.
- Examples: the Dream tool's *dreamcraft copy* (1 Dream per 1,000 gp, upkeep in Wisps), *masterwork*, *timeworn*, *cursed*, *broken*, *dream-bound*.
- Applied when an item is sent, so the item is always created as a full copy, never as a compendium reference.

**Currency systems** are templates too:
- A set of coins with exchange rates, which coin is primary, how prices are converted ("1 Dream = 1,000 gp") and how they're shown ("4,000 gp (4 Dreams)").
- The tool writes them into Item Piles' `overrideCurrencies`.
- Ships with standard PF1 coins, the Dimension of Dreams coins, and examples: trade goods as currency, a setting where only gems are accepted, a faction's scrip.

**Merchants**
- Lists of items and quantities, a description (written into Item Piles' description), a stamp line on each item ("bears the maker's sigil"), and buy and sell factors (PF1's sell at half, a patron's 10% discount).
- Sent to an actor, or to a new actor made from a merchant template that copies its Item Piles setup.

**Stock from a settlement** (with Settlements)
- Whatever the GM stocks by hand, the settlement's Base Value and Purchase Limit decide what is *always* available.
- The minor, medium and major item counts are rolled into the stock (behind 🎲), and restocks are scheduled (weekly, or monthly as the GMG suggests).

**Selling and buying**
- Item Piles handles the trade itself.
- The module can request an Appraise check, or Diplomacy if your table uses a bargaining house rule, and set a one-off price factor from the result.

**Upkeep**
- Items with fuel or upkeep, such as the Dream tool's "1 Wisp a day while carried", have the cost charged on schedule from the bearer's purse, with the ask-first popup (§2.6).

**Treasure**
- Loot by CR and treasure type (CRB table 12-5 and the GMG's treasure tables), rolled into a list that is then sent to the party's loot actor or an Item Piles pile.
- Optionally tracked against wealth by level.

### 4.3 Settlements

`pf1-endless-settlements`. The GMG settlement stat block (GMG pp. 203–209, extended by Ultimate Campaign's kingdom rules), built and run.

**The model**

| Part | Contents |
|---|---|
| Name, alignment, size | Thorp to metropolis, with the defaults for that size |
| Modifiers | Corruption, Crime, Economy, Law, Lore, Society, each with a breakdown (size, alignment, government, qualities, disadvantages, buildings, events, the GM's own adjustment) |
| Qualities and disadvantages | From a content pack, with their modifiers and other effects (spellcasting levels, base value) |
| Government | Its modifiers |
| Danger | Used for urban encounter chances |
| Demographics | Population, notable NPCs (linked actors), districts |
| Marketplace | Base Value, Purchase Limit, Spellcasting, minor/medium/major item counts |

**Builder**
- Pick a size, alignment, government, qualities and disadvantages, and every number is worked out with its breakdown.
- Ultimate Campaign's settlement grid and buildings are optional: a building adds its modifiers and Base Value.

**Journal integration**
- **Read:** scan journal entries for a settlement stat block, written the way the GMG prints them (`Corruption +2; Crime +2; Economy +5…`, `Base Value 12,800 gp; Purchase Limit 75,000 gp; Spellcasting 8th`). It offers to link each one found to a settlement record, with a diff when they disagree.
- **Write:** create a journal entry (or a page in an existing entry) with the stat block and sections for districts, NPCs and places of interest.
- **Settlement page type:** possibly a custom journal page type (Foundry v13 supports module-defined page subtypes) that renders the block and edits it in place.

**Runner**
- The party's current settlement is set on a scene, or by hand.
- The settlement's modifiers are then offered on the checks they apply to:

| Modifier | Applies to |
|---|---|
| Corruption | Bluff against officials and guards; Stealth outdoors |
| Crime | Sense Motive against Bluff; Sleight of Hand to pick pockets |
| Economy | Craft, Perform, Profession to earn money |
| Law | Intimidate to force friendliness; Diplomacy with officials or to call the guard |
| Lore | Diplomacy to gather information; Knowledge using its libraries |
| Society | Disguise; Diplomacy to change attitudes of people who aren't officials |

- One click requests the check, and the modifier goes with it, labelled with its source.
- **Services:** spellcasting available up to the settlement's level, with service prices (CRB) and maybe NPC casters from the NPC module.
- **Events:** settlement events (UC's kingdom and settlement events) rolled on schedule, logged, and changing modifiers temporarily ("plague: Economy −2 for 1d4 weeks").
- **Danger:** an urban encounter roll on schedule, with tables per district.
- **Notoriety:** optionally, the party's standing *in this settlement*, kept with the place (§2.7), such as wanted posters, Law reacting to past crimes, or Society to fame.

### 4.4 NPCs and Relationships

`pf1-endless-npcs`. The Dream tool's NPC library (D22, D32), generalised and given PF1's social rules.

**The pool**
- Actors from chosen folders (with subfolders), shown in the tool's folders, with search, active and inactive flags, and tags.
- The name, portrait and biography come from the actor. Bonuses are read from the sheet, never stored.

**Creating NPCs**
- Make an actor from a template: an NPC Codex statblock from the compendium, a race plus class levels, or a blank one with a role.
- The tool fills in the biography sections (appearance, personality, wants, secrets) in the actor's notes.

**The social stat block** (Ultimate Intrigue, individual influence)
- Biases, strengths, weaknesses, the influence skills and their DCs, how many successes are needed, and what influencing them gives.
- Discovery checks reveal parts of it. Influence checks are requested with the bonus from a known weakness already added.
- Discovered parts are recorded per party, so the next session remembers what they learned.

**Attitude** (CRB Diplomacy): hostile → helpful, per character or per party, changed by checks and events, and logged.

**Contacts** (Ultimate Campaign, p. 148)
- Trust 1–5 and the risk of a task. Asking a favour checks whether Trust allows it, and a burned contact lowers Trust across your network.
- Monthly decay checks run on the core clock.
- Cultivating a contact asks for the check and, on success, raises Trust.

**Relationships** (Ultimate Campaign, p. 176): a relationship score and its levels, with the benefits each level gives, per PC.

**Leadership** (CRB feat; Ultimate Intrigue's expanded leadership)
- The Leadership score with its modifiers worked out, plus the cohort and followers as linked actors.
- Followers' loyalty and events are tracked.

**Where they are**: an NPC's home settlement, faction memberships, and a simple schedule ("at the docks by day").

**Rolling**: the GM rolls an NPC's checks directly with the requester's `roll` (the Dream tool's D32), with no popup.

### 4.5 Factions

`pf1-endless-factions`. Organisations, the party's place in them, and what they do.

**The faction record** (the Dream tool's `ZoneFaction` as its descriptive core)
- Who they are (leaders, members, where), what they want, what they care about and hate.
- Ties with other factions: allied, rival, or something more specific.
- How to gain and lose standing with them, with amounts.

**Organisational influence** (Ultimate Intrigue, pp. 110–111)
- Influence points per faction, per party or per PC (§9).
- Helping or harming an allied faction counts for its ally as well. Rivals make the party choose a side.
- Opposed organisations act against the party once the party crosses them.
- Thresholds unlock resources (§9: use UI's numbers as written, or our own ladder).
- Optional alternatives: the Faction Guide's Fame and Prestige (spend Prestige on boons, Fame sets your ceiling), or a homebrew ladder like the Dream tool's Standing (−10…15 with named bands).

**Inside a faction**
- Ranks and positions: titles, what each gives, what each requires.
- **Tasks**: missions with a giver, a deadline on the core clock, steps, checks to request, and rewards (influence, items sent through Items, boons). Failure carries consequences.
- **Boons**: granted favours with uses, durations and costs, shown on the character (as a buff where they have a PF1 effect).

**Trackers**: progress clocks for faction goals, war or tension meters between rivals, and an "alarm" track like the Dream tool's Uproar.

**Holdings**: settlements and districts they control (with Settlements, a faction can modify a settlement's numbers), merchants they run (prices by standing), and NPC members (from NPCs).

**Relations view**: a graph of factions and their ties, using the same graph component as the Dream tool's plans (React Flow). The GM can hide it, or show players only the ties they know of (the Dream tool's players' view, D21).

### 4.6 Research

`pf1-endless-research`. Ultimate Intrigue's research rules.

**A library or repository**
- Its Complexity (the check DC), its research thresholds, and the knowledge points needed to reach each one.
- Which skills it allows (Knowledge, Linguistics, or a custom one).
- What each threshold reveals.

**Reveals are journal content**
- Each threshold points at a journal page (or a section of one) that is hidden from players until reached, then shown to them, with a chat card ("You find…").
- The GM writes the research the way they'd write any journal, and the module only controls what's visible when.

**Running it**
- A research session requests the check from whoever researches. Aid another and the settlement's Lore modifier, with Settlements, go into the request.
- Each success adds knowledge points, and 8 hours pass on the core clock.
- Failures by 5 or more can trigger the library's complications (rolled on its table).

**Several libraries per topic**: with Settlements, a library is a place in a settlement, and the settlement's Lore modifier applies.

The Dream tool's reveals with a check (D37) and its Casebook (clues tallied toward Truths) are the same mechanism in a different form.

## 5. More module ideas

Each could be its own module, or a part of one above.

| Idea | PF1 basis | What it would do |
|---|---|---|
| **Downtime** | Ultimate Campaign, Downtime | Capital (Goods, Influence, Labour, Magic), rooms, buildings, teams and organisations. Earnings checks requested each day, with Economy from Settlements. A party ledger on the core clock. Likely the module the others feed most. |
| **Kingdom and mass combat** | Ultimate Campaign, Kingdom Building, Mass Combat | Kingdom turns: upkeep, edicts, Unrest, Stability/Economy/Loyalty checks via the requester (leaders as actors). Its settlements are the Settlements module's, with buildings. Armies are actor-like records. |
| **Infiltration and disguise** | Ultimate Intrigue, Infiltration | Suspicion and infiltration points per location, cover identities, and the checks that move them, like the Dream tool's Cover track in its Locked Palaces. |
| **Verbal duels** | Ultimate Intrigue, Verbal Duels | A duel tracker: exchanges, tactics, biases learned by assessing the audience, determination. Both sides' checks requested, and the edges applied. |
| **Pursuits and chases** | Ultimate Intrigue, Pursuit; GMG, Chases | A chase card track or pursuit days, with obstacles as checks, edges, and who's ahead. |
| **Travel and exploration** | Ultimate Wilderness, Exploration; CRB overland movement; GMG weather | Hexcrawl or route travel with watches and legs, encounter rolls, forced march, starvation and thirst, and weather rolled each day. This is the Dream tool's watches, legs and plan graph with ordinary terrain. |
| **Hazard zones** | CRB environment; Planar Adventures plane traits; haunts | The Dream tool's danger-zone engine without the dream: tracks, clocks, intervals and checks for hostile environments (a plane's traits, a cursed forest, a radiation zone in a homebrew setting). The Dream Danger Zones tool becomes its biggest content pack. |
| **Investigations** | GMG mysteries; Ultimate Intrigue | A casebook: clues found, what they point to, theories, and a "the party knows" view (the Dream tool's Casebook, generalised). |
| **Crafting and spell research** | CRB magic item creation; Ultimate Magic spell research; Ultimate Campaign downtime | Projects with cost, days, Spellcraft or Craft checks requested, progress on the clock, and the item made through Items when done. |
| **Honor, fame and reputation** | Ultimate Campaign, Honor; Faction Guide, Fame | Per-PC tracks with events, shown on the sheet, readable by Factions and Settlements. |
| **Campaign map and plans** | Ultimate Intrigue, intrigue themes | The Dream tool's branching plans and maps (D19, D21): a campaign flowchart, known and unknown nodes, a players' view, nodes that open settlements, factions, journals or scenes. |
| **Rumours and gossip** | GMG; Ultimate Intrigue | Rumour tables per settlement or faction, gathered with Diplomacy (Lore added), each true, false or partial, and each able to reveal a map node or a research threshold (the Dream tool's D37). |
| **Session log and summary** | n/a | Every module's events in one timeline, an end-of-night summary (Markdown or a journal page), and "what changed" written back to actors (the Dream tool's M7). |
| **Party sheet** | n/a | The party's shared purse and loot actor, boons, contacts, faction standings, and current settlement on one screen. |

## 6. What to take from the Dream Danger Zones tool

| Dream tool part | Generalised as | Goes to |
|---|---|---|
| Event-sourced sessions, undo, setup vs story (§5.2, D33) | The event log kit | Core |
| Known modifiers and `uncoveredModifiers` (D9, D29) | The modifier model and the check bridge | Core, Check Requester |
| Check fluff and the GM's note (D29) | Request text templates: where you are, what's happening; the rule and DC for the GM | Core |
| Interval scheduler, ▶ Next stopping at what's due (§5.2.1) | The core clock | Core |
| Charges asked first (D30) | Costs and gains: automatic, ask, note only | Core |
| World-folder storage, mirrored with the browser (§11.6) | Storage | Core |
| Rules data parsed from Markdown, rules links and the popup (D5, D18) | Content packs and rules links | Core |
| Field declarations, so the UI renders what modules declare (D13) | The UI kit's declared forms | Core |
| Tracks with bands, clocks with thresholds, counters (§4.1) | Track, clock and counter primitives | Core (used by Factions, Settlements, Infiltration, Hazard zones) |
| Places remember (D24) | Per-place state | Core (used by Settlements, Factions, NPCs) |
| NPC library: folders, active, Foundry-linked, GM rolls (D22, D32) | NPCs and Relationships | NPCs |
| Factions written out (D38), Courts' Standing, Favour, Uproar | Faction records, standing ladders, alarm tracks | Factions |
| Items, lists, Item Piles, prices in Dreams (D39) | Items and Merchants, item templates, currency systems | Items |
| Loot classification, icons matched against compendiums (D26) | The icon matcher and item kinds | Items |
| The purse in step both ways (D31) | Party purse sync with any container or Item Piles actor | Items, Party sheet |
| Branching plans, maps, known and unknown, players' view (D19, D21) | The campaign map | Campaign map |
| Reveals with a check (D37), the Casebook | Gated reveals of journal content | Research, Investigations, Rumours |
| Zone modules, region cores (§4.4) | The hazard-zone engine | Hazard zones |
| Summary and write-back (M7) | Session log and summary | Session log |
| Character sheets popup (D25) | The party sheet | Party sheet |

**The Dream tool's future.** The Dream tool could become the first content pack of the Hazard zones engine plus the Dimension of Dreams item and currency pack. It would then lose most of its own plumbing. That's a large refactor, worth doing only once the core exists and has been proven by one or two smaller modules (§8).

## 7. How the modules connect

```
                     pf1-endless-core
   registry · events · storage · modifiers · clock · journals · check bridge · UI kit
        │            │             │            │            │            │
  Check Requester   Items      Settlements     NPCs       Factions     Research
     (rolls)    (+Item Piles)  (+journals)   (+actors)   (+journals)  (+journals)

  Every module asks for checks through the Check Requester (via the core's check bridge).
  The other modules talk to each other only through the registry, and only when both are active.
```

Examples of features that only appear when two modules meet:

| Together | What it adds |
|---|---|
| Settlements + Items | Merchant stock from the marketplace numbers, and restocks |
| Settlements + Check Requester | Settlement modifiers on requested checks |
| Settlements + Factions | A faction controls a settlement or district, and its standing shifts the numbers |
| Factions + Items | Prices by standing; boons that grant items |
| NPCs + Factions | Members, leaders, and who gives tasks |
| NPCs + Settlements | Notable NPCs, casters for spellcasting services |
| Research + Settlements | The Lore modifier, and libraries as places in a settlement |
| Anything + the core clock | Restocks, upkeep, Trust decay, task deadlines, events |

## 8. A possible build order

1. **Core, minimally**: the registry, storage, the modifier model, the check bridge, journal stat-block parsing, and the UI kit. Extract it from the Dream tool's code and the Check Requester's API rather than writing it fresh.
2. **Settlements**: small enough to finish, and visible at the table at once. It proves the journal integration and modifier passing, and it's the hub the others plug into.
3. **Items and Merchants**: much of it already exists in the Dream tool. Generalise the item templates and currencies, then add stock from Settlements.
4. **NPCs**: the Dream tool's NPC pool, plus attitudes, contacts and the social stat block.
5. **Factions**: builds on NPCs and Settlements.
6. **Research**: small, and depends only on the core and journals.
7. **Downtime and the rest** as wanted. **The Dream tool's migration** comes last.

The Check Requester stays its own module throughout. The core wraps it, and it never depends on the core, so it keeps working on its own exactly as it does now.

## 9. Open questions

- **Packaging**: one repository (this one) holding every module, built into separate zips, or a repository per module? One repository makes shared code and versions simple, and Foundry installs each module separately anyway.
- **Foundry version**: v13 only (like the Check Requester), or v12 too?
- **Settlement storage**: should the settlement *record* be a journal page with a custom type (visible and editable in Foundry, synced to every client) or a file with a link to a journal (lighter on traffic)? Settlements change rarely, so a journal page type may be fine here, unlike the Dream tool's sessions.
- **Influence per party or per PC**: Ultimate Intrigue mostly assumes the party shares influence, with notes on divided parties. Which should be the default?
- **Organisational influence as written, or modified**: Ultimate Intrigue's rules are thin in places. Should we keep its numbers, or define our own ladder (like the Dream tool's Standing bands) and keep UI's as an option?
- **Player-facing views**: the Check Requester is GM-driven. Should players get read-only windows (their contacts, standings, known research), or only journal pages the modules keep up to date?
- **Calendar**: depend on Simple Calendar or Calendaria, or keep our own clock that can follow one?
- **Content licensing**: PF1 rules text can be used under the OGL / ORC with attribution, but the Dream tool's setting content is the user's own. Content packs should keep the two apart.
