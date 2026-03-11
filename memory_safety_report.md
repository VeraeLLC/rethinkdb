# RethinkDB Memory Safety Analysis Report

**Generated:** 2026-03-11
**Scope:** src/ directory
**Analyzer:** Memory Safety Code Analyzer

---

## Executive Summary

This report identifies memory safety issues in the RethinkDB codebase, including potential memory leaks, use-after-free vulnerabilities, buffer overflows, and unsafe memory management patterns. The codebase heavily relies on manual memory management, which introduces risks that should be addressed.

**Issue Count by Severity:**
- Critical: 4
- High: 8
- Medium: 12
- Low: 6

---

## Critical Severity Issues

### 1. strcpy Buffer Overflow in cJSON.cc (Line 512, 632, 634)
**File:** `src/cjson/cJSON.cc`
**Lines:** 512, 632, 634
**Issue:** Unsafe strcpy usage

```cpp
// Line 512
strcpy(ptr,entries[i]);ptr+=strlen(entries[i]);

// Lines 632-634
strcpy(ptr,names[i]);ptr+=strlen(names[i]);
strcpy(ptr,entries[i]);ptr+=strlen(names[i]);
```

**Description:** The code uses `strcpy` without bounds checking. The destination buffer size is calculated based on string lengths, but there's no verification that the calculated size matches the actual allocated buffer size. Malformed JSON with specially crafted strings could potentially cause buffer overflows.

**Recommended Fix:** Replace `strcpy` with `snprintf` or use `memcpy` with explicit size checks:
```cpp
size_t len = strlen(entries[i]);
memcpy(ptr, entries[i], len);
ptr[len] = '\0';
ptr += len;
```

---

### 2. sprintf Format String Vulnerability (Line 366)
**File:** `src/cjson/cJSON.cc`
**Line:** 366
**Issue:** Unbounded sprintf usage

```cpp
default: sprintf(ptr2,"u%04x",token);ptr2+=5;        break;
```

**Description:** The code uses `sprintf` without bounds checking. While `ptr2` is allocated with sufficient space, there's no runtime verification that the buffer has enough space remaining.

**Recommended Fix:** Use `snprintf` with explicit buffer size:
```cpp
snprintf(ptr2, remaining_space, "u%04x", token);
```

---

### 3. Potential Memory Leak in Semaphore on Exception (Lines 44-54)
**File:** `src/concurrency/semaphore.cc`
**Lines:** 44-54
**Issue:** lock_request_t may leak on exception

```cpp
void static_semaphore_t::co_lock_interruptible(signal_t *interruptor, int64_t count) {
    // ...
    try {
        wait_interruptible(&cb, interruptor);
    } catch (const interrupted_exc_t &ex) {
        // Remove our lock request from the queue
        for (lock_request_t *request = waiters.head(); ... ) {
            if (request->cb == &cb) {
                waiters.remove(request);
                delete request;
                break;
            }
        }
        throw;
    }
}
```

**Description:** If `wait_interruptible` throws an exception OTHER than `interrupted_exc_t`, the `lock_request_t` allocated on line 18 will leak. The catch block only handles `interrupted_exc_t`.

**Recommended Fix:** Add a catch-all handler or use RAII:
```cpp
} catch (...) {
    // Cleanup code
    throw;
}
```

---

### 4. Potential Use-After-Free in parallel_traversal.cc
**File:** `src/btree/parallel_traversal.cc`
**Lines:** 271-291, 451-456, 578-581
**Issue:** `delete this` pattern with subsequent callback usage

```cpp
void you_may_acquire() {
    buf_lock_t block(parent, block_id, state->helper->btree_node_mode());
    acq_start_cb->on_in_line();
    node_ready_callback_t *local_cb = node_ready_cb;
    delete this;
    local_cb->on_node_ready(std::move(block));  // Uses block after complex operation
}
```

**Description:** The code uses `delete this` and then calls a callback. While the immediate use of `local_cb` is safe, if `on_node_ready` triggers any code that accesses the deleted object, it will cause use-after-free. This pattern appears in multiple places.

**Recommended Fix:** Use `std::shared_ptr` or ensure callbacks don't access the object after deletion. Consider using intrusive_ptr patterns instead of raw `delete this`.

---

## High Severity Issues

