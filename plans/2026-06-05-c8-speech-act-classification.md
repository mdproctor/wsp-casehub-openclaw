# C8 Speech Act Classification Phase 2+3 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the always-DONE `DefaultSpeechActClassifier` with a three-tier detection pipeline (JSON → bracket prefix → STATUS fallback) that strips classification metadata from Qhorus message content.

**Architecture:** A new public utility class `SpeechActDetection` encapsulates JSON and prefix detection, returning `Optional<SpeechActResult>`. `DefaultSpeechActClassifier` delegates to it and applies the STATUS fallback. `OversightGateService` uses the classified type and stripped content throughout, while preserving the raw output for the oversight-channel COMMAND audit record.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson Databind 2.21.1 (transitive via `quarkus-rest-client-jackson`), JUnit 5, Mockito, AssertJ. No new dependencies required.

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `casehub/src/main/java/io/casehub/openclaw/casehub/DetectionTier.java` | **Create** | Enum: JSON, PREFIX, FALLBACK |
| `casehub/src/main/java/io/casehub/openclaw/casehub/SpeechActResult.java` | **Create** | Record: type + stripped content + tier |
| `casehub/src/main/java/io/casehub/openclaw/casehub/SpeechActDetection.java` | **Create** | Public utility: JSON + prefix detection → Optional\<SpeechActResult\> |
| `casehub/src/test/java/io/casehub/openclaw/casehub/SpeechActDetectionTest.java` | **Create** | All tier detection cases including edge cases |
| `casehub/src/main/java/io/casehub/openclaw/casehub/SpeechActContext.java` | **Modify** | Drop `actionType` field — breaking |
| `casehub/src/main/java/io/casehub/openclaw/casehub/SpeechActClassifier.java` | **Modify** | Return type `MessageType` → `SpeechActResult` — breaking |
| `casehub/src/main/java/io/casehub/openclaw/casehub/DefaultSpeechActClassifier.java` | **Modify** | Delegate to `SpeechActDetection`; STATUS fallback; Tier 3 log |
| `casehub/src/test/java/io/casehub/openclaw/casehub/DefaultSpeechActClassifierTest.java` | **Modify** | Replace Phase 1 tests; add Phase 2+3 cases |
| `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java` | **Modify** | Use `SpeechActResult`; stripped content; add speechAct param to `openGate()` |
| `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java` | **Modify** | Update mock setup; add 9 new tests |
| `skills/casehub-global/SKILL.md` | **Modify** | Add case step response protocol section |

---

## Task 1: Data Types — DetectionTier + SpeechActResult

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/DetectionTier.java`
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/SpeechActResult.java`

These are pure data types with no logic. No tests needed.

- [ ] **Step 1: Create `DetectionTier.java`**

```java
package io.casehub.openclaw.casehub;

/**
 * Indicates which detection tier produced a {@link SpeechActResult}.
 * JSON and PREFIX represent explicit agent signals; FALLBACK means no
 * explicit signal was present. Future NliSpeechActClassifier adds NEURAL.
 */
public enum DetectionTier {
    JSON,
    PREFIX,
    FALLBACK
}
```

- [ ] **Step 2: Create `SpeechActResult.java`**

```java
package io.casehub.openclaw.casehub;

import io.casehub.qhorus.api.message.MessageType;

/**
 * Result of {@link SpeechActClassifier#classify(SpeechActContext)}.
 *
 * @param type    the classified Qhorus {@link MessageType}
 * @param content the stripped message body — bracket prefix and JSON envelope
 *                are removed; never null (empty string when output was null or blank)
 * @param tier    which detection tier produced this result
 */
public record SpeechActResult(MessageType type, String content, DetectionTier tier) {}
```

- [ ] **Step 3: Build to confirm no compile errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl casehub -am -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  casehub/src/main/java/io/casehub/openclaw/casehub/DetectionTier.java \
  casehub/src/main/java/io/casehub/openclaw/casehub/SpeechActResult.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): add DetectionTier enum and SpeechActResult record

