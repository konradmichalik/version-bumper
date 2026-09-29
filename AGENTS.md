# AGENTS.md

Guidance for coding agents working in this repository.

## Project overview

`eliashaeussler/version-bumper` is a Composer plugin (`composer-plugin-api`) that bumps project versions across configured files during release preparations. It registers a `bump-version` console command (alias `bv`) through Composer's plugin capability system and exposes a standalone PHP API (`Version\VersionBumper`, `Version\VersionReleaser`, `Version\VersionRangeDetector`) usable outside Composer.

- PHP: `~8.2 || ~8.3 || ~8.4 || ~8.5`
- Composer plugin API: `^2.0`
- Namespace: `EliasHaeussler\VersionBumper\` maps to `src/`

## Structure

- `src/VersionBumperPlugin.php`: plugin bootstrap, only wires up `Command\BumpVersionCommand`
- `src/Command/`: `BumpVersionCommand` drives the whole flow and owns console output
- `src/Config/`: config reading (`ConfigReader`), mapped config objects, `Preset/` for project-type defaults
- `src/Version/`: `VersionBumper`, `VersionReleaser`, `VersionRangeDetector`, `ActionDispatcher`, `Action/`, `RangeDetection/`
- `src/Enum/`, `src/Exception/`, `src/Helper/`, `src/Result/`, `src/Error/`: supporting code
- `tests/unit/`: PHPUnit tests mirroring `src/` 1:1, with fixtures in `tests/unit/Fixtures/`
- `build/checks/`: tool configs (PHPStan, PHP CS Fixer, Rector, dependency analyser)
- `build/tests/`: `phpunit.xml` and coverage output
- `docs/`: user documentation (`schema.md` is the authoritative config reference, also `presets.md`, `actions.md`, `version-range.md`, `api.md`, `cli.md`)

Core flow of `BumpVersionCommand::execute`: read config, merge presets, resolve the version range (explicit or detected from Git history), run pre-actions, bump versions per file via regex patterns with a named `version` group, run post-actions, optionally release (commit and tag). Nothing is written in `--dry-run` mode.

No tool configuration lives in the repository root. Everything is under `build/`.

## Development commands

```bash
composer install
composer ci                 # validate, audit, deps, refactor dry-run, static analysis, style
composer check              # deps + refactor (dry-run) + static + style
composer check:deps         # composer-dependency-analyser
composer check:static       # phpstan analyse -c build/checks/phpstan.php
composer check:style        # composer normalize, editorconfig, php-cs-fixer (all dry-run)
composer check:refactor     # rector dry-run
composer fix                # composer normalize + editorconfig fix + php-cs-fixer fix
composer refactor:php       # rector process, run separately from fix because it can change semantics
```

Typical workflow: `composer fix`, then `composer check`, then `composer test`, then open a pull request.

## Testing

```bash
composer test               # phpunit -c build/tests/phpunit.xml, no coverage
composer test:coverage      # clover, html and junit reports in build/tests/coverage
vendor/bin/phpunit -c build/tests/phpunit.xml tests/unit/Version/VersionBumperTest.php
vendor/bin/phpunit -c build/tests/phpunit.xml --filter <name>
```

- All tests live in `tests/unit/`, mirroring the `src/` namespace (`EliasHaeussler\VersionBumper\Tests\`)
- Fixture configs, fake Git output (`Fixtures/Git/*.txt`) and sample manifests live in `tests/unit/Fixtures/`
- Test doubles `DummyAction`, `DummyPreset` and `DummyCaller` live in `tests/unit/Fixtures/Classes/`
- Git-dependent code is tested by injecting a fake `Caller` that returns fixture text instead of calling Git
- CI (`.github/workflows/ci.yaml`) runs tests on PHP 8.2 to 8.5 with Composer 2.2 and 2.10 and with highest and lowest dependencies, plus a separate coverage job

## Code style and static analysis

- PHP CS Fixer: `build/checks/.php-cs-fixer.php`
- PHPStan: `build/checks/phpstan.php`, baseline in `build/checks/phpstan-baseline.neon`
- Rector: `build/checks/rector.php`
- Dependency analyser: `build/checks/composer-dependency-analyser.php`
- `composer.json` must stay normalized (`composer fix:composer`)
- Follow `.editorconfig` (checked with `ec`)
- Update `docs/schema.md` instead of re-deriving the config schema when config fields change

## Git workflow

- Commit format: `<type>: <description>` with type one of `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `ci`
- Do not add co-author trailers
- Describe the change in the pull request and reference the issue it solves
