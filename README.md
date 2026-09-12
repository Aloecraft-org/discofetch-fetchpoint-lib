# discofetch-fetchpoint-lib

Serving a FetchPoint: the verbs, the rooms, the rendezvous reads. Extracted
from `api/supervisor.lua` (the serving banner at 2366-2380, the rooms tier at
2381-2490, and `serve_fetchpoint` at 2535-3129) without changing arithmetic,
branch order or refusal behaviour.

This is the piece a self-hoster needs: the code that decides which FetchPoint
is being addressed, whether the caller may do what they are asking, and what
an address that resolves to nothing says.

**The Host header SELECTS a FetchPoint and confers NOTHING.** Nothing upstream
verifies it, so every authority question inside a fetchpoint is answered by its
own config and its own tokens, never by the name used to arrive. This library
never parses a Host: it is handed a label and a verb by the dispatcher.

## Surface

### Entry points

```
M.new(deps)                                   -> instance
M.valid_room(name)                            pure
M.ice_defaults(zone)                          pure

inst:serve_fetchpoint(conn, req, label, verb) the whole door

inst:room_prune(fp_id, room, ttl, t)
inst:room_list(fp_id, room, ttl, t)               -> newest-first list
inst:room_join(fp_id, room, ttl, cap, cand, t)    -> list | nil, code, msg
inst:room_get(fp_id, room, ttl, key, t)           -> entry | nil
inst:room_entry_json(c)                           -> the wire shape, as text
inst:room_allowed(cfg, req)                       -> boolean
inst:rooms_count()                                -> count, cap

inst:traffic_bump(fp_id, field)
inst:traffic_of(fp_id)                            -> counters (zeros, never nil)
inst:traffic_totals()                             -> totals, names
inst:traffic_each()                               -> pairs() over the live rows
```