Refs #10"
```

---

## Task 2: SpeechActDetection Utility (TDD)

**Files:**
- Create: `casehub/src/test/java/io/casehub/openclaw/casehub/SpeechActDetectionTest.java`
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/SpeechActDetection.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.openclaw.casehub;

import org.junit.jupiter.api.Test;
import io.casehub.qhorus.api.message.MessageType;
import static org.assertj.core.api.Assertions.assertThat;

class SpeechActDetectionTest {

    // ── Tier 1: JSON ──────────────────────────────────────────────────────────

    @Test
    void detect_jsonDone_returnsJsonResult() {
        var result = SpeechActDetection.detect("{\"type\":\"DONE\",\"content\":\"ok\"}");
        assertThat(result).isPresent();
        assertThat(result.get().type()).isEqualTo(MessageType.DONE);
        assertThat(result.get().content()).isEqualTo("ok");
        assertThat(result.get().tier()).isEqualTo(DetectionTier.JSON);
    }

    @Test
    void detect_jsonStatusLowercase_normalises() {
        var result = SpeechActDetection.detect("{\"type\":\"status\",\"content\":\"working\"}");
        assertThat(result).isPresent();
        assertThat(result.get().type()).isEqualTo(MessageType.STATUS);
        assertThat(result.get().content()).isEqualTo("working");
    }

    @Test
    void detect_jsonDecline_returnsDecline() {
        var result = SpeechActDetection.detect("{\"type\":\"DECLINE\",\"content\":\"can't do it\"}");
        assertThat(result).isPresent();
        assertThat(result.get().type()).isEqualTo(MessageType.DECLINE);
        assertThat(result.get().content()).isEqualTo("can't do it");
    }

    @Test
    void detect_jsonFailure_returnsFailure() {
        var result = SpeechActDetection.detect("{\"type\":\"FAILURE\",\"content\":\"error\"}");
        assertThat(result).isPresent();
        assertThat(result.get().type()).isEqualTo(MessageType.FAILURE);
    }

    @Test
    void detect_jsonResponse_returnsResponse() {
        var result = SpeechActDetection.detect("{\"type\":\"RESPONSE\",\"content\":\"answer\"}");
        assertThat(result).isPresent();
        assertThat(result.get().type()).isEqualTo(MessageType.RESPONSE);
    }

    @Test
    void detect_jsonUnknownType_returnsEmpty() {
        var result = SpeechActDetection.detect("{\"type\":\"ESCALATE\",\"content\":\"x\"}");
        assertThat(result).isEmpty();
    }

    @Test
    void detect_jsonNonClassifiableType_returnsEmpty() {
        // COMMAND is a valid MessageType but not classifiable from webhook output
        var result = SpeechActDetection.detect("{\"type\":\"COMMAND\",\"content\":\"x\"}");
        assertThat(result).isEmpty();
    }

    @Test
    void detect_jsonMissingContent_returnsEmpty() {
        var result = SpeechActDetection.detect("{\"type\":\"DONE\"}");
        assertThat(result).isEmpty();
    }

    @Test
    void detect_jsonNullContent_returnsEmpty() {
        var result = SpeechActDetection.detect("{\"type\":\"DONE\",\"content\":null}");
        assertThat(result).isEmpty();
    }

    @Test
    void detect_jsonMalformed_returnsEmpty() {
        var result = SpeechActDetection.detect("{broken");
        assertThat(result).isEmpty();
    }

    @Test
    void detect_jsonFenced_returnsEmpty() {
        var result = SpeechActDetection.detect("```json\n{\"type\":\"DONE\",\"content\":\"x\"}");
        assertThat(result).isEmpty();
    }

    @Test
    void detect_jsonTrailingText_returnsEmpty() {
        // Strict parsing: trailing non-JSON text is a parse failure → fall through
        var result = SpeechActDetection.detect("{\"type\":\"DONE\",\"content\":\"ok\"} extra text here");
        assertThat(result).isEmpty();
    }

    @Test
    void detect_jsonLeadingWhitespace_trimsAndDetects() {
        var result = SpeechActDetection.detect("  {\"type\":\"DONE\",\"content\":\"ok\"}");
        assertThat(result).isPresent();
        assertThat(result.get().content()).isEqualTo("ok");
    }

    // ── Tier 2: Prefix ────────────────────────────────────────────────────────

    @Test
    void detect_prefixDone_returnsPrefixResult() {
        var result = SpeechActDetection.detect("[DONE] task finished");
        assertThat(result).isPresent();
        assertThat(result.get().type()).isEqualTo(MessageType.DONE);
        assertThat(result.get().content()).isEqualTo("task finished");
        assertThat(result.get().tier()).isEqualTo(DetectionTier.PREFIX);
    }

    @Test
    void detect_prefixStatusLowercase_normalises() {
        var result = SpeechActDetection.detect("[status] still running");
        assertThat(result).isPresent();
        assertThat(result.get().type()).isEqualTo(MessageType.STATUS);
        assertThat(result.get().content()).isEqualTo("still running");
    }

    @Test
    void detect_prefixNoSpace_stripsPrefix() {
        var result = SpeechActDetection.detect("[DONE]task finished");
        assertThat(result).isPresent();
        assertThat(result.get().content()).isEqualTo("task finished");
    }

    @Test
    void detect_prefixWithColon_stripsColonAndWhitespace() {
        var result = SpeechActDetection.detect("[STATUS]: progress update");
        assertThat(result).isPresent();
        assertThat(result.get().type()).isEqualTo(MessageType.STATUS);
        assertThat(result.get().content()).isEqualTo("progress update");
    }

    @Test
    void detect_prefixEmptyContent_returnsEmptyString() {
        var result = SpeechActDetection.detect("[DONE]");
        assertThat(result).isPresent();
        assertThat(result.get().type()).isEqualTo(MessageType.DONE);
        assertThat(result.get().content()).isEqualTo("");
    }

    @Test
    void detect_prefixUnknownType_returnsEmpty() {
        var result = SpeechActDetection.detect("[ESCALATE] help");
        assertThat(result).isEmpty();
    }

    @Test
    void detect_prefixLeadingWhitespace_trimsAndDetects() {
        var result = SpeechActDetection.detect("  [STATUS] working");
        assertThat(result).isPresent();
        assertThat(result.get().type()).isEqualTo(MessageType.STATUS);
        assertThat(result.get().content()).isEqualTo("working");
    }

    // ── Tier 3: No signal ─────────────────────────────────────────────────────

    @Test
    void detect_noSignal_returnsEmpty() {
        assertThat(SpeechActDetection.detect("Task is complete.")).isEmpty();
    }

    @Test
    void detect_nullInput_returnsEmpty() {
        assertThat(SpeechActDetection.detect(null)).isEmpty();
    }

    @Test
    void detect_emptyInput_returnsEmpty() {
        assertThat(SpeechActDetection.detect("")).isEmpty();
    }

    @Test
    void detect_blankInput_returnsEmpty() {
        assertThat(SpeechActDetection.detect("   ")).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests — confirm they fail with NoClassDefFound**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dtest=SpeechActDetectionTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: compilation failure — `SpeechActDetection` does not exist yet.

- [ ] **Step 3: Create `SpeechActDetection.java`**

```java
package io.casehub.openclaw.casehub;

import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.qhorus.api.message.MessageType;

/**
 * Detects explicit speech act signals in agent output text.
 *
 * <p>Tries Tier 1 (JSON) then Tier 2 (bracket prefix). Returns
 * {@code Optional.empty()} when no explicit signal is found — the caller
 * supplies the fallback.
 *
 * <p>JSON parsing uses strict mode ({@code FAIL_ON_TRAILING_TOKENS}) so that
 * output containing JSON followed by explanatory text falls through to Tier 2
 * rather than silently matching.
 *
 * <p><b>Internal API — no stability guarantee.</b> Will be promoted to a
 * stable API when {@code casehub-openclaw-inference} is introduced
 * (openclaw#27).
 */
public final class SpeechActDetection {

    private static final ObjectMapper MAPPER = new ObjectMapper()
            .enable(DeserializationFeature.FAIL_ON_TRAILING_TOKENS);

    private static final Pattern PREFIX_PATTERN =
            Pattern.compile("^\\[([A-Za-z]+)\\]:?\\s*", Pattern.DOTALL);

    private static final Set<MessageType> CLASSIFIABLE = Set.of(
            MessageType.DONE, MessageType.STATUS, MessageType.DECLINE,
            MessageType.FAILURE, MessageType.RESPONSE);

    private SpeechActDetection() {}

    /**
     * Attempts to detect a speech act in {@code output}.
     * Trims input before matching. Returns {@code Optional.empty()} if no
     * explicit signal is found.
     */
    public static Optional<SpeechActResult> detect(String output) {
        if (output == null || output.isBlank()) {
            return Optional.empty();
        }
        String trimmed = output.trim();

        // Tier 1 — JSON
        if (trimmed.startsWith("{")) {
            Optional<SpeechActResult> json = tryJson(trimmed);
            if (json.isPresent()) return json;
        }

        // Tier 2 — bracket prefix
        return tryPrefix(trimmed);
    }

    @SuppressWarnings("unchecked")
    private static Optional<SpeechActResult> tryJson(String trimmed) {
        try {
            // Use Map to avoid Jackson record deserialization issues with a standalone ObjectMapper.
            // instanceof String guards handle both absent and null content field values.
            Map<String, Object> map = MAPPER.readValue(trimmed, Map.class);
            Object typeObj = map.get("type");
            Object contentObj = map.get("content");
            if (!(typeObj instanceof String typeStr)) return Optional.empty();
            if (!(contentObj instanceof String contentStr)) return Optional.empty();
            MessageType type = parseType(typeStr);
            if (type == null) return Optional.empty();
            return Optional.of(new SpeechActResult(type, contentStr, DetectionTier.JSON));
        } catch (Exception e) {
            return Optional.empty();
        }
    }

    private static Optional<SpeechActResult> tryPrefix(String trimmed) {
        Matcher m = PREFIX_PATTERN.matcher(trimmed);
        if (!m.find()) return Optional.empty();
        MessageType type = parseType(m.group(1));
        if (type == null) return Optional.empty();
        String content = trimmed.substring(m.end());
        return Optional.of(new SpeechActResult(type, content, DetectionTier.PREFIX));
    }

    private static MessageType parseType(String s) {
        if (s == null) return null;
        try {
            MessageType t = MessageType.valueOf(s.toUpperCase());
            return CLASSIFIABLE.contains(t) ? t : null;
        } catch (IllegalArgumentException e) {
            return null;
        }
    }
}
```

- [ ] **Step 4: Run tests — confirm all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dtest=SpeechActDetectionTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `Tests run: 24, Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  casehub/src/test/java/io/casehub/openclaw/casehub/SpeechActDetectionTest.java \
  casehub/src/main/java/io/casehub/openclaw/casehub/SpeechActDetection.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): add SpeechActDetection — JSON + prefix classification utility

Refs #10"
```

---

## Task 3: SPI Breaking Changes — SpeechActContext + SpeechActClassifier

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/SpeechActContext.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/SpeechActClassifier.java`

These changes break two callers. Fix them to restore compile.

- [ ] **Step 1: Update `SpeechActContext.java` — drop `actionType`**

Replace the entire file:

```java
package io.casehub.openclaw.casehub;

/**
 * Input to {@link SpeechActClassifier} describing an agent's output.
 *
 * @param agentId the OpenClaw agent that produced the output
 * @param output  the raw agent output text
 */
public record SpeechActContext(String agentId, String output) {}
```

- [ ] **Step 2: Update `SpeechActClassifier.java` — return type to `SpeechActResult`**

Replace the entire file:

```java
package io.casehub.openclaw.casehub;

import io.casehub.qhorus.api.message.MessageType;

/**
 * Classifies an OpenClaw agent output into a Qhorus {@link MessageType} and
 * a stripped content string.
 *
 * <p><b>Phase 1 ({@link DefaultSpeechActClassifier}):</b> STATUS fallback for
 * unrecognised output; DONE/STATUS/DECLINE/FAILURE/RESPONSE for explicit signals.
 *
 * <p><b>Phase 2 (openclaw#10):</b> bracket prefix detection —
 * {@code [STATUS] Boiler pressure 1.2 bar} → STATUS.
 *
 * <p><b>Phase 3 (openclaw#10):</b> structured JSON detection —
 * {@code {"type":"STATUS","content":"..."}} → STATUS.
 *
 * <p>Override with {@code @Alternative @Priority(1)}.
 * Any {@code @Alternative} implementation must be updated to return
 * {@link SpeechActResult} — the return type changed from {@link MessageType}
 * in this pass.
 *
 * <p>Future: {@code NliSpeechActClassifier @Alternative @Priority(1)} in
 * {@code casehub-openclaw-inference} (openclaw#27) will call
 * {@link SpeechActDetection#detect(String)} and fall back to ML classification.
 */
public interface SpeechActClassifier {
    SpeechActResult classify(SpeechActContext context);
}
```

- [ ] **Step 3: Fix compile errors in callers**

`OversightGateService.java` has two call sites that must compile:
1. `speechActClassifier.classify(new SpeechActContext(agentId, output, null))` — remove third arg
2. The return value is now `SpeechActResult` — temporarily assign to `var` to suppress type errors

Make the minimal change to restore compilation. Replace the `SpeechActContext` constructor call:

In `OversightGateService.java`, find and update:
```java
// OLD:
MessageType messageType = speechActClassifier.classify(
        new SpeechActContext(agentId, output, null));
// NEW (temporary — full update in Task 5):
SpeechActResult speechAct = speechActClassifier.classify(
        new SpeechActContext(agentId, output));
MessageType messageType = speechAct.type();
```

- [ ] **Step 4: Fix compile error in `OversightGateServiceTest.java`**

Line 89 currently returns `MessageType.DONE` but the interface now returns `SpeechActResult`. Update the mock setup in `@BeforeEach`:

```java
// OLD:
when(speechActClassifier.classify(any())).thenReturn(MessageType.DONE);
// NEW — passthrough mock: returns DONE with whatever content the SpeechActContext carries
when(speechActClassifier.classify(any())).thenAnswer(invocation -> {
    SpeechActContext ctx = invocation.getArgument(0);
    String content = ctx.output() != null ? ctx.output() : "";
    return new SpeechActResult(MessageType.DONE, content, DetectionTier.FALLBACK);
});
```

Also update the test `evaluate_autonomous_doesNotRequireInReplyToForStatusType` which overrides the mock:
```java
// OLD:
when(speechActClassifier.classify(any())).thenReturn(MessageType.STATUS);
// NEW:
when(speechActClassifier.classify(any())).thenReturn(
    new SpeechActResult(MessageType.STATUS, "Progress update.", DetectionTier.PREFIX));
```

- [ ] **Step 5: Build and run existing tests to confirm they compile and pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dtest="OversightGateServiceTest,DefaultSpeechActClassifierTest" \
  -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `BUILD SUCCESS` — existing tests pass (DefaultSpeechActClassifier still compiles since it returns `MessageType.DONE` from a method that now must return `SpeechActResult` — this will be a compile error; fix in Task 4 Step 1).

Note: if `DefaultSpeechActClassifier` fails to compile (it still returns `MessageType`), exclude it from this step: `-Dtest=OversightGateServiceTest` only, and fix `DefaultSpeechActClassifier` first in Task 4.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  casehub/src/main/java/io/casehub/openclaw/casehub/SpeechActContext.java \
  casehub/src/main/java/io/casehub/openclaw/casehub/SpeechActClassifier.java \
  casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java \
  casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "refactor(casehub): SpeechActClassifier returns SpeechActResult; drop SpeechActContext.actionType

Breaking SPI change — only internal callers.
Temporary minimal fix to OversightGateService (full update in next commit).

Refs #10"
```

---

## Task 4: DefaultSpeechActClassifier — TDD

**Files:**
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/DefaultSpeechActClassifierTest.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/DefaultSpeechActClassifier.java`

- [ ] **Step 1: Replace `DefaultSpeechActClassifierTest.java` with full test suite**

```java
package io.casehub.openclaw.casehub;

import org.junit.jupiter.api.Test;
import io.casehub.qhorus.api.message.MessageType;
import static org.assertj.core.api.Assertions.assertThat;

class DefaultSpeechActClassifierTest {

    DefaultSpeechActClassifier classifier = new DefaultSpeechActClassifier();

    @Test
    void classify_jsonDone_returnsDoneWithStrippedContent() {
        var result = classifier.classify(
            new SpeechActContext("agent", "{\"type\":\"DONE\",\"content\":\"Task complete.\"}"));
        assertThat(result.type()).isEqualTo(MessageType.DONE);
        assertThat(result.content()).isEqualTo("Task complete.");
        assertThat(result.tier()).isEqualTo(DetectionTier.JSON);
    }

    @Test
    void classify_prefixDecline_returnsDeclineWithStrippedContent() {
        var result = classifier.classify(
            new SpeechActContext("agent", "[DECLINE] Cannot access external API."));
        assertThat(result.type()).isEqualTo(MessageType.DECLINE);
        assertThat(result.content()).isEqualTo("Cannot access external API.");
        assertThat(result.tier()).isEqualTo(DetectionTier.PREFIX);
    }

    @Test
    void classify_noPrefix_returnsStatusFallback() {
        var result = classifier.classify(
            new SpeechActContext("agent", "I have analysed the data."));
        assertThat(result.type()).isEqualTo(MessageType.STATUS);
        assertThat(result.content()).isEqualTo("I have analysed the data.");
        assertThat(result.tier()).isEqualTo(DetectionTier.FALLBACK);
    }

    @Test
    void classify_nullOutput_returnsStatusFallbackWithEmptyContent() {
        var result = classifier.classify(new SpeechActContext("agent", null));
        assertThat(result.type()).isEqualTo(MessageType.STATUS);
        assertThat(result.content()).isEqualTo("");
        assertThat(result.tier()).isEqualTo(DetectionTier.FALLBACK);
    }

    @Test
    void classify_emptyOutput_returnsStatusFallback() {
        var result = classifier.classify(new SpeechActContext("agent", ""));
        assertThat(result.type()).isEqualTo(MessageType.STATUS);
        assertThat(result.content()).isEqualTo("");
        assertThat(result.tier()).isEqualTo(DetectionTier.FALLBACK);
    }

    @Test
    void classify_jsonFailure_returnsFailure() {
        var result = classifier.classify(
            new SpeechActContext("agent", "{\"type\":\"FAILURE\",\"content\":\"DB timeout.\"}"));
        assertThat(result.type()).isEqualTo(MessageType.FAILURE);
        assertThat(result.content()).isEqualTo("DB timeout.");
    }

    @Test
    void classify_prefixStatus_returnsStatus() {
        var result = classifier.classify(
            new SpeechActContext("agent", "[STATUS] Still processing row 3 of 10."));
        assertThat(result.type()).isEqualTo(MessageType.STATUS);
        assertThat(result.content()).isEqualTo("Still processing row 3 of 10.");
    }
}
```

- [ ] **Step 2: Run tests — confirm they fail** (DefaultSpeechActClassifier still returns MessageType)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dtest=DefaultSpeechActClassifierTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: compile error — `DefaultSpeechActClassifier.classify()` returns `MessageType`, not `SpeechActResult`.

- [ ] **Step 3: Update `DefaultSpeechActClassifier.java`**

```java
package io.casehub.openclaw.casehub;

import jakarta.enterprise.context.ApplicationScoped;
import org.jboss.logging.Logger;
import io.casehub.qhorus.api.message.MessageType;

@ApplicationScoped
public class DefaultSpeechActClassifier implements SpeechActClassifier {

    private static final Logger log = Logger.getLogger(DefaultSpeechActClassifier.class);

    @Override
    public SpeechActResult classify(SpeechActContext ctx) {
        return SpeechActDetection.detect(ctx.output())
                .orElseGet(() -> {
                    log.infof("SpeechActDetection: no explicit signal from agentId=%s"
                            + " — STATUS fallback applied", ctx.agentId());
                    String content = ctx.output() != null ? ctx.output() : "";
                    return new SpeechActResult(MessageType.STATUS, content, DetectionTier.FALLBACK);
                });
    }
}
```

- [ ] **Step 4: Run tests — confirm all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dtest=DefaultSpeechActClassifierTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `Tests run: 7, Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  casehub/src/test/java/io/casehub/openclaw/casehub/DefaultSpeechActClassifierTest.java \
  casehub/src/main/java/io/casehub/openclaw/casehub/DefaultSpeechActClassifier.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): DefaultSpeechActClassifier Phase 2+3 — JSON+prefix detection, STATUS fallback

