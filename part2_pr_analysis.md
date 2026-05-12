Repository: beetbox/beets
PRs Selected: #3877 (Web readonly) and #3279 (parentwork plugin)
PR 1 — #3877: Web readonly
Link: https://github.com/beetbox/beets/pull/3877


PR Summary:
The beets web plugin runs a Flask-based HTTP server that lets outside applications talk to the music library. Before this change, any client reaching that server could freely send DELETE or PATCH requests and actually remove or alter tracks stored in the library. There was no switch to stop this. A user reported the concern in issue #3870 — they wanted to share read access over a home network without risking unwanted changes. This PR adds a readonly configuration option to the web plugin. When enabled, which is now the default, the server only accepts GET requests and blocks all write operations with an HTTP 405 response. Users who genuinely need write access must explicitly add readonly: no to their config file. Making safe behaviour the default protects users who may not even realise write access was previously open.




Technical Changes:
beetsplug/web/__init__.py — setup() method updated to read readonly from beets config and store it inside Flask's app.config as READONLY; route handlers for item_delete, item_change, album_delete, and album_change each get a guard at the very top that returns HTTP 405 before touching the database when READONLY is True
docs/plugins/web.rst — New section added explaining the readonly option, its default value of true, and the config line needed to re-enable write access
test/test_web.py — New test cases added for DELETE and PATCH under readonly mode expecting 405, the same operations with readonly disabled expecting success, and GET requests under both modes always expecting 200





Implementation Approach :
During plugin startup, the code reads self.config['readonly'].get(bool) from the user's config.yaml under the web: block, falling back to True when the key is missing entirely. That value gets pushed into Flask's app.config['READONLY'] so every route handler can access it through the Flask application context without re-reading beets config on every request.
Each write-capable route handler checks this flag as its very first action. If READONLY is True, the handler immediately calls flask.abort(405) and returns, skipping all database work. This ordering is intentional — checking before any database query means a DELETE request targeting a non-existent item ID still returns 405 rather than 404, so the readonly mode does not accidentally reveal whether specific item IDs exist in the library.
Keeping the default as True was a deliberate choice from the discussion in issue #3870. The philosophy is that the safest behaviour should require no action from the user, and anyone who needs write access must make a conscious opt-in decision inside their configuration file.




Potential Impact: 
Users who rely on the web plugin's DELETE or PATCH endpoints through scripts or third-party clients will find those requests silently rejected after upgrading unless they add readonly: no to their config. Users who only read from the web plugin are completely unaffected. No other beets plugins or core library components are touched — the change is limited entirely to the web plugin's Flask layer and its configuration handling.





PR 2 — #3279: parentwork plugin
Link: https://github.com/beetbox/beets/pull/3279
PR Summary 
MusicBrainz organises musical works in a parent-child hierarchy. A symphony movement is its own work, and that movement sits underneath the full symphony, which is a separate work one level above. Beets already recorded the MusicBrainz work ID of the directly linked work for each track, but it never climbed further up that hierarchy. For classical music listeners this is a noticeable gap — knowing a track is labelled "Contrapunctus I" is far less useful without also knowing it belongs to "The Art of Fugue, BWV 1080." This PR adds a brand-new parentwork plugin that solves this by repeatedly following the parent-work relationship in MusicBrainz until it reaches the topmost ancestor, then saves that information as new metadata fields on every matched track in the library.



Technical Changes:
beetsplug/parentwork.py (new file) — Full plugin implementation containing ParentWorkPlugin class, a find_work() method driving the traversal, a mb_asdict() helper that calls the MusicBrainz API and walks parent relations, and a commands() method exposing the beet parentwork CLI command
docs/plugins/parentwork.rst (new file) — Plugin documentation describing the four new metadata fields, the auto and force config options, and usage examples
docs/plugins/index.rst — parentwork added to the plugin index so users can discover it in documentation
test/test_parentwork.py (new file) — Unit tests using mocked musicbrainzngs responses to verify traversal logic, correct field population, and edge case handling




Implementation Approach:
The plugin works in two modes depending on configuration. With auto: yes set, it hooks into the beets import pipeline and runs automatically after MusicBrainz data arrives for each track. Without it, users trigger it manually by running beet parentwork followed by any standard beets query.
The actual climbing logic lives in mb_asdict(). It takes a MusicBrainz work ID, calls musicbrainzngs.get_work_by_id() requesting work relations, then checks the result for a "parts of" link pointing to a parent work. If found, the function calls itself again with the parent's ID and keeps going until it lands on a work that has no further parent. The title and ID of that topmost work are returned.
The composition date is captured from the first work in the chain that carries one. Four new fields get written to each track: parentwork, parent_mb_workid, parentwork_date, and parentwork_workid_current. The last field is an internal bookkeeping tag recording which work ID was active at fetch time, allowing the plugin to detect when a track's MusicBrainz link has changed and a refresh is needed.



Potential Impact: 
Since this is a completely new plugin with no edits to existing files, users who do not enable it see absolutely no change in behaviour. Classical music users who do enable it gain four new metadata fields usable in path templates and queries. The only practical cost is import speed — with auto: yes, each track triggers an additional MusicBrainz API call, and the one-request-per-second rate limit means large classical libraries take noticeably longer to import.