`rooms_count()` answers the count AND the cap it is racing, because a number
without its cap is not worth printing; `GET /v1/admin/stats` reads both.
`traffic_of` / `traffic_totals` / `traffic_each` exist because the meter's
readers (the owner's traffic page, the admin traffic page, the stats card) live
outside this library while every one of its twelve write sites lives inside it.

### Configurable values

| name | ships as | what it bounds |
| --- | --- | --- |
| `M.ROOMS_MAX_PER_FP` | 256 | distinct rooms one fetchpoint may hold |
| `M.ROOMS_GLOBAL_MAX` | 100000 | candidates across everything; a memory backstop |
| `M.LIMITS.room_ttl` | 300 | default TTL when the config says nothing |
| `M.LIMITS.room_capacity` | 16 | default members per room |
| `M.LIMITS` (rest) | - | port range, candidate name, ufrag/pwd, wg key, rtt |
| `M.ice_defaults(zone)` | - | STUN/TURN URLs, vantages, TURN TTL, candidate caps |
| `M.TRAFFIC_FIELDS` | - | the ten counters, spelled once for the bump and the totals |

`M.ice_defaults(zone)` returns a **fresh** table per call, so two instances can
never share one mutable default. `VANTAGES` is the one field a self-hoster must
replace by hand: everything else follows the zone, those are two specific
edges' addresses.

### Fan-out points

- `M.ROOM_AUTH` - room policy, per the owner's config: `open`, `pin`, `key`.
  A fourth policy is a registration (`deps.room_auth`), not a new `elseif`, and
  the registry is per instance. An unrecognised mode falls back to `key`, never
  to `open`.
- `M.VERB` / `M.VERB_DEFAULT` - the `<label>--<verb>` dispatch, by method name.
  The kind registry gates which verbs a row answers at all (404 `no_verb`);
  this table says which code serves the ones that get through. Anything the
  registry lists that is not `dns` / `ice` / `outcome` is a join, which is how
  it has always behaved.
- The bare-name answer is a **matrix, not a dispatch** - four outcomes over two
  dimensions, and the order of the tests is load-bearing:
  1. `report` + `caller` -> `serve_reflect`
  2. (`report` + `self`) or (`redirect` with no target) -> `serve_rendezvous`
  3. `redirect` (with a target) -> `serve_static_redirect`
  4. anything else -> 501 `kind_not_implemented`

  A dispatch table keyed on `behavior` would send a `report` row with a NULL
  `subject` to the rendezvous answer; today it falls to the 501. The four
  handlers are named functions and the chain is intact.

## The injected-deps contract

Everything that reaches the host, the database, the clock or another library in
this set arrives through `M.new`. A missing or wrong-typed dep is a **named
failure at construction** - never a nil dereference three branches into a
request that a self-hoster then reports as "rooms are broken".

| dep | shape | why |
| --- | --- | --- |
| `now` | `function -> unix SECONDS` | never `host.time()`, which answers milliseconds; a 1000x error in every rate |
| `ready` | `function -> ok, error_sentence` | a PREDICATE. `db_ready` flips after `migrate()`, so a boolean captured at construction freezes `false` |
| `db` | the drt sql connector handle | the fetchpoint lookup and the advertisement read |
| `rows_by_name` | `function(result) -> rows` | drt-db-lib; nothing here indexes a result positionally |
| `json` | `encode` / `decode` | the module calls no global |
| `http` | drt-http-api-lib instance | `reply`, `fail` (called with `:`), `observed_address`, `observed_port`, `public_forwarded` (called with `.` - they are pure) |
| `model` | discofetch-model-lib | `KINDS`, `DIM.of`, `valid_ip` |
| `db_ops` | discofetch-db-lib instance | `advertise(fp_id, addr, t) -> at, changed` |
| `rate` | token-rate-limit-lib instance | `check(key, policy, what)`, `headers(wait)`, `POLICIES.authfail` |
| `tier` | `tier_of(name) -> tier row` | the OWNER's tier: whose name it is decides how fast it may be exercised, not who is calling |
| `zone` | string | deployment config, e.g. `discofetch.link` |
| `turn_credential` | optional `function(label, ttl)` | absent, a `turn=true` room answers the same 503 `turn_unavailable` a failed mint answers |
| `ice` | optional table | overrides merged field-by-field over `M.ice_defaults(zone)`; supplying `STUN_HOSTS` re-derives `STUN_URLS` |
| `room_auth` | optional table | extra room policies, merged over `M.ROOM_AUTH` |
| `authfail` | optional | an override for `rate.POLICIES.authfail`, for a limiter that does not re-export its policies |

`deps.rate:check(key, policy, what)` returns `nil` when the call is allowed, or
a refusal table `{status, code, message, headers}` when it is not. The message
convention is `<what was spent>; retry in Ns`, composed by the limiter. The
caller - this library - decides what to do with the refusal, which is why the
traffic bumps stay at the call sites: `denied` fires before the authfail check,
`rate_limited` only after a real bucket says no, and three of the nine refusal
sites bump nothing at all.

`policy` is a `{per_hour, burst}` pair. The key derivations stay here, including
the `* 60` on the join verb that converts a per-MINUTE tier field into the
bucket's per-hour rate while the sentence the caller reads still says per minute.

**Not injected, on purpose:** the traffic meter is this library's - it has no
other writer - and `BOOT_AT` stays in the composition root, because a
process-lifetime fact is not the meter's to own.

## Usage

```lua
local fetchpoint = require('discofetch-fetchpoint-lib')  -- when DRT ships require

local fp = fetchpoint.new({
  now   = now,                       -- unix SECONDS
  ready = function() return db_ready, db_error end,
  db    = db,
  rows_by_name = drtdb.rows_by_name,
  json  = json,
  http  = http,                      -- drt-http-api-lib instance
  model = model,                     -- discofetch-model-lib
  db_ops = dfdb,                     -- discofetch-db-lib instance
  rate  = limiter,                   -- token-rate-limit-lib instance
  tier  = accounts.tier_of,
  zone  = 'discofetch.link',
  turn_credential = host.crypto.turn_credential,
  ice   = { VANTAGES = { '198.51.100.10', '203.0.113.7' } },
})

-- in the dispatcher, where fetchpoint_host() has already split the Host:
local label, verb = model.fetchpoint_host(req.host)
if label then
  local okp, perr = pcall(fp.serve_fetchpoint, fp, m.conn, req, label, verb)
  ...
end

-- the readers that live outside:
local rooms, rooms_max = fp:rooms_count()
local totals, names    = fp:traffic_totals()
```

## Dependency edges

This library sits near the top of the stack and imports downward only:

```
discofetch-fetchpoint-lib
  -> drt-http-api-lib        reply, fail, observed_address, observed_port,
                             public_forwarded
  -> drt-db-lib              rows_by_name
  -> discofetch-model-lib    KINDS, DIM.of, valid_ip
  -> discofetch-db-lib       advertise (+ its ADVERTISE_REFRESH staleness bound)
  -> token-rate-limit-lib    check, headers, POLICIES.authfail
  -> discofetch-accounts-lib tier_of
```

Nothing in this set depends on this library. Symbols this library uses but does
not own, per the ownership ruling - a copy is how two libraries drift:

- `advertise` + `ADVERTISE_REFRESH` belong to **discofetch-db-lib**. One writer
  for both doors (the dns verb and the panel's PUT), and the 300-second
  staleness bound is a page-write argument about the sqlite file, not a
  fetchpoint argument. The dns verb calls it and reports the `at` the ROW now
  holds, never this request's, so the answer never claims a freshness the
  readers will not see.
- The `authfail` 30/10 tuple belongs to **token-rate-limit-lib**. It appears at
  four sites in the original; naming it in the limiter is what lets a test
  assert the number the brute-force argument rests on. This library keeps only
  the key derivation (`authfail:<fp_id>`) and the decision to spend the bucket
  on failures ONLY - so a guesser can never lock a right pin out, because the
  right pin never gets here.
- `valid_ip`, `KINDS`, `DIM` belong to **discofetch-model-lib**; `tier_of` and
  `TIERS` to **discofetch-accounts-lib**; `rows_by_name` to **drt-db-lib**.

## Consumption waits on `require`

Guests in DRT have no `require` and no `dofile` today; the load-time modules
slice is designed but unshipped. Nothing consumes this file yet, and that is
expected. Until then, `test/run.sh` wraps the module in an IIFE and
concatenates `test/cases.dlua` after it - when `require` lands, the runner
becomes two `require` lines and `supervisor.lua` drops its copy of these
regions in favour of `fetchpoint.new(...)`.

```
$ sh test/run.sh
...
PASS
```

216 assertions. A run is green only if the last line is exactly `PASS`.

## What changed in the move, and what did not

Nothing about arithmetic, branch order or refusal behaviour changed. What did
change is shape, and only where the ruling required it:

- `room_get` and `room_entry_json` moved to TOP LEVEL. They were nested inside
  `serve_fetchpoint` only because the main chunk was at Lua's 200-local limit -
  a constraint that does not exist behind a module table.
- `room_allowed`'s `if/elseif` chain became the `ROOM_AUTH` table, so room
  policy stays open to extension. It now returns a strict boolean; the three
  shipped policies already did.
- The verb chain became the `VERB` dispatch table. `ice` and `outcome` share one
  method because they share a gate.
- The `TRAFFIC` counter list, spelled twice in the original (once in the bump,
  once in the totals), is spelled once as `TRAFFIC_FIELDS`.
- Constants that sat at first use (`300`, `16`, `32`, `4`/`256`, `22`, `44`,
  `60000`, the port range) are hoisted into `M.LIMITS` and interpolated into the
  refusal sentences they appear in, so a number and the message that quotes it
  cannot drift. Every sentence is byte-identical to the original.
- The traffic meter's proposed injection slot is **deleted** - the meter is this
  library's own.
- The supervisor comment about `observed_address` living "up in the helpers"
  and the one about the two room helpers being "scoped here" did not travel:
  both are about upvalue ordering in one file and are meaningless behind a
  module table.

## Known, carried over

Behaviour I believe is wrong or surprising and did **not** change:

1. **The unnamed room is not gated by `config.rooms`.** Both the join verb and
   the room read guard `rooms_disabled` and `valid_room` with `if room ~= ''`,
   so a request with no `?room=` (or `?room=`) joins and reads the room whose
   name is the empty string - on a fetchpoint that has rooms switched off.
   Candidates therefore accumulate in memory for a name whose owner never
   enabled rooms. The two doors agree with each other, so the behaviour is at
   least consistent; it is carried as-is.
2. **`serve_reflect` dereferences `req.query` unguarded** (`next(req.query)`)
   while the dns verb guards it (`req.query and req.query.key`). A request
   arriving with no query table at all reaches a nil index on the reflect path.
   The dispatcher always supplies one today.
3. **`rtt_ms` is range-checked but not integer-checked**, where every port in
   the same file is checked with `% 1 ~= 0`. `?rtt_ms=12.5` is recorded.
4. **A shortened `room_ttl` retroactively expires members.** The TTL arrives
   per request from the config, and `room_prune` applies it to entries that
   joined under the old one; lowering it drops live members on the next touch.
5. **The 501 answer carries no cache headers**, where every other bare-name
   answer carries `PUBLIC_READ` or `NO_STORE`.
6. `room_full`'s sentence interpolates `cfg.room_capacity` directly, so a
   non-numeric capacity in a config would reach the caller in the message.

## Unresolved

- **The fetchpoints CRUD handlers** (`fetchpoints_create` / `_update` /
  `_delete` / `_list` and `advertisement_put` / `advertisement_delete`, roughly
  `supervisor.lua` 3296-4050) are assigned to this library by the ownership
  ruling's orphans section, but sit outside the regions this extraction pass was
  scoped to (2366-3129). They are not in this repo. They are the only callers of
  `claim_label`, `fp_row`, `config_view`, `secret_fields`, `REDACTED` and
  `same_config`, so until they move, six of discofetch-model-lib's exports have
  no library-side caller.
- **`db` + `rows_by_name` versus named ops.** The ruling's edge list says both
  that this library depends on drt-db-lib for `rows_by_name` at the two SELECT
  sites AND that the fetchpoint lookup and the advertisement read are named ops
  on discofetch-db-lib. This pass took the first reading: the two SELECTs are
  extracted verbatim, because "extract, do not rewrite" is the stronger
  instruction and inventing two named ops would have asserted an API in another
  repo. Moving them behind `db_ops` later removes `db` and `rows_by_name` from
  this library's deps entirely and is a pure win; it is not this pass's call.
- **The sibling APIs are written against the ruling, not against shipped code.**
  `rate:check` returning a refusal as data, `db_ops:advertise` returning
  `at, changed`, and the split between `http:reply`/`http:fail` (instance
  methods) and `http.observed_*` (pure functions) are all specified here and
  asserted in the tests through doubles. If a sibling repo lands a different
  shape, this library's `M.new` will say which dep is wrong, by name.