Refs #10"
```

---

## Task 5: OversightGateService — Full Update (TDD)

**Files:**
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java`

- [ ] **Step 1: Add 9 new tests to `OversightGateServiceTest.java`**

Add these tests inside the class, after the existing `evaluate_autonomous_doesNotInvokeOpenClaw` test. The existing `@BeforeEach` passthrough mock and the `evaluate_autonomous_doesNotRequireInReplyToForStatusType` fix from Task 3 must already be in place.

```java
// ── New tests: Phase 2+3 content stripping and type routing ──────────────

@Test
void evaluate_prefixStatus_dispatchesStatusWithInReplyTo() {
    // [STATUS] prefix → STATUS dispatched with inReplyTo + correlationId
    // (ACKNOWLEDGED is the verified Qhorus consequence — see spec §Classifiable MessageTypes;
    // this test asserts dispatch arguments only)
    when(speechActClassifier.classify(any())).thenReturn(
        new SpeechActResult(MessageType.STATUS, "Still processing.", DetectionTier.PREFIX));

    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();
    long commandMessageId = 55L;

    when(commitmentStore.findOpenByObligor(agentId, workChannelId))
            .thenReturn(List.of(openCommandCommitment(workChannelId, agentId, correlationId)));
    Message commandMsg = new Message();
    commandMsg.id = commandMessageId;
    commandMsg.messageType = MessageType.COMMAND;
    commandMsg.correlationId = correlationId;
    when(messageService.findAllByCorrelationId(correlationId)).thenReturn(List.of(commandMsg));

    service.evaluate(workChannelId, agentId, "[STATUS] Still processing.");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().type()).isEqualTo(MessageType.STATUS);
    assertThat(captor.getValue().inReplyTo()).isEqualTo(commandMessageId);
    assertThat(captor.getValue().correlationId()).isEqualTo(correlationId);
}

@Test
void evaluate_jsonDecline_dispatchesDeclineWithInReplyTo() {
    when(speechActClassifier.classify(any())).thenReturn(
        new SpeechActResult(MessageType.DECLINE, "can't do it", DetectionTier.JSON));

    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();
    long commandMessageId = 77L;

    when(commitmentStore.findOpenByObligor(agentId, workChannelId))
            .thenReturn(List.of(openCommandCommitment(workChannelId, agentId, correlationId)));
    Message commandMsg = new Message();
    commandMsg.id = commandMessageId;
    commandMsg.messageType = MessageType.COMMAND;
    commandMsg.correlationId = correlationId;
    when(messageService.findAllByCorrelationId(correlationId)).thenReturn(List.of(commandMsg));

    service.evaluate(workChannelId, agentId, "{\"type\":\"DECLINE\",\"content\":\"can't do it\"}");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().type()).isEqualTo(MessageType.DECLINE);
    assertThat(captor.getValue().inReplyTo()).isEqualTo(commandMessageId);
}

@Test
void evaluate_jsonDecline_contentIsStrippedNotRawJson() {
    when(speechActClassifier.classify(any())).thenReturn(
        new SpeechActResult(MessageType.DECLINE, "can't do it", DetectionTier.JSON));

    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();
    when(commitmentStore.findOpenByObligor(agentId, workChannelId))
            .thenReturn(List.of(openCommandCommitment(workChannelId, agentId, correlationId)));
    Message commandMsg = new Message();
    commandMsg.id = 1L;
    commandMsg.messageType = MessageType.COMMAND;
    commandMsg.correlationId = correlationId;
    when(messageService.findAllByCorrelationId(correlationId)).thenReturn(List.of(commandMsg));

    service.evaluate(workChannelId, agentId, "{\"type\":\"DECLINE\",\"content\":\"can't do it\"}");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().content()).isEqualTo("can't do it");
    assertThat(captor.getValue().content()).doesNotContain("{\"type\":");
}

@Test
void evaluate_prefixDone_contentIsStrippedNotBracketed() {
    when(speechActClassifier.classify(any())).thenReturn(
        new SpeechActResult(MessageType.DONE, "task finished", DetectionTier.PREFIX));

    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();
    when(commitmentStore.findOpenByObligor(agentId, workChannelId))
            .thenReturn(List.of(openCommandCommitment(workChannelId, agentId, correlationId)));
    Message commandMsg = new Message();
    commandMsg.id = 2L;
    commandMsg.messageType = MessageType.COMMAND;
    commandMsg.correlationId = correlationId;
    when(messageService.findAllByCorrelationId(correlationId)).thenReturn(List.of(commandMsg));

    service.evaluate(workChannelId, agentId, "[DONE] task finished");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().content()).isEqualTo("task finished");
    assertThat(captor.getValue().content()).doesNotStartWith("[DONE]");
}

@Test
void evaluate_noPrefixFallback_withOpenCommitment_dispatchesStatusWithInReplyTo() {
    // No explicit signal → STATUS fallback; commitment is present
    // (ACKNOWLEDGED is the verified Qhorus consequence; this test asserts dispatch arguments only)
    when(speechActClassifier.classify(any())).thenReturn(
        new SpeechActResult(MessageType.STATUS, "I have analysed the data.", DetectionTier.FALLBACK));

    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();
    long commandMessageId = 99L;

    when(commitmentStore.findOpenByObligor(agentId, workChannelId))
            .thenReturn(List.of(openCommandCommitment(workChannelId, agentId, correlationId)));
    Message commandMsg = new Message();
    commandMsg.id = commandMessageId;
    commandMsg.messageType = MessageType.COMMAND;
    commandMsg.correlationId = correlationId;
    when(messageService.findAllByCorrelationId(correlationId)).thenReturn(List.of(commandMsg));

    service.evaluate(workChannelId, agentId, "I have analysed the data.");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().type()).isEqualTo(MessageType.STATUS);
    assertThat(captor.getValue().inReplyTo()).isEqualTo(commandMessageId);
    assertThat(captor.getValue().correlationId()).isEqualTo(correlationId);
}

@Test
void evaluate_noPrefixFallback_watchdogExpiredPath_dispatchesStatusWithoutInReplyTo() {
    // No explicit signal + no open commitment (Watchdog expired)
    when(speechActClassifier.classify(any())).thenReturn(
        new SpeechActResult(MessageType.STATUS, "I have analysed the data.", DetectionTier.FALLBACK));
    when(commitmentStore.findOpenByObligor("finance-agent", workChannelId))
            .thenReturn(Collections.emptyList());

    service.evaluate(workChannelId, "finance-agent", "I have analysed the data.");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().type()).isEqualTo(MessageType.STATUS);
    assertThat(captor.getValue().inReplyTo()).isNull();
}

@Test
void evaluate_oversightGate_commandContentIsRawOutput() {
    // openGate() COMMAND message content = raw output (audit fidelity), NOT stripped content
    when(actionRiskClassifier.classify(any()))
            .thenReturn(new RiskDecision.GateRequired("risk", false));
    String rawOutput = "{\"type\":\"DONE\",\"content\":\"Cancel Netflix.\"}";
    when(speechActClassifier.classify(any())).thenReturn(
        new SpeechActResult(MessageType.DONE, "Cancel Netflix.", DetectionTier.JSON));

    service.evaluate(workChannelId, "finance-agent", rawOutput);

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    // COMMAND content must be the RAW output, not the stripped content
    assertThat(captor.getValue().content()).isEqualTo(rawOutput);
}

@Test
void evaluate_oversightGate_promptUsesStrippedContent() {
    // buildOversightPrompt receives speechAct.content() (stripped), not raw JSON
    when(actionRiskClassifier.classify(any()))
            .thenReturn(new RiskDecision.GateRequired("risk", false));
    String rawOutput = "{\"type\":\"DONE\",\"content\":\"Cancel Netflix subscription.\"}";
    when(speechActClassifier.classify(any())).thenReturn(
        new SpeechActResult(MessageType.DONE, "Cancel Netflix subscription.", DetectionTier.JSON));

    service.evaluate(workChannelId, "finance-agent", rawOutput);

    // The prompt passed to hookClient.invoke() must contain the stripped content
    // and must NOT contain the raw JSON envelope
    ArgumentCaptor<String> promptCaptor = ArgumentCaptor.forClass(String.class);
    verify(hookClient).invoke(anyString(), promptCaptor.capture(), anyString(), anyInt(), anyString());
    String prompt = promptCaptor.getValue();
    assertThat(prompt).contains("Cancel Netflix subscription.");
    assertThat(prompt).doesNotContain("{\"type\":");
}

@Test
void evaluate_watchdogExpiredPath_contentIsStripped() {
    // Watchdog-expired path (no correlationId): STATUS dispatched with speechAct.content()
    when(speechActClassifier.classify(any())).thenReturn(
        new SpeechActResult(MessageType.DONE, "stripped content", DetectionTier.PREFIX));
    when(commitmentStore.findOpenByObligor("finance-agent", workChannelId))
            .thenReturn(Collections.emptyList());

    service.evaluate(workChannelId, "finance-agent", "[DONE] stripped content");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().content()).isEqualTo("stripped content");
    assertThat(captor.getValue().content()).doesNotStartWith("[DONE]");
}
```