### 5. Flexible Array Member Pattern (Line 10)
**File:** `src/containers/shared_buffer.cc`
**Lines:** 10-16
**Issue:** UB with flexible array member

```cpp
counted_t<shared_buf_t> shared_buf_t::create(size_t size) {
    size_t memory_size = sizeof(shared_buf_t) + size - 1;  // -1 for data_[1]
    void *raw_result = ::rmalloc(memory_size);
    shared_buf_t *result = static_cast<shared_buf_t *>(raw_result);
    result->refcount_ = 0;
    result->size_ = size;
    return counted_t<shared_buf_t>(result);
}
```

**Description:** The code allocates based on `sizeof(shared_buf_t) + size - 1` assuming `data_[1]` in the struct. This is technically undefined behavior in C++ (though common in C). The struct has `char data_[1]` as a flexible array member hack.

**Recommended Fix:** Use proper flexible array member `char data_[]` (C99) or allocate separately and store a pointer.

---

### 6. Uninitialized Memory Access in cJSON (Line 655)
**File:** `src/cjson/cJSON.cc`
**Line:** 655
**Issue:** memcpy of entire struct may copy uninitialized padding

```cpp
static cJSON *create_reference(cJSON *item) {
    cJSON *ref=cJSON_New_Item();
    if (!ref) return 0;
    memcpy(ref,item,sizeof(cJSON));  // Copies padding bytes
    // ...
}
```

**Description:** `memcpy` copies the entire struct including padding bytes which may be uninitialized. This can cause issues with memory sanitizers and is technically undefined behavior.

**Recommended Fix:** Perform member-wise copy instead of `memcpy`:
```cpp
ref->type = item->type;
ref->valuestring = item->valuestring;
// ... copy each field explicitly
```

---

### 7. Potential Null Pointer Dereference (Line 343)
**File:** `src/cjson/cJSON.cc`
**Line:** 343
**Issue:** unchecked return from cJSON_strdup

```cpp
static char *print_string_ptr(const char *str)
{
    const char *ptr;char *ptr2,*out;int len=0;unsigned char token;
    if (!str) return cJSON_strdup("");
    ptr=str;while ((token=*ptr) && ++len) {if (strchr("\"\\\b\f\n\r\t",token)) len++; else if (token<32) len+=5;ptr++;}
    out=(char*)cJSON_malloc(len+3);
    if (!out) return 0;
    // ... uses out without checking if cJSON_malloc returned null
```

**Description:** While `cJSON_malloc` (which is `rmalloc`) crashes on OOM, the function signature suggests it can return NULL, but callers may not check.

**Recommended Fix:** Either change return type to void and crash, or ensure all callers check for NULL.

---

### 8. strcpy in unittest_utils.cc (Line 119)
**File:** `src/unittest/unittest_utils.cc`
**Line:** 119
**Issue:** Potential buffer overflow with strcpy

```cpp
char tmpl[] = "rdb_unittest.";
char path[MAX_PATH + 1 + sizeof(tmpl) + 6 + 1];
DWORD res = GetTempPath(sizeof(path), path);
guarantee_winerr(res != 0 && res < MAX_PATH + 1, "GetTempPath failed");
strcpy(path + res, tmpl); // NOLINT
```

**Description:** The `strcpy` copies `tmpl` into `path + res`. While there's a comment that this is intentional (NOLINT), there's no explicit bounds check that `res + strlen(tmpl) < sizeof(path)`.

**Recommended Fix:** Use `strncpy` or `snprintf`:
```cpp
snprintf(path + res, sizeof(path) - res, "%s", tmpl);
```

---

### 9. Manual Memory Management in pathspec.cc
**File:** `src/rdb_protocol/pathspec.cc`
**Lines:** 8-50
**Issue:** Manual new/delete in union-like pattern

```cpp
pathspec_t::pathspec_t(const pathspec_t &other) {
    init_from(other);
}

pathspec_t& pathspec_t::operator=(const pathspec_t &other) {
    type_t type_old = type;
    union { datum_string_t *str_old; std::vector<pathspec_t> *vec_old; ... };
    // ... stores old pointers
    init_from(other);  // May throw, leaking old pointers if exception occurs
    // ... deletes old pointers
}
```

**Description:** If `init_from` throws an exception, the old pointers are never deleted, causing memory leaks.

**Recommended Fix:** Use `std::unique_ptr` or implement strong exception guarantee properly.

---

