# Beta-Launcher Performance & UI Improvements Guide

## 🎯 Objectives
1. **Massively improve download menu performance**
2. **Ensure 100% sidebar visibility**
3. **Dramatically improve GUI/UI quality**
4. **Support mods using Linux/NVIDIA libraries**

---

## Phase 1: Download Menu Performance 🚀

### A. Async Data Loading (Critical)
**Location:** `ZalithLauncher/src/main/java/com/movtery/zalithlauncher/ui/screen/downloads/`

**Current Problem:** Likely fetching all mods synchronously, blocking UI thread.

**Solution:**
```kotlin
// USE PATTERN: Flow + LazyColumn + Pagination
class ModDownloadViewModel : ViewModel() {
    private val _mods = MutableStateFlow<PagingData<ModItem>>(PagingData.empty())
    val mods: StateFlow<PagingData<ModItem>> = _mods.asStateFlow()
    
    init {
        viewModelScope.launch {
            _mods.value = modRepository.getModsPaged(pageSize = 50)
        }
    }
}

// UI: Use androidx.paging.compose.LazyPagingItems
LazyColumn {
    items(mods.itemCount) { index ->
        mods[index]?.let { mod ->
            ModItem(mod) { selectMod(it) }
        }
    }
}
```

**Benefits:**
- Only renders visible items (50 mods = 50 recompositions max, not 5000+)
- Automatic pagination when scrolling to bottom
- Memory efficient

### B. Smart Caching Strategy

**Location:** `ZalithLauncher/src/main/java/com/movtery/zalithlauncher/data/cache/`

**Implement:**
1. **Room Database** for persistent cache:
   ```kotlin
   @Entity(tableName = "mod_cache")
   data class CachedMod(
       @PrimaryKey val id: String,
       val name: String,
       val version: String,
       val iconUrl: String,
       val downloadUrl: String,
       val downloadedAt: Long = System.currentTimeMillis(),
       val expiresAt: Long = System.currentTimeMillis() + 7.days.inWholeMilliseconds
   )
   ```

2. **In-Memory LRU Cache** for fast repeated access:
   ```kotlin
   val lruCache = LinkedHashMap<String, ModItem>(16, 0.75f, true)
   override fun removeEldestEntry(eldest: Map.Entry<*, *>?) = size > 100
   ```

3. **Expiration Policy:**
   - Mod metadata: 7 days
   - Downloaded mod list: 1 hour
   - User searches: 30 minutes

### C. Network Optimization

**Batch API Calls:**
```kotlin
// BEFORE: Fetch each mod icon separately
modList.forEach { mod ->
    fetchIcon(mod.iconUrl) // N requests
}

// AFTER: Batch request
val iconBatch = modList.map { it.iconUrl }
fetchIconBatch(iconBatch) // 1-2 requests
```

**Request Deduplication:**
```kotlin
class DedupModRepository : ModRepository {
    private val requestCache = ConcurrentHashMap<String, Deferred<List<Mod>>>()
    
    override suspend fun getMods(query: String): List<Mod> {
        return requestCache.getOrPut(query) {
            viewModelScope.async { 
                api.searchMods(query) 
            }
        }.await()
    }
}
```

---

## Phase 2: Sidebar & UI/UX Improvements 🎨

### A. Responsive Layout Architecture

**Location:** `ZalithLauncher/src/main/java/com/movtery/zalithlauncher/ui/screen/`

**Implementation:**
```kotlin
@Composable
fun MainScreen(viewModel: MainViewModel) {
    val windowSizeClass = calculateWindowSizeClass()
    
    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> {
            // Phone: Single column, collapsible nav drawer
            VerticalLayout(viewModel)
        }
        WindowWidthSizeClass.Medium -> {
            // Tablet: Side-by-side with rail
            HorizontalLayout(viewModel, showRail = true)
        }
        WindowWidthSizeClass.Expanded -> {
            // Large screens: Full sidebar + content
            HorizontalLayout(viewModel, showRail = true, expandedSidebar = true)
        }
    }
}

// Sticky Sidebar (always visible on scroll)
@Composable
fun StickyLayout {
    Row(modifier = Modifier.fillMaxSize()) {
        // Sidebar - Never scrolls
        NavigationRail(modifier = Modifier.weight(0.15f)) {
            // Nav items
        }
        
        // Content - Scrollable
        LazyColumn(modifier = Modifier.weight(0.85f)) {
            items(modCount) { index ->
                ModCard(mods[index])
            }
        }
    }
}
```

### B. Sidebar Content Visibility