- [ ] **Step 2: Run new tests — confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dtest=OversightGateServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: 9 new tests fail (OversightGateService still uses old partial implementation from Task 3).

- [ ] **Step 3: Update `OversightGateService.java` — full implementation**

Replace the `evaluate()` method and `openGate()` signature. The rest of the file is unchanged.

Find the section starting `public void evaluate(` and replace through `private void openGate(`:

```java
    public void evaluate(UUID workChannelId, String agentId, String output) {
        try {
            Channel workChannel = channelService.findById(workChannelId).orElse(null);
            if (workChannel == null) {
                log.warnf("evaluate() called for unknown workChannelId=%s — failing open", workChannelId);
                return;
            }

            UUID caseId = CaseChannelNames.extractCaseId(workChannel.name);
            if (caseId == null) {
                log.warnf("Could not extract caseId from channel name '%s' — failing open", workChannel.name);
                return;
            }

            SpeechActResult speechAct = speechActClassifier.classify(
                    new SpeechActContext(agentId, output));

            RiskDecision decision = actionRiskClassifier.classify(
                    new PlannedAction(agentId, caseId, speechAct.content(), null, Map.of()));

            if (decision instanceof RiskDecision.Autonomous) {
                List<Commitment> open =
                        commitmentStore.findOpenByObligor(agentId, workChannelId);
                boolean hadCommitment = !open.isEmpty();
                String correlationId = open.stream()
                        .filter(c -> c.messageType == MessageType.COMMAND)
                        .map(c -> c.correlationId)
                        .findFirst()
                        .orElse(null);

                Long commandMessageId = resolveCommandMessageId(correlationId);

                MessageDispatch.Builder builder = MessageDispatch.builder()
                        .channelId(workChannelId)
                        .sender(agentId)
                        .content(speechAct.content())
                        .actorType(ActorType.AGENT);

                if (commandMessageId != null && correlationId != null) {
                    builder.type(speechAct.type()).inReplyTo(commandMessageId).correlationId(correlationId);
                } else {
                    builder.type(MessageType.STATUS);
                    if (hadCommitment) {
                        log.errorf("Open COMMAND commitment found for agentId=%s on channel=%s but "
                                + "COMMAND message lookup failed — Commitment will not be fulfilled; "
                                + "Watchdog will escalate. Operator attention required.",
                                agentId, workChannelId);
                    } else {
                        log.warnf("No open COMMAND commitment for agentId=%s on channel=%s — "
                                + "Watchdog may have expired it during agent execution. Dispatching STATUS.",
                                agentId, workChannelId);
                    }
                }
                messageService.dispatch(builder.build());
                return;
            }

            RiskDecision.GateRequired gate = (RiskDecision.GateRequired) decision;
            openGate(caseId, workChannelId, agentId, output, speechAct, gate);

        } catch (Exception e) {
            log.errorf("OversightGateService.evaluate() failed for channel=%s agent=%s: %s",
                    workChannelId, agentId, e.getMessage());
        }
    }

    private void openGate(UUID caseId, UUID workChannelId, String agentId,
                          String rawOutput, SpeechActResult speechAct,
                          RiskDecision.GateRequired gate) {
        Channel oversightChannel = channelService.findByName(
                CaseChannelNames.oversightChannelName(caseId)).orElse(null);
        if (oversightChannel == null) {
            log.errorf("Oversight channel not found for caseId=%s — cannot open gate; failing open. " +
                    "Oversight channel should be created by OpenClawCaseChannelProvider.openChannel().", caseId);
            return;
        }

        UUID gateId = UUID.randomUUID();
        String oversightPrompt = buildOversightPrompt(agentId, speechAct.content(), gate);

        // COMMAND content = raw output for audit fidelity (machine-retrievable).
        // The oversight prompt uses stripped content for human readability.
        messageService.dispatch(MessageDispatch.builder()
                .channelId(oversightChannel.id)
                .sender(GATE_SENDER)
                .type(MessageType.COMMAND)
                .content(rawOutput != null ? rawOutput : "")
                .correlationId(gateId.toString())
                .actorType(ActorType.AGENT)
                .build());

        String oversightDeliveryUrl =
                clientConfig.delivery().baseUrl() + "/openclaw/delivery/oversight/" + gateId;
        String oversightAgentId = casehubConfig.oversight().agentId()
                .filter(s -> !s.isBlank())
                .orElse(agentId);

        hookClient.invoke(oversightAgentId, oversightPrompt,
                clientConfig.agent().defaultModel(),
                clientConfig.agent().defaultTimeoutSeconds(),
                oversightDeliveryUrl);

        log.infof("Oversight gate opened: gateId=%s caseId=%s agent=%s reason=%s",
                gateId, caseId, agentId, gate.reason());
    }
```

