# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Planet91 — a browser-based strategy game (colonies, buildings/units, troops, missions, a coordinate map) written in plain PHP + MySQL (mysqli) + jQuery. Live demo: www.planet91.com. There is no build system, package manager, linter, or test suite.

## Running

- Import `db/p91.sql` into MySQL (database `P91`). Credentials are hardcoded in `inc/definitions.php` (`root`, empty password, localhost:3306).
- Serve `public/` as the web root (e.g. `cd public && php -S localhost:8000`). **Paths are relative to `public/`** (`../inc/...`, `../classes/...`, `../view/...`), so scripts must be run with `public/` as the working directory.
- Game-tick processing runs in the background and must be started separately (see below). Note `Server_Daemons.php` / `Server_cron.php` use `require_once '../inc/autoloader.php'`, so run them from a sibling directory of `inc/` (e.g. from `public/` or `Server/`), and `Server.php` hardcodes `/usr/local/bin/php` for self-respawn.

## Architecture

**Request flow.** `public/index.php` is the single entry page: it starts the session, loads `inc/autoloader.php` (PSR-0-style: `Namespace\Class` → `classes/Namespace/Class.php`), checks login via `UserProcess\User`, then includes a chain of `inc/get*.php` scripts that load per-request state into globals/constants (e.g. `inc/getmaps.php` defines `UMLAT`, `UMLONG`, `CID` (current colony id), `UMRANGE`, `MCOLONY` from the session user's active colony, selectable via `?co=`). Then `view/frame/{head,body,footer}.php` render the page; `Controller\DefaultController` includes `view/<p>.php` based on the `?p=` query param (default `view/main.php`).

**AJAX endpoints** live in `public/ajax*.php` (`ajaxpost.php`, `ajaxdialog.php`, `ajaxtroop.php`), driven by `public/js/main.js`. They dispatch on `$_GET['p']` / `$_POST['do']` and call into `classes/`. `public/login.php` + `js/loginpanel.js` handle login/registration (`UserProcess\Userdb`).

**Game engine** (`classes/GameEngine/`): domain classes all extend `Database\Dbconnect`, which opens its own mysqli connection per instance (`$this->conn`) — so each `new Foo()` is a new DB connection. Key classes: `Units` (buildings and construction timers), `Barrack`/`Troops`/`Troopprocess*` (troop training and movement), `Missions` (largest file; mission/combat resolution), `Resources` (resource accrual), `Colony`, `Map`, `Souls`.

**Time-based simulation.** Game progress (building completion, troop production, resource income, missions) is advanced by polling, not events:
- `Server/Server_Daemons.php` — long-running loop (~1s sleep) calling `Barrack::usersTroopproduce()`, `Units::buildingUnits()`, `Resources::UsersResourceUpdater()`, and `new Missions()` (Missions processes work in its constructor). It respawns itself every ~250s and exits when memory exceeds 50 MB.
- `Server/Server_cron.php` — the same single tick, for running from cron.
- `Server\Server::CheckServer()` gates both the daemon and `ajaxpost.php` ("Server is offline"); its DB-backed check against the `server` table is currently commented out and always returns `true`.
- `inc/server.php` and `inc/troopupdate.php` also run updates during page requests.

**Database.** Single schema in `db/p91.sql` (tables: `map` (huge seed of coordinate tiles), `units`/`userunits`, `troops`/`usertroops`/`usertroopsproduce`, `usermissions`/`usermissionarmy`, `usercolony`, `users`/`userstats`, `resources`, `message`, `server`, `dungeon`, `userblueprints`). Static definitions (`units`, `troops`) vs. per-user instances (`user*`) are separate tables. Queries use prepared statements with `bind_param`.