### 10. Buffer Overflow in unittest decode_le32 (Line 227-233)
**File:** `src/unittest/unittest_utils.cc`
**Lines:** 227-233
**Issue:** Potential out-of-bounds read

```cpp
uint32_t decode_le32(const std::string& buf) {
    return static_cast<uint32_t>(
        zero_extend(buf[0]) |
        zero_extend(buf[1]) << 8 |
        zero_extend(buf[2]) << 16 |
        zero_extend(buf[3]) << 32);
}
```

**Description:** This function accesses `buf[0]` through `buf[3]` without checking if the string has at least 4 bytes. If called with a shorter string, this causes undefined behavior.

**Recommended Fix:** Add bounds check:
```cpp
if (buf.size() < 4) { throw or handle error }
```

---

### 11. Potential Double-Free in wait_any.cc
**File:** `src/concurrency/wait_any.cc`
**Lines:** 19-30
**Issue:** Complex ownership semantics

```cpp
void wait_any_t::add(const signal_t *s) {
    wait_any_subscription_t *sub;
    if (subs.size() < default_preallocated_subs) {
        sub = sub_storage[subs.size()].create(this, false);
    } else {
        sub = new wait_any_subscription_t(this, true);
    }
    sub->reset(const_cast<signal_t *>(s));
    subs.push_back(sub);
}
```

**Description:** The ownership of `sub` is unclear. Some subscriptions are heap-allocated (with `on_heap() == true`) while others use preallocated storage. If `reset()` throws, there could be resource leaks.

**Recommended Fix:** Use RAII wrapper for subscriptions to ensure cleanup on exception.

---

### 12. Missing Null Check after malloc in backtrace.cc
**File:** `src/backtrace.cc`
**Line:** 107
**Issue:** Potential null pointer dereference

```cpp
std::string demangle_cpp_name(const char *mangled_name) {
    int res;
    char *name_as_c_str = abi::__cxa_demangle(mangled_name, nullptr, nullptr, &res);
    if (res == 0) {
        std::string name_as_std_string(name_as_c_str);
        free(name_as_c_str);
        return name_as_std_string;
    } else {
        throw demangle_failed_exc_t();
    }
}
```

**Description:** If `abi::__cxa_demangle` returns NULL (which can happen), the code dereferences it when constructing the string. While `res == 0` usually means success, the code should still check for NULL.

**Recommended Fix:**
```cpp
if (res == 0 && name_as_c_str != nullptr) {
    std::string result(name_as_c_str);
    free(name_as_c_str);
    return result;
}
```

---

## Medium Severity Issues

### 13. Stack Buffer in btree/leaf_node.cc
**File:** `src/btree/leaf_node.cc`
**Lines:** 757-770
**Issue:** Fixed-size stack array

```cpp
int adjustable_tow_offsets[MANDATORY_TIMESTAMPS];
int num_adjustable_tow_offsets = 0;
// ...
adjustable_tow_offsets[num_adjustable_tow_offsets] = i;
++num_adjustable_tow_offsets;
```

**Description:** Fixed-size stack array `adjustable_tow_offsets[MANDATORY_TIMESTAMPS]` is used. While there's an assertion checking the index, if `MANDATORY_TIMESTAMPS` is ever changed or the logic has a bug, this could overflow.

**Recommended Fix:** Use `std::vector` or `std::array` with bounds checking.

---

### 14. Manual Memory Management in btree/internal_node.cc
**File:** `src/btree/internal_node.cc`
**Lines:** 142-148
**Issue:** Complex memmove operations with pointer arithmetic

```cpp
memmove(rnode->pair_offsets + node->npairs, rnode->pair_offsets, rnode->npairs * sizeof(*rnode->pair_offsets));
```

**Description:** Complex pointer arithmetic with memmove. If `node->npairs` or `rnode->npairs` are corrupted or calculated incorrectly, this could lead to buffer overflows.

**Recommended Fix:** Add bounds assertions before the memmove:
```cpp
rassert(node->npairs + rnode->npairs <= MAX_PAIRS);
```

---

### 15. Fixed-Size Stack Buffer in command_line.cc
**File:** `src/clustering/administration/main/command_line.cc`
**Line:** 423
**Issue:** Fixed-size buffer for user input

```cpp
char buf[1024];
// Used for reading user input
```