- [ ] **Step 4: Run all casehub tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all tests pass including the 9 new ones.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java \
  casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): OversightGateService Phase 2+3 — stripped content, raw audit, SpeechActResult

Refs #10"
```

---

## Task 6: SKILL.md — Case Step Response Protocol

**Files:**
- Modify: `skills/casehub-global/SKILL.md`

- [ ] **Step 1: Add case step response section to `casehub-global/SKILL.md`**

Append the following section after the existing `**Open commitments**` paragraph (before the closing of the file):

```markdown

## Case step responses

**When CaseHub invokes you as a case step (you received a COMMAND and are replying via
the deliver:webhook path), you MUST prefix every response with the speech act type.**
Omitting a prefix is treated as an in-progress update — CaseHub will leave the commitment
open and the Watchdog will escalate, even if you intended to signal completion.

Do not wrap responses in markdown code fences.

**JSON format (preferred — machine-readable, bare JSON only):**

{"type": "DONE", "content": "Your response here."}

**Bracket prefix format (simpler alternative):**

[DONE] Your response here.
[STATUS]: colon after the bracket is also accepted.

Valid types:
- DONE — task complete; commitment resolved as fulfilled
- STATUS — still in progress; Watchdog stays armed (you can send DONE later)
- DECLINE — you cannot complete the task; commitment resolved as declined
- FAILURE — task failed with an error; commitment resolved as failed
- RESPONSE — only if you received a QUERY obligation (not a COMMAND); if in doubt, use DONE
```

- [ ] **Step 2: Verify the SKILL.md renders correctly**

```bash
cat /Users/mdproctor/claude/casehub/openclaw/skills/casehub-global/SKILL.md
```

Confirm the YAML frontmatter is intact and the new section appears at the end.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add skills/casehub-global/SKILL.md
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "docs(skills): add case step response protocol to casehub-global

Agents must prefix responses with [TYPE] or JSON envelope when
replying to a CaseHub COMMAND via deliver:webhook. Omission
defaults to STATUS (Watchdog stays armed).

Refs #10"
```

