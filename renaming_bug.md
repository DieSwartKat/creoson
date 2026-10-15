## Renaming Bug Root Cause and Fix

Summary
-------
When running the installed `creoson` the `file.rename` JSON request can fail with the Creo message:

> A Pro/TOOLKIT error has occurred: General Error.  Check your model/drawing for accuracy.

This happens while the server attempts to rename a model when `onlyInSession == true`.

Root cause (what the code does)
--------------------------------
- The JSON handler default for the `onlysession` flag is `false`, so the failing behavior only happens if the client sets `onlysession=true` (see `JLJsonFileHandler.actionRename`).
- In `JLFile.rename(...)` the code attempts to change Creo's working directory to Java's temp directory when `onlyInSession` is true:

- File: [creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JLFile.java](creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JLFile.java#L573)

```java
if (onlyInSession) {
    // we're doing this because creo renames it on disk anyway, if you're in the model's working dir already
    JlinkUtils.changeDirectory(session, System.getProperty("java.io.tmpdir"));
    changedir=true;
}
```

- `JlinkUtils.changeDirectory(...)` calls `session.changeDirectory(dir)` and only catches `XToolkitInvalidName`; any other `jxthrowable` propagates back to `JLFile.rename` and is caught by the outer `catch (jxthrowable e)` which calls `JlinkUtils.createException(e)`.

- When the exception is a `XToolkitError` the helper `toolkitError()` builds the user-facing message. If the underlying error code returned by Creo is generic (`-1`), `toolkitError()` returns the message you saw: "A Pro/TOOLKIT error has occurred: General Error.  Check your model/drawing for accuracy."

Key locations in the code
-------------------------
- `JLFile.rename(...)` — where the directory is changed and `m.rename(...)` is invoked.
  - File: [creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JLFile.java](creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JLFile.java#L532-L620)
- `JlinkUtils.changeDirectory(...)` — calls `session.changeDirectory(dir)` and only converts `XToolkitInvalidName` into `JLIException`; other `jxthrowable` exceptions escape.
  - File: [creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JlinkUtils.java](creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JlinkUtils.java#L285-L295)
- `JlinkUtils.toolkitError(...)` — builds the generic Pro/TOOLKIT message when Creo returns a toolkit error code (e.g. `-1`).
  - File: [creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JlinkUtils.java](creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JlinkUtils.java#L407-L416)
- `CallModel.rename(...)` — wrapper around the JLink `Model.Rename(...)` call that ultimately performs the rename on the model object.
  - File: [creoson-core/src/com/simplifiedlogic/nitro/jlink/calls/model/CallModel.java](creoson-core/src/com/simplifiedlogic/nitro/jlink/calls/model/CallModel.java#L170-L180)

Why it happens only in the installed environment
------------------------------------------------
- In the debug environment the Creo session and JVM environment may accept and be able to change into the temp directory returned by `System.getProperty("java.io.tmpdir")`.
- In the installed environment the temp path may be incompatible for `session.changeDirectory()` (path format, permissions, drive mapping, network path, or Creo restrictions). That results in a `jxthrowable` from the session changeDirectory call, which is not specifically handled and bubbles up to produce the generic Pro/TOOLKIT message.

Concrete repro/observation
--------------------------
- When `onlyInSession=true` the code issues `JlinkUtils.changeDirectory(session, System.getProperty("java.io.tmpdir"))` (see `JLFile.rename`).
- If that `changeDirectory` call throws a `jxthrowable` other than `XToolkitInvalidName`, the outer catch in `JLFile.rename` converts it to a user message using `JlinkUtils.createException`, which for toolkit errors emits the "General Error" message.

Recommended fixes
------------------
1. Safer directory change with fallback (recommended quick fix)

   Replace the direct call with a guarded attempt and fallback to the current Creo directory if change fails. Example (pseudocode):

   ```java
   String tmp = System.getProperty("java.io.tmpdir");
   try {
       JlinkUtils.changeDirectory(session, tmp);
       changedir = true;
   }
   catch (jxthrowable e) {
       // Log the error detail for diagnosis and do not change directory
       DebugLogging.sendDebugMessage("file.rename: changeDirectory to tmp failed: " + e.getMessage(), NitroConstants.DEBUG_KEY);
       // leave changedir false so we won't try to restore to curdir
   }
   ```

   This prevents the `jxthrowable` from reaching the outer handler and producing the generic Pro/TOOLKIT message, and keeps behavior safe in environments where `java.io.tmpdir` isn't acceptable.

2. Improve `JlinkUtils.changeDirectory(...)` error handling (broader fix)

   Modify `JlinkUtils.changeDirectory` to catch and convert other `jxthrowable` exceptions into clearer `JLIException` messages (via `createException`), or at minimum to include the PTC error code/message so the user sees the root cause instead of the generic message.

   Example change in `JlinkUtils.changeDirectory`:

   ```java
   public static void changeDirectory(CallSession session, String dir) throws JLIException {
       if (session==null) return;
       if (dir==null || dir.length()==0) return;
       try {
           session.changeDirectory(dir);
       }
       catch (XToolkitInvalidName jxe) {
           throw new JLIException("Unable to change to directory " + dir);
       }
       catch (jxthrowable jxe) {
           // convert to JLIException with PTC-specific message
           throw createException(jxe, "Unable to change to directory " + dir);
       }
   }
   ```

3. Prefer using the model's path or `curdir` for the in-session rename trick

   Instead of `java.io.tmpdir`, use a directory Creo already uses/knows about (for example `curdir` or the model path) when the intention is to force an in-memory rename without renaming disk files. That is safer because the session already recognizes those directories.

4. Additional diagnostics (helpful for future debugging)

   Add a debug log before and after `changeDirectory` with the exact directory string being used, and include the full `jxthrowable` message/stack when caught. That will make it easy to see why the session refused to change directories in an installed runtime.

Why these fixes are safe
------------------------
- They avoid forcing Creo into a directory it cannot access, preventing toolkit errors.
- They preserve the original intent (avoid disk rename when requested) while failing gracefully when the temp directory path is not usable.
- Logging and better exception wrapping produce actionable error messages instead of a generic toolkit message.

References (exact code locations)
---------------------------------
- `JLFile.rename(...)` — change-dir + rename call: [creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JLFile.java](creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JLFile.java#L532-L620)
- `JlinkUtils.changeDirectory(...)` — wrapper around `session.changeDirectory`: [creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JlinkUtils.java](creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JlinkUtils.java#L285-L295)
- `JlinkUtils.toolkitError(...)` — builds the generic Pro/TOOLKIT message: [creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JlinkUtils.java](creoson-core/src/com/simplifiedlogic/nitro/jlink/impl/JlinkUtils.java#L407-L416)
- `CallModel.rename(...)` — calls the underlying JLink `Rename`: [creoson-core/src/com/simplifiedlogic/nitro/jlink/calls/model/CallModel.java](creoson-core/src/com/simplifiedlogic/nitro/jlink/calls/model/CallModel.java#L170-L180)

Next steps
----------
- I can apply the guarded `changeDirectory` patch and add logging now; tell me if you want the quick fix (guard in `JLFile.rename`) or the broader `JlinkUtils.changeDirectory` error-handling change.
