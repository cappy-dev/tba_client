# tba_client

A typed Dart client for [The Blue Alliance](https://www.thebluealliance.com) API v3. Pure Dart, so it works in Flutter apps, CLIs, and servers alike.

```dart
import 'package:tba_client/tba_client.dart';

final client = TbaClient(config: InMemoryTbaConfig('your-tba-key'));
final team = await client.getTeam(1234);
final matches = await client.getEventMatches('2026txhou');
```

Covers teams, team avatars (media), events, event team lists, match schedules, plain OPR/DPR/CCWM, component OPR (COPRS) breakdowns, qualification rankings, playoff alliances, event awards, and TBA's own match predictions, decoded into plain Dart models. The `TbaConfig` seam decides where the `X-TBA-Auth-Key` comes from: `CompileTimeTbaConfig` reads a `--dart-define=TBA_API_KEY`, `InMemoryTbaConfig` holds one directly, and your app can implement the interface to resolve keys from anywhere (the source app chains a Firestore-stored team key). A missing key throws `TbaApiKeyMissingException` before any request goes out.

## API key resolution

`TbaClient` needs a TBA auth key on every request (the `X-TBA-Auth-Key` header, preferred over the query-string form so CDN caching stays intact). Three out-of-the-box options, all injectable through `TbaConfig`:

- `CompileTimeTbaConfig` - reads `String.fromEnvironment('TBA_API_KEY')`, set via `--dart-define=TBA_API_KEY=...` or `--dart-define-from-file=tba.env`. The default for the source app.
- `InMemoryTbaConfig` - holds a key in memory; handy for tests and quick scripts. Pass `--define=TBA_API_KEY=` (or omit) to represent an empty key.
- Custom - implement `TbaConfig` yourself to resolve keys from a remote store, user settings, or a secrets manager.

```dart
final client = TbaClient(config: CompileTimeTbaConfig());
```

## API reference

`TbaClient` targets `/api/v3` on `www.thebluealliance.com`. List endpoints return an empty list on 404; single-object endpoints return `null` on 404. Some event sub-resources (`getEventRankings`, `getEventAlliances`, `getEventAwards`, `getEventCoprs`, `getEventOprs`) also return `null` for a normal pre-event state (no rankings yet, no alliance selection, no awards ceremony), so a null is not an error. `getEventPredictions` instead returns an empty map both on 404 and while an event has nothing to predict, which is the normal state early at an event and the permanent state at an offseason one. Anything else outside 2xx throws `TbaApiException`. `getStatus` treats 404 as a hard error so you can tell a misconfigured base URL / bad key apart from a normal "not found".

| Method | Endpoint | Returns |
| --- | --- | --- |
| `getStatus()` | `GET /status` | `TbaApiStatus` |
| `getTeam(int teamNumber)` | `GET /team/frc{n}` | `TbaTeam?` |
| `getEventTeams(String eventKey)` | `GET /event/{key}/teams/simple` | `List<TbaTeam>` |
| `fetchTeamAvatar(int teamNumber, int year)` | `GET /team/frc{n}/media/{year}` | `Uint8List?` (PNG bytes) |
| `getEvent(String eventKey)` | `GET /event/{key}` | `TbaEvent?` |
| `getEventsForYear(int year)` | `GET /events/{year}` | `List<TbaEvent>` |
| `getEventMatches(String eventKey)` | `GET /event/{key}/matches/simple` | `List<TbaScheduleMatch>` |
| `getEventMatchesDetailed(String eventKey)` | `GET /event/{key}/matches` | `List<TbaScheduleMatch>` |
| `getEventOprs(String eventKey)` | `GET /event/{key}/oprs` | `TbaEventOprs?` |
| `getEventCoprs(String eventKey)` | `GET /event/{key}/coprs` | `TbaEventCoprs?` |
| `getEventRankings(String eventKey)` | `GET /event/{key}/rankings` | `TbaEventRankings?` |
| `getEventAlliances(String eventKey)` | `GET /event/{key}/alliances` | `TbaEventAlliances?` |
| `getEventAwards(String eventKey)` | `GET /event/{key}/awards` | `TbaEventAwards?` |
| `getEventPredictions(String eventKey)` | `GET /event/{key}/predictions` | `Map<String, TbaMatchPrediction>` |
| `getMatch(String matchKey)` | `GET /match/{key}` | `TbaMatch?` |

Examples:

```dart
// Team basics
final team = await client.getTeam(254);
print('${team?.teamNumber}: ${team?.nickname} (${team?.displayLocation})');

// Event schedule, alliances resolved to team numbers
final matches = await client.getEventMatches('2026cmptx');
for (final m in matches) {
  print('${m.key} red=${m.redTeams} blue=${m.blueTeams}');
}

// Event COPRS breakdown (component OPRs). The payload is stat major:
// outer key is the stat name, inner keys are team keys.
final coprs = await client.getEventCoprs('2026cmptx');
if (coprs != null) {
  final foulsFor254 = coprs['foulPoints']?['frc254'];
  print('Foul points for frc254: ${foulsFor254 ?? 'N/A'}');

  // Every component stat this event reports for one team.
  final teamStats = coprs.forTeam('frc254');
  print('Component stats for frc254: ${teamStats.keys.join(', ')}');
}

// Plain OPR, DPR and CCWM live in a separate payload (not in COPRS).
final oprs = await client.getEventOprs('2026cmptx');
if (oprs != null) {
  print('OPR for frc254: ${oprs.oprs['frc254'] ?? 'N/A'}');
}

// TBA's own predicted scores per match, merged across qual and playoff
// and keyed by match key. Empty until the event has enough data.
final predictions = await client.getEventPredictions('2026cmptx');
final? qm1 = predictions['2026cmptx_qm1'];
if (qm1 != null) {
  print('qm1: red ${qm1.redScore} vs blue ${qm1.blueScore} '
      '(TBA picks ${qm1.winningAlliance} at ${qm1.probability})');
}
```

### Models

- `TbaTeam` - `key`, `teamNumber`, `nickname`, `name`, and nullable `city` / `stateProv` / `country`. `displayLocation` joins the non-empty location parts with commas.
- `TbaEvent` - `key`, `name`, `year`, and optional `week` (TBA weeks are zero-based; this model offsets to one-based), `country`, `stateProv`, `startDate`, `endDate`.
- `TbaScheduleMatch` - `key`, `compLevel`, `matchNumber`, and `redTeams` / `blueTeams` as plain `int` team numbers (non-`frc`-prefixed keys are dropped). Missing `comp_level` defaults to `'qm'`.
- `TbaEventCoprs` - component OPR breakdown. `eventKey` plus a `stats` map that is **stat major**: the outer keys are stat names and the inner keys are team keys (for example `{"foulPoints": {"frc254": 4.5}}`). Stat names vary per game year and mix human-readable labels (`Total Coral Points`) with raw camelCase (`teleopCoralPoints`), so the model carries an open map rather than named fields. `operator [](statName)` fetches a team-keyed column; `forTeam(teamKey)` returns every stat that team has as a map; `statNames` lists them; `isEmpty` reports whether any stats are present. Entries with non-numeric values are skipped individually. This endpoint carries **component** OPRs only: plain OPR, DPR and CCWM are not in this payload, use [TbaEventOprs] for those.
- `TbaEventOprs` - plain OPR, DPR and CCWM per team for an event, as three team-keyed maps (`oprs`, `dprs`, `ccwms`). Separate from `TbaEventCoprs` because TBA serves them separately and the COPRS payload has no OPR in it. `isEmpty` reports whether every section came back empty.
- `TbaEventRankings` - the qualification ranking table. `eventKey` plus `rankings` (one `TbaTeamRanking` per team in rank order) and `sortOrderNames`, the column names the payload pairs with each row's `sortOrders`. Those names are game-specific and change every season, so they are read from the payload rather than hardcoded. `sortOrdersFor(ranking)` pairs a row's values with its names for a table; extra values with no matching name are dropped. `isEmpty` reports whether any rows are present. `TbaTeamRanking` carries `teamKey`, `rank`, `teamNumber`, `wins`, `losses`, `ties`, `qualScore`, and `sortOrders` (positional).
- `TbaEventAlliances` - playoff alliances in pick order. `eventKey` plus an `alliances` list of `TbaAlliance`. The order is preserved exactly as returned and never sorted: `picks` is team keys in pick order (captain first), `captain` is the first pick (or null when empty), `status` is how far the alliance got (e.g. `f`, `sf`, or empty), and `record` is the playoff record as `wins-losses-ties`. `isEmpty` reports whether any alliances are present.
- `TbaEventAwards` - awards presented at an event. `eventKey` plus an `awards` list of `TbaAward`. Each `TbaAward` carries `name`, `awardType`, and `recipients` (`TbaAwardRecipient`), where a recipient may be a team, a person, or both, so team awards can be told apart from individual ones. `forTeam(teamKey)` returns every award that team received. `isEmpty` reports whether any awards are present.
- `TbaMatch` - `key` and a `List<TbaMatchVideo>`. `youtubeVideo` returns the first YouTube entry; `TbaMatchVideo.youtubeUrl` builds the watch URL.
- `TbaMatchPrediction` - one row of TBA's own predicted outcome for a match: `matchKey`, the two predicted `redScore` / `blueScore`, `winningAlliance` (`red`, `blue`, or empty when the payload does not say), and `probability` (TBA's confidence in `winningAlliance`, 0 to 1, so always at least 0.5 on a well-formed payload). The per-game component means and variances in the predictions payload are deliberately not modelled because they are renamed every season. `getEventPredictions` merges the qualification and playoff maps into one keyed by match key and returns an empty map while there is nothing to predict.
- `TbaApiStatus` - `currentSeason` and `maxSeason` from the `/status` endpoint.

### Exceptions

- `TbaApiKeyMissingException` - no usable key was resolved. Thrown before any network call so you can handle it as a configuration error rather than a transport one.
- `TbaApiException` - carries `statusCode` and the raw response `body`. Raised on non-2xx responses that are not 404 (or on any non-2xx for `getStatus`).

Call `client.close()` when you are done to release the underlying `http.Client`.

## Development

```sh
dart pub get
dart test
```

Tests use a mock `http.Client` (from `package:http/testing`) so they run without network access.

## License

AGPL-3.0
