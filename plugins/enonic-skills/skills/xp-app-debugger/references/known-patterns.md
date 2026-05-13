# Known Debug Patterns

Concise heuristics from past debug sessions. Max ~5 lines per entry. Prune when file exceeds ~30 entries.

---

## Serializer NPE on modify

**Symptom**: `ScriptValueTranslator.handleValue` throws NPE after `node.modify()`
**Cause**: Any null property in the returned object graph triggers it, not just the edited field. The serializer iterates the entire object.
**Fix**: Sanitize the entire returned object — strip nulls before returning from the editor function. Don't assume you know which property
is null.
**Applies to**: XP 8+ (stricter null handling than XP 7)

---

## API not found — wrong descriptor format (XP 8)

**Symptom**: Browser gets `API [com.example.app:my-api] not found` (404). JAR contains `apis/<name>/<name>.xml` and `apis/<name>/<name>.js`
correctly.
**Cause**: XP 8 uses `YmlApiDescriptorParser` — it only looks for `.yml` descriptors. XML descriptors are silently ignored, so the API is
never registered.
**Fix**: Rename `<name>.xml` → `<name>.yml` and convert to YAML format: `allow:\n  - "role:system.admin"`.
**Applies to**: XP 8+ (XP 7 uses XML). Same applies to admin tool and webapp descriptors.

---

## Silent app — no log entries at all

**Symptom**: User reports "app doesn't load" or "I can't see it in logs". Grepping `server.log` for the app name returns zero results.
**Cause**: The app JAR was never deployed to the directory XP reads from. The deploy target path doesn't match `$XP_HOME/deploy/`.
**Fix**: Verify where the build tool places the JAR vs. where XP actually reads from. Check project deploy configuration and `$XP_HOME`.
**Key insight**: Actual app errors (bad code, version mismatch) always produce log entries. Complete absence means XP never saw the JAR.
