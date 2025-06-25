# pathguard-fs: Dynamic File System Permissions for Node.js

## Overview

A Node.js module that wraps the native `fs` module to enforce granular, dynamically-changeable file system permissions. Designed for environments where LLM-generated code needs safe file system access with runtime-modifiable restrictions.

## Core Requirements

### Permission Model
- **8 permission types**: `read`, `write`, `delete`, `delete-recursive`, `execute`, `stat`, `chmod`, `traverse`
- **Path patterns**: Glob support (`/app/data/**/*.json`)
- **Dynamic updates**: Permissions changeable during execution without restart
- **Explicit-only**: No inheritance; `/app` access doesn't imply `/app/subdir`

### Security Guarantees
- **Path traversal prevention**: All paths resolved via `fs.realpath()` before permission check
- **File descriptor tracking**: Operations on numeric fds check current permissions
- **Child process isolation**: No fd inheritance to spawned processes
- **Worker thread blocking**: `new Worker()` throws when pathguard active
- **Resource limits**: Maximum 1000 open fds (configurable)

### Implementation Scope
- **~40 wrapped methods**: All path-based and fd-based fs operations
- **Stream interception**: `pipe()` calls validate destination permissions
- **FileHandle support**: Promise-based API with weak reference tracking
- **Symlink resolution**: Always follows links (TOCTOU race accepted and documented)

## Technical Architecture

### Core Components

```javascript
class PathGuard {
  constructor(config) {
    this.rules = new PermissionRules(config.rules);
    this.fdTracker = new FdTracker(config.maxFds || 1000);
    this.pathCache = new LRUCache({ max: 10000, ttl: 3600000 });
    this.handleRefs = new WeakMap();
  }
}
```

### Permission Checking Flow
1. Intercept fs method call
2. Extract paths from arguments
3. Resolve symlinks with timeout (5s default)
4. Check against permission rules
5. Execute original method if allowed
6. Track fd/handle if applicable

### Critical Design Decisions

**fd Persistence**: Once opened, fds remain valid but each operation rechecks current permissions:
```javascript
// Time T1: fd opened with read permission
const fd = fs.openSync('/data/file.txt', 'r');
// Time T2: read permission revoked
pathguard.revoke('/data/file.txt', ['read']);
// Time T3: read attempt fails
fs.readSync(fd, buffer); // throws PermissionError
```

**Stream Wrapping**: Returned streams proxy `pipe()` method:
```javascript
const readStream = fs.createReadStream('/allowed/input');
const writeStream = fs.createWriteStream('/forbidden/output');
readStream.pipe(writeStream); // throws PermissionError
```

**Child Process Isolation**:
```javascript
spawn(cmd, args); // auto-converts to stdio: ['pipe','pipe','pipe']
spawn(cmd, args, {stdio: 'inherit'}); // throws Error
```

## Performance Characteristics

- **Path resolution**: 1-5ms per operation (cached: <0.1ms)
- **Memory overhead**: ~1KB per open fd + 10MB for 10k cached paths
- **Cache eviction**: LRU with 1hr TTL, auto-prune at 100MB heap

## Security Limitations

### Accepted Risks
1. **TOCTOU races**: Gap between permission check and operation
2. **Hard links**: Multiple paths to same inode unchecked
3. **Memory mapped files**: Not intercepted if exposed by other modules

### Controlled Environment Assumptions
- No untrusted native modules
- No direct syscall access
- Worker threads disabled
- Module whitelist enforced externally

## API

```javascript
const pathguard = require('pathguard-fs');

// Initialize with rules
pathguard.init({
  rules: [
    { path: '/app/data/**', permissions: ['read', 'write'] },
    { path: '/tmp/**', permissions: ['read', 'write', 'delete'] }
  ],
  maxFds: 1000,
  realpathTimeout: 5000
});

// Dynamic permission updates
pathguard.grant('/app/logs', ['write']);
pathguard.revoke('/app/data/sensitive', ['read']);

// Error types
try {
  fs.readFileSync('/etc/passwd');
} catch (e) {
  if (e instanceof pathguard.PermissionError) {
    // Permission denied: read on /etc/passwd
  }
}
```

## Implementation Priority

1. **Phase 1**: Basic path validation (read/write/delete)
2. **Phase 2**: fd tracking and cleanup
3. **Phase 3**: Stream interception
4. **Phase 4**: Advanced permissions (chmod, execute)
5. **Phase 5**: Performance optimization (caching, locks)

## Testing Strategy

- Unit tests for each wrapped method
- fd exhaustion tests
- Permission change race conditions
- Symlink escape attempts
- Child process isolation verification

## anything else

You're right. For LLM safety, those are overengineered concerns.

## Actually Critical Gaps

### 1. Rule Precedence (must fix)
```javascript
rules: [
  { path: '/app/**', permissions: ['read'] },
  { path: '/app/data/**', permissions: ['read', 'write'] }
]
```
Most specific wins? Order matters? This will confuse LLMs. Pick one: **most specific path wins**.

### 2. Method→Permission Mapping (must fix)
LLM needs clear mental model:
```javascript
const opMap = {
  'readFile': ['read'],
  'writeFile': ['write'],
  'unlink': ['delete'],
  'rename': ['delete', 'write'], // both paths
  'copyFile': ['read', 'write'],
  'chmod': ['chmod'],
  'mkdir': ['write'], // parent dir
  'rmdir': ['delete']
}
```

### 3. Initialization (missing)
```javascript
require('pathguard-fs'); // Side effect: wraps fs immediately? 
// or
const pathguard = require('pathguard-fs');
pathguard.init(rules); // Explicit activation?
```
LLMs work better with explicit init.