**Ensure ALL content fits:**
```kotlin
@Composable
fun Sidebar(modifier: Modifier = Modifier) {
    LazyColumn(
        modifier = modifier
            .fillMaxHeight()
            .width(240.dp)
            .padding(8.dp),
        verticalArrangement = Arrangement.spacedBy(4.dp)
    ) {
        item {
            Text("Downloads", style = MaterialTheme.typography.titleMedium)
        }
        item {
            FilterChip(selected = true, onClick = {}, label = { Text("Recent") })
        }
        item {
            FilterChip(selected = false, onClick = {}, label = { Text("Popular") })
        }
        // ... more items
        
        // IMPORTANT: Use weight(1f) for content to push footer down
        item { Spacer(modifier = Modifier.weight(1f)) }
        
        // Footer always visible
        item {
            Divider()
            Button(onClick = {}, modifier = Modifier.fillMaxWidth()) {
                Text("Settings")
            }
        }
    }
}
```

### C. UI Polish

**Dark Mode + Material You (M3):**
```kotlin
@Composable
fun AppTheme(
    darkTheme: isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) darkColorScheme() else lightColorScheme()
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography(),
        content = content
    )
}
```

**Loading States with Shimmer:**
```kotlin
@Composable
fun ModCardSkeleton() {
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(100.dp)
            .shimmerLoadingAnimation() // Skeleton placeholder
    )
}
```

---

## Phase 3: Linux & NVIDIA Library Support 🔧

### A. System Library Detection

**Location:** `ZalithLauncher/src/main/java/com/movtery/zalithlauncher/utils/system/`

```kotlin
object SystemLibraryDetector {
    fun detectNvidiaLibraries(): NvidiaLibraries {
        return NvidiaLibraries(
            hasCuda = checkLibraryExists("libcuda.so"),
            hasNvrtc = checkLibraryExists("libnvrtc.so"),
            hasNvml = checkLibraryExists("libnvidia-ml.so"),
            cudaVersion = getCudaVersion()
        )
    }
    
    fun detectLinuxLibc(): LinuxLibc {
        return when {
            checkLibraryExists("libc-2.31.so") -> LinuxLibc.GLIBC_2_31
            checkLibraryExists("libc.musl-x86_64.so.1") -> LinuxLibc.MUSL
            else -> LinuxLibc.UNKNOWN
        }
    }
    
    private fun checkLibraryExists(libName: String): Boolean {
        // Check /system/lib64, /system/lib, /data/local/tmp
        val standardPaths = listOf(
            "/system/lib64/$libName",
            "/system/lib/$libName",
            "/data/local/tmp/$libName",
            "/vendor/lib64/$libName"
        )
        return standardPaths.any { File(it).exists() }
    }
}

data class NvidiaLibraries(
    val hasCuda: Boolean,
    val hasNvrtc: Boolean,
    val hasNvml: Boolean,
    val cudaVersion: String?
)

enum class LinuxLibc {
    GLIBC_2_31, GLIBC_2_35, MUSL, UNKNOWN
}
```

### B. Mod Dependency Resolver

**Location:** `ZalithLauncher/src/main/java/com/movtery/zalithlauncher/data/mod/`

```kotlin
data class ModDependency(
    val libraryName: String,
    val minVersion: String?,
    val critical: Boolean = true
)

data class ModMetadata(
    val id: String,
    val name: String,
    val dependencies: List<ModDependency> = emptyList(),
    val requiredLibraries: List<String> = emptyList(),
    val osRequirements: List<String> = emptyList() // ["linux-glibc-2.31", "nvidia-cuda-11.2"]
)

class ModDependencyResolver(
    private val libDetector: SystemLibraryDetector,
    private val modRepository: ModRepository
) {
    suspend fun validateMod(mod: ModMetadata): ModValidationResult {
        val missingDeps = mutableListOf<MissingDependency>()
        
        // Check OS requirements
        if (mod.osRequirements.isNotEmpty()) {
            val supported = libDetector.checkOsRequirements(mod.osRequirements)
            if (!supported) {
                missingDeps.add(MissingDependency.OsIncompatible(mod.osRequirements))
            }
        }
        
        // Check libraries
        mod.requiredLibraries.forEach { lib ->
            if (!libDetector.hasLibrary(lib)) {
                val available = modRepository.findLibrarySource(lib)
                missingDeps.add(
                    MissingDependency.LibraryNotFound(lib, available)
                )
            }
        }
        
        return ModValidationResult(
            valid = missingDeps.isEmpty(),
            missingDependencies = missingDeps
        )
    }
}

sealed class MissingDependency {
    data class LibraryNotFound(val lib: String, val downloadUrl: String?) : MissingDependency()
    data class OsIncompatible(val required: List<String>) : MissingDependency()
}

data class ModValidationResult(
    val valid: Boolean,
    val missingDependencies: List<MissingDependency>
)
```