---

## Task 7: Full Build + Verify

- [ ] **Step 1: Build all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install
```

Expected: `BUILD SUCCESS` across all modules (`core`, `casehub`, `app`).

- [ ] **Step 2: Run casehub module tests explicitly**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all tests pass. Confirm test counts:
- `SpeechActDetectionTest`: 24 tests
- `DefaultSpeechActClassifierTest`: 7 tests
- `OversightGateServiceTest`: original tests + 9 new = ~25 total

- [ ] **Step 3: Confirm `ActionRiskClassifier` javadoc is updated**

Open `casehub/src/main/java/io/casehub/openclaw/casehub/ActionRiskClassifier.java` and verify the javadoc states that `PlannedAction.description` receives stripped content (without speech-act prefix/JSON envelope), not raw output. If not present, add:

```java
/**
 * ...existing javadoc...
 *
 * <p><b>Content semantics:</b> {@code PlannedAction.description()} receives the
 * stripped agent output — any bracket prefix (e.g. {@code [DONE]}) or JSON envelope
 * ({@code {"type":...}}) is removed before this SPI is called. The description
 * reflects the agent's intended action, not the speech-act classification metadata.
 * This applies to real implementations (casehubio/engine#402) — the Phase 1 default
 * is always-AUTONOMOUS and ignores description content.
 */
public interface ActionRiskClassifier {
    RiskDecision classify(PlannedAction action);
}
```

- [ ] **Step 4: Final commit if javadoc was updated**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  casehub/src/main/java/io/casehub/openclaw/casehub/ActionRiskClassifier.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "docs(casehub): ActionRiskClassifier javadoc — PlannedAction.description is stripped content

Refs #10"
```