### 4. Glob Syntax (undefined)
Using `**` and `*` but never specified minimatch vs custom. LLMs need one clear syntax.

Everything else (error leakage, async races, traverse semantics) is noise for this use case. The proposal is solid for preventing LLM file system damage.

Missing: "Works with require() or import()" - both should work identically.

## updates:

```javascript
const methodPermissions = {
  // Read operations
  'readFile': ['read'],
  'readFileSync': ['read'],
  'createReadStream': ['read'],
  'readdir': ['read'],
  'readdirSync': ['read'],
  'opendir': ['read'],
  'opendirSync': ['read'],
  
  // Write operations  
  'writeFile': ['write'],
  'writeFileSync': ['write'],
  'createWriteStream': ['write'],
  'appendFile': ['write'],
  'appendFileSync': ['write'],
  'truncate': ['write'],
  'truncateSync': ['write'],
  
  // Delete operations
  'unlink': ['delete'],
  'unlinkSync': ['delete'],
  'rmdir': ['delete'],
  'rmdirSync': ['delete'],
  'rm': ['delete'],  // or delete-recursive based on options
  'rmSync': ['delete'],
  
  // Multi-path operations
  'rename': ['delete', 'write'],  // arg[0]: delete, arg[1]: write
  'renameSync': ['delete', 'write'],
  'copyFile': ['read', 'write'],   // arg[0]: read, arg[1]: write
  'copyFileSync': ['read', 'write'],
  'link': ['read', 'write'],       // arg[0]: read, arg[1]: write parent
  'linkSync': ['read', 'write'],
  'symlink': ['write'],            // only needs write at destination
  'symlinkSync': ['write'],
  
  // Metadata operations
  'stat': ['stat'],
  'statSync': ['stat'],
  'lstat': ['stat'],
  'lstatSync': ['stat'],
  'access': ['stat'],
  'accessSync': ['stat'],
  
  // Permission/ownership operations
  'chmod': ['chmod'],
  'chmodSync': ['chmod'],
  'chown': ['chmod'],
  'chownSync': ['chmod'],
  'utimes': ['chmod'],
  'utimesSync': ['chmod'],
  
  // Directory operations
  'mkdir': ['write'],  // needs write on parent
  'mkdirSync': ['write'],
  'mkdtemp': ['write'],
  'mkdtempSync': ['write'],
  
  // Watch operations
  'watch': ['read'],
  'watchFile': ['read'],
  'unwatchFile': [],  // no permission needed to stop watching
  
  // fd-based operations (check against tracked path)
  'read': ['read'],
  'readSync': ['read'],
  'write': ['write'],
  'writeSync': ['write'],
  'ftruncate': ['write'],
  'ftruncateSync': ['write'],
  'fstat': ['stat'],
  'fstatSync': ['stat'],
  'fchmod': ['chmod'],
  'fchmodSync': ['chmod'],
  'fchown': ['chmod'],
  'fchownSync': ['chmod'],
  'futimes': ['chmod'],
  'futimesSync': ['chmod'],
  'fdatasync': ['write'],
  'fdatasyncSync': ['write'],
  'fsync': ['write'],
  'fsyncSync': ['write'],
  
  // Special handling needed
  'open': ['varies'],  // depends on flags: 'r'→read, 'w'→write
  'openSync': ['varies'],
  'close': [],  // no permission check, just cleanup
  'closeSync': []
}
```

## Additional Document Sections

### Rule Precedence Algorithm
Most specific path wins, measured by:
1. Explicit paths > glob patterns
2. Longer paths > shorter paths  
3. First match wins for equal specificity

### Default Behavior
Paths with no matching rules: **deny all operations**

### Permission Definitions
- `read`: Read file contents, list directory entries
- `write`: Modify files, create new files/directories in parent
- `delete`: Remove single file/directory (must be empty)
- `delete-recursive`: Remove directory and all contents
- `execute`: Run file as program
- `stat`: Read metadata (size, timestamps, permissions)
- `chmod`: Change permissions, ownership, timestamps
- `traverse`: Not used (simplified model for LLM comprehension)

### Edge Cases
- `fs.rm` with `recursive: true` requires `delete-recursive`
- `fs.open` permission depends on flags ('r', 'w', 'r+', etc.)
- Watch operations check permission once at start
- Streams check permission at creation and pipe

## more updates:

**Problems with the method list:**

1. **`symlink` permission wrong** - creates symlink pointing to target that may not exist. Should check write permission on parent of symlink location, not target. Current mapping allows creating symlinks to `/etc/passwd` if you have write anywhere.

2. **`rm` conditional logic undefined** - "delete or delete-recursive based on options" requires implementation to parse options. Should be explicit:
```javascript
'rm': ['delete'],  // without recursive
'rmSync': ['delete'],
// Separate validation in implementation for recursive flag
```

3. **Missing critical methods:**
- `realpath` / `realpathSync` - needs `stat`? Or no check since used internally?
- `readlink` / `readlinkSync` - read symlink target
- `exists` / `existsSync` - deprecated but still used

4. **`open` "varies" is vague** - Should enumerate:
```javascript
// 'r' → read, 'w' → write, 'r+' → read+write, 'a' → write
```

5. **FileHandle methods missing** - promises API has different method names:
- `filehandle.read()`
- `filehandle.write()`
- `filehandle.close()`

Otherwise solid. The permission definitions and precedence algorithm fill critical gaps.

## final update:

Fix symlink mapping to ['write-parent'] or clarify it checks parent directory

