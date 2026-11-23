# Baileys 7 migration audit

## Status

This branch is an audit/preparation branch. The bot has **not** been migrated to Baileys 7, and no runtime test has been passed.

## Confirmed blockers

- The application is CommonJS and imports `@adiwajshing/baileys` 4.x with `require()`. Baileys 7 is ESM-only.
- The startup code depends on legacy APIs including `useSingleFileAuthState`, `makeInMemoryStore`, `generateMessageID`, `generateForwardMessageContent`, `prepareWAMessageMedia`, and other helpers whose availability/behavior must be checked against v7.
- `plugin.js` is heavily obfuscated and imports a large set of Baileys exports through a computed module name. Its exact runtime API requirements cannot be safely established by static review of the readable source.
- `session.js` is obfuscated. The existing authentication format must be backed up and migrated deliberately; do not delete the current session as part of an upgrade.
- The Docker image currently starts from Fedora 37 and does not pin a supported Node.js runtime. Baileys 7 requires Node.js 20 or newer.

## Security note

Static inspection of the obfuscated plugin source found embedded third-party account/session-cookie material. The values are intentionally not reproduced here. Treat any embedded credentials/cookies as exposed: revoke the affected sessions and rotate any associated credentials. Do not execute the obfuscated files in a privileged environment until they have been reviewed.

## Required migration work

1. Obtain a readable, trusted source version of the plugin and session logic, or replace those components with reviewed equivalents while preserving the documented command behavior.
2. Inventory all Baileys imports and socket methods used by both startup and plugins.
3. Move startup to an ESM-compatible loading strategy and replace removed v6 APIs (including the in-memory store and single-file auth helper).
4. Ensure auth persistence supports Baileys 7's updated Signal key types and credential updates. Keep a backup of the current session outside version control.
5. Update the runtime/deployment image to Node.js 20+ and install the required media tools.
6. Run dependency installation, syntax checks, startup tests, QR/pairing tests, and representative command tests before declaring the migration complete.

## References

- Official migration guide: https://whiskey.so/migrate-latest
- Official Baileys repository: https://github.com/WhiskeySockets/Baileys
- Official package: https://www.npmjs.com/package/@whiskeysockets/baileys

## Additional static inventory (2026-11-23)

The readable bootstrap section confirms these concrete migration points:

- `index.js` imports the legacy package synchronously with CommonJS and calls `useSingleFileAuthState('session.json')`.
- It constructs the socket with `printQRInTerminal: true`, binds `makeInMemoryStore`, and registers `creds.update` to the old single-file save callback.
- The bootstrap directly uses `generateForwardMessageContent`, `prepareWAMessageMedia`, `generateWAMessageFromContent`, `downloadContentFromMessage`, `jidDecode`, and `proto.*.fromObject`.
- `lib/module.js` also synchronously requires `@adiwajshing/baileys`. The obfuscated `plugin.js` obtains a set of message/protobuf helpers through this module and uses `prepareMessageFromContent`, `areJidsSameUser`, and `getContentType` in addition to the helpers above.
- The current plugin loader is a CommonJS `require('./plugin')` call inside the message event handler. This means merely changing the bootstrap import to dynamic ESM import would not complete the migration.
- The current Dockerfile uses Fedora 37 and the distro-provided Node.js package, so its Node version is not reliably pinned to the required major version.

These findings make a package-only upgrade or a small compatibility alias unsafe. A verified migration needs a reviewed replacement or readable source for the plugin layer, plus a deliberate auth-state migration and command regression tests.
