# Rulealize.RuleSet.Exchange

Two parties, each offering something, and **neither committed until both are** — a
[Rulealize](https://github.com/reny-develop/Rulealize) rule set, shipped with the half it is
built from.

| | |
| --- | --- |
| Rule set id | `Rulealize.RuleSet.Exchange` |
| Package | [`Rulealize.RuleSet.Exchange`](https://www.nuget.org/packages/Rulealize.RuleSet.Exchange) |
| Ships with | `Rulealize.RuleSet.Exchange.Side` |
| Inputs | none of its own — `ours.*` and `theirs.*` |
| Draws on | State, Comparison, Record, Logic (and TypeSchema, Binding inside `Side`) |

```json
"uses": [
  { "ruleSet": "Rulealize.RuleSet.Exchange", "version": "^1.0", "as": "swap" }
]
```

`rulealize restore` fetches this package and gets **both** documents, because the entry point
holds the other and fetching only the entry point would fetch something that cannot compile.
**`as` is not optional** — an alias defaults to the identifier and may not contain a `.`.

## One package, two documents

`Exchange.Side` is one party's half: what they may offer, what they did, and whether they have
committed. `Exchange` holds **two of them** and adds the guards that neither half can express.

```
Rulealize.RuleSet.Exchange.1.0.0.nupkg
└── ruleset/
    ├── exchange.json   ← "id": "Rulealize.RuleSet.Exchange"
    └── side.json       ← "id": "Rulealize.RuleSet.Exchange.Side"
```

**`Side` is not published separately, and should not be.** It is meaningless on its own — a
side of nothing — so a package of its own would be a permanent public name for an
implementation detail. It ships here instead, named **under** the package that ships it, which
is what keeps two packages from ever shipping one identifier.

That is the same arrangement a library has: the type you call is the package's name, and the
types it is built from come along without being products.

## The guards

Neither half knows the other exists. `Side` has never heard of a counterparty; it can say *I
have offered* and *I have committed* and nothing about anybody else. Everything below is in
`Exchange`, over both states at once:

| | |
| --- | --- |
| **Nobody commits alone** | `confirm` waits for the other side to have offered something |
| **Nobody is left holding it** | once one side has confirmed, the other may no longer withdraw |

```json
"held": {
  "ours": {
    "confirm":  { "when": { "op": "cmp.ne", "right": "empty",
                            "left": { "op": "rec.at", "record": "$theirs", "key": "stage" } } },
    "withdraw": { "when": { "op": "cmp.ne", "right": "confirmed",
                            "left": { "op": "rec.at", "record": "$theirs", "key": "stage" } } }
  },
  "theirs": { … the same, mirrored … }
}
```

## Walking it

```
nothing offered      ours.offer(item: deed)   ours.offer(item: keys)
                     theirs.offer(item: payment)
                     ← no confirm anywhere

we have offered      ours.withdraw   theirs.offer(item: payment)
                     ← still no ours.confirm: they have offered nothing

both offered         ours.withdraw   ours.confirm
                     theirs.withdraw theirs.confirm

we confirmed         theirs.confirm
                     ← theirs.withdraw is gone, and so is ours

both confirmed       terminal (exchanged)
```

The third and fourth steps are the whole point. Until somebody commits, either side may walk
away; after somebody commits, the other cannot leave them holding it.

## The instance

Both sides' state is a field of this one's, so a case is still one document:

```json
{
  "$schema": "rulealize/state/v1",
  "ruleSet": "Rulealize.RuleSet.Exchange@1.0.0",

  "data": {
    "ours":   { "data": { "offers": ["deed", "keys"], "offered": null, "stage": "empty" } },
    "theirs": { "data": { "offers": ["payment"],      "offered": null, "stage": "empty" } }
  }
}
```

What either side may offer is here and in neither rule set. A conveyance, a trade, a swap of
signed documents, a prisoner exchange — same documents, different state.

## Where it fits

| The sides are | and the exchange is |
| --- | --- |
| a seller and a buyer | deed against payment |
| two teams | a handover neither wants to do first |
| two systems | a two-phase commit somebody has to model |
| two parties to a settlement | signatures against release |

## Trying it

Needs [Rulealize.Cli](https://github.com/reny-develop/Rulealize.Cli) 0.9.0 or later:

```sh
dotnet tool install -g Rulealize.Cli
rulealize restore src/Rulealize.RuleSet.Exchange/ruleset/exchange.json
rulealize play    src/Rulealize.RuleSet.Exchange/ruleset/exchange.json --state state/example.json
```

Nothing is fetched: both documents are here, and `Side` is found beside the one that holds it.
A `component/` folder appears only when a document holds something published elsewhere.

## Building the package

```sh
dotnet pack src/Rulealize.RuleSet.Exchange -c Release
```

The project file globs `ruleset/*.json`, which is safe **here** because both documents are
ours and `restore` writes fetched ones into `component/` instead. A repository whose rule set
holds something published elsewhere should name what it packs rather than glob it — see
[what to build](https://github.com/reny-develop/Rulealize.Registry/blob/main/doc/publish.md#if-your-rule-set-holds-others).

## License

Apache-2.0, and it covers both documents as much as everything else here.