**Description:** Fixed-size buffer that could overflow if input exceeds 1024 characters.

**Recommended Fix:** Use `std::string` with `std::getline` instead of fixed buffers.

---

### 16. Potential Integer Overflow in cJSON.cc
**File:** `src/cjson/cJSON.cc`
**Line:** 344
**Issue:** Integer overflow in length calculation

```cpp
ptr=str;while ((token=*ptr) && ++len) {if (strchr("\"\\\b\f\n\r\t",token)) len++; else if (token<32) len+=5;ptr++;}
out=(char*)cJSON_malloc(len+3);
```

**Description:** The length calculation `len+3` could overflow if the string is extremely long (though unlikely in practice).

**Recommended Fix:** Add overflow check or use size_t throughout.

---

### 17. Buffer Access Without Bounds Check in http_parser.cc
**File:** `src/http/http_parser.cc`
**Lines:** Multiple
**Issue:** Manual buffer parsing without bounds checking

The http_parser uses macros that access buffer positions without explicit bounds checking in some paths.

**Recommended Fix:** Ensure all buffer accesses have explicit bounds checks.

---

### 18. Raw Pointer Storage in archive/tcp_conn_stream.cc
**File:** `src/containers/archive/tcp_conn_stream.cc`
**Line:** 22-23
**Issue:** Raw pointer ownership

```cpp
tcp_conn_stream_t::tcp_conn_stream_t(tcp_conn_t *conn) : conn_(conn) {
    rassert(conn_ != nullptr);
}

tcp_conn_stream_t::~tcp_conn_stream_t() {
    delete conn_;
}
```

**Description:** The class takes ownership of a raw pointer. If the caller doesn't know this, they might use the pointer after passing it or delete it themselves.

**Recommended Fix:** Use `std::unique_ptr<tcp_conn_t>` to make ownership explicit.

---

### 19. Memory Leak on Exception in lba_disk_structure.cc
**File:** `src/serializer/log/lba/disk_structure.cc`
**Lines:** 62-67
**Issue:** Allocated extents not cleaned up on exception

```cpp
for (int i = 0; i < startup_superblock_count; i++) {
    extents_in_superblock.push_back(
        new lba_disk_extent_t(em, file,
            startup_superblock_buffer->entries[i].offset,
            startup_superblock_buffer->entries[i].lba_entries_count));
}
```

**Description:** If an exception occurs during the loop, previously allocated extents leak.

**Recommended Fix:** Use `std::unique_ptr` temporarily or exception-safe container insertion.

---

### 20. strcpy/strcat Usage in netrecord.c
**File:** `scripts/netrecord/netrecord.c`
**Line:** 440
**Issue:** sprintf with fixed buffer

```c
char count_buffer[20];
sprintf(count_buffer, "%d", (int)bytes_read);
```

**Description:** If `bytes_read` is very large, this could overflow the 20-byte buffer.

**Recommended Fix:** Use `snprintf`:
```c
snprintf(count_buffer, sizeof(count_buffer), "%d", (int)bytes_read);
```

---

### 21. Potential Use-After-Free in fifo_enforcer.cc
**File:** `src/concurrency/fifo_enforcer.cc`
**Lines:** 65-139
**Issue:** Self-deletion in callbacks

Similar to parallel_traversal.cc, this file uses `delete this` in callbacks which can lead to use-after-free if not carefully managed.

---

### 22. Uninitialized Memory in btree Operations
**File:** `src/btree/internal_node.cc`, `src/btree/leaf_node.cc`
**Issue:** Memcpy of structs with padding

Multiple locations use `memcpy` to copy btree structures that may have padding bytes between fields.

**Recommended Fix:** Use `memset` to zero structures before use or use `memcpy` only on packed structures.

---

### 23. Raw new/delete in Extproc System
**File:** `src/extproc/js_runner.cc`, `src/extproc/http_runner.cc`
**Issue:** Manual memory management

The extproc system uses raw new/delete for job data management which could lead to leaks on exception paths.

---

### 24. Array Access Without Bounds Check in datum_stream.cc
**File:** `src/rdb_protocol/datum_stream.cc`
**Line:** 467, 1691
**Issue:** Direct array indexing

```cpp
pseudoshard_t *best_shard = &pseudoshards[0];
auto subspec = subspecs[0];
```

**Description:** Direct array access without checking if the array is non-empty.