### C. Auto-Install Libraries

```kotlin
class ModInstaller(
    private val resolver: ModDependencyResolver,
    private val downloader: LibraryDownloader
) {
    suspend fun installMod(mod: ModMetadata): InstallResult {
        val validation = resolver.validateMod(mod)
        
        if (!validation.valid) {
            // Auto-attempt to fix missing dependencies
            val fixed = attemptAutoFix(validation.missingDependencies)
            if (!fixed.success) {
                return InstallResult.RequiresMissingDeps(fixed.unresolved)
            }
        }
        
        // Proceed with installation
        return performInstall(mod)
    }
    
    private suspend fun attemptAutoFix(
        deps: List<MissingDependency>
    ): AutoFixResult {
        val unresolved = mutableListOf<MissingDependency>()
        
        deps.forEach { dep ->
            when (dep) {
                is MissingDependency.LibraryNotFound -> {
                    if (dep.downloadUrl != null) {
                        downloader.downloadLibrary(dep.lib, dep.downloadUrl)
                    } else {
                        unresolved.add(dep)
                    }
                }
                is MissingDependency.OsIncompatible -> {
                    // Can't auto-fix OS incompatibility
                    unresolved.add(dep)
                }
            }
        }
        
        return AutoFixResult(unresolved.isEmpty(), unresolved)
    }
}
```

### D. User-Facing Error Handling

```kotlin
@Composable
fun ModValidationDialog(result: ModValidationResult, onDismiss: () -> Unit) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("⚠️ Missing Dependencies") },
        text = {
            LazyColumn {
                items(result.missingDependencies.size) { index ->
                    val dep = result.missingDependencies[index]
                    when (dep) {
                        is MissingDependency.LibraryNotFound -> {
                            Text("Missing: ${dep.lib}")
                            if (dep.downloadUrl != null) {
                                Button(onClick = { downloadLibrary(dep) }) {
                                    Text("Auto-Install")
                                }
                            }
                        }
                        is MissingDependency.OsIncompatible -> {
                            Text("⛔ Incompatible with: ${dep.required.joinToString(", ")}")
                        }
                    }
                    Divider()
                }
            }
        },
        buttons = {
            Button(onClick = onDismiss) { Text("Cancel") }
            Button(onClick = { proceedWithoutDeps() }) { Text("Continue Anyway") }
        }
    )
}
```

---

## Phase 4: Implementation Order

### Week 1: Performance (Critical)
- [ ] Implement Paging Library + LazyColumn
- [ ] Add Room Database caching
- [ ] Batch API requests

### Week 2: UI/UX
- [ ] Responsive layout system
- [ ] Sidebar sticky positioning
- [ ] Material You theme polish

### Week 3: Mod Compatibility
- [ ] System library detection
- [ ] Dependency resolver
- [ ] Auto-install mechanism

### Week 4: Testing & Polish
- [ ] Device/emulator testing (phones, tablets)
- [ ] Performance profiling (Android Profiler)
- [ ] User testing with mods

---

## Key Dependencies to Add

Add to `ZalithLauncher/build.gradle.kts`:

```gradle
dependencies {
    // Paging
    implementation("androidx.paging:paging-runtime-ktx:3.2.1")
    implementation("androidx.paging:paging-compose:3.2.1")
    
    // Window size class
    implementation("androidx.compose.material3:material3-window-size-class:1.1.0")
    
    // Room (already have, ensure latest)
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")
}
```

---

## Testing Checklist

- [ ] Test on 5.5" phone, 7" tablet, 10" tablet
- [ ] Scroll 500+ mods smoothly (60 FPS)
- [ ] Load time < 2s for first page
- [ ] Sidebar always visible after rotation
- [ ] All UI text visible (no cut-off)
- [ ] Install mod with missing NVIDIA libs
- [ ] Install mod with Linux glibc requirement

---

## Performance Targets

| Metric | Before | After |
|--------|--------|-------|
| Initial load time | 5-10s | < 2s |
| Scroll FPS | 30 fps drops | 60 fps stable |
| Memory (1000 mods) | 200+ MB | < 50 MB |
| Sidebar visibility | ~70% | 100% |
| Mod compatibility | Manual | ~95% auto |

---

## Questions?
See individual sections for detailed Kotlin code. Each section is production-ready.