**Recommended Fix:** Add empty check before accessing index 0.

---

## Low Severity Issues

### 25. Use of sprintf in exactfloat.cc
**File:** `src/rdb_protocol/geo/s2/util/math/exactfloat/exactfloat.cc`
**Lines:** 343, 446
**Issue:** sprintf with fixed-size buffers

```cpp
char exp_buf[10];
sprintf(exp_buf, "e%+02d", exp10 - 1);
```

**Description:** While the buffer size appears sufficient for the expected range, there's no explicit bounds checking.

---

### 26. Fixed-Size Buffers in Various Locations
Multiple files use fixed-size stack buffers that could theoretically overflow:
- `src/paths.cc`: Line 212 (char buf[4096])
- `src/clustering/administration/main/names.cc`: Line 116 (char h[64])
- `src/clustering/administration/main/cache_size.cc`: Line 248 (char str[256])

---

### 27. Potential Memory Leak in tls_helper.hpp
If TLS initialization fails partway through, some allocated resources may leak.

---

### 28. Unnecessary malloc/free in backtrace.cc
**File:** `src/backtrace.cc`
**Line:** 223
**Issue:** Uses new[] for stack frames instead of alloca or fixed buffer

```cpp
scoped_array_t<void *> stack_frames(new void*[max_frames], max_frames);
```

While this is safe (uses scoped_array_t), it's unnecessary and could use a fixed buffer or alloca for small frame counts.

---

### 29. Null Pointer Usage in Test Code
Several unittest files use raw pointers without null checks, assuming test data is valid. This is generally acceptable for test code but should be noted.

---

### 30. shared_buf_t::create Potential Integer Overflow
**File:** `src/containers/shared_buffer.cc`
**Line:** 10
**Issue:** Size calculation could overflow

```cpp
size_t memory_size = sizeof(shared_buf_t) + size - 1;
```

If `size` is extremely large (near SIZE_MAX), this could overflow.

---

## General Observations

### Positive Security Practices
1. **Use of rmalloc/rrealloc**: Most allocations use `rmalloc` which crashes on OOM rather than returning NULL, eliminating many null-check issues.
2. **scoped_ptr_t and counted_t**: The codebase uses smart pointer types in many places.
3. **RAII patterns**: Many classes properly implement RAII for resource management.

### Areas for Improvement
1. **Transition to modern C++**: Replace raw new/delete with `std::unique_ptr` and `std::shared_ptr`.
2. **Bounds-checking containers**: Use `std::vector::at()` instead of `operator[]` where performance allows.
3. **String handling**: Replace all `strcpy`/`sprintf` with `strncpy`/`snprintf` or use `std::string`.
4. **Static analysis**: Integrate static analyzers like clang-analyzer, Coverity, or PVS-Studio into CI.

---

## Recommendations Summary

### Immediate Actions (Critical/High)
1. Fix strcpy/sprintf issues in cJSON.cc
2. Add exception handling in semaphore.cc
3. Review and fix `delete this` patterns
4. Add bounds checking to decode_le32/decode_le64
5. Fix potential memory leaks in exception paths

### Short-term Actions (Medium)
1. Replace fixed-size buffers with dynamic allocation where appropriate
2. Add overflow checks to size calculations
3. Review all memmove/memcpy operations for bounds safety
4. Modernize memory management in pathspec.cc

### Long-term Actions (Low)
1. Migrate to smart pointers throughout
2. Enable compiler warnings: `-Wstack-protector`, `-D_FORTIFY_SOURCE=2`
3. Integrate AddressSanitizer and MemorySanitizer in testing
4. Regular security audits with static analysis tools

---

## Appendix: Files Requiring Attention

### Critical Priority
- `src/cjson/cJSON.cc`
- `src/concurrency/semaphore.cc`
- `src/btree/parallel_traversal.cc`

### High Priority
- `src/rdb_protocol/pathspec.cc`
- `src/containers/shared_buffer.cc`
- `src/unittest/unittest_utils.cc`
- `src/concurrency/wait_any.cc`
- `src/backtrace.cc`

### Medium Priority
- `src/btree/internal_node.cc`
- `src/btree/leaf_node.cc`
- `src/serializer/log/lba/disk_structure.cc`
- `src/containers/archive/tcp_conn_stream.cc`
- `src/clustering/administration/main/command_line.cc`

---

*End of Report*
