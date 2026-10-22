# The following is a verbatim copy of a bug report, following thorough investigative efforts by your fellow players.
# LunatiK and Sculas are a C++ developers, and the result of their combined diagnostic effort is catalogued here to explain the E&CC Linux crash situation. Ultimately, this is a failing of the Clausewitz engine, and while we can do our best to work around it, it is up to Paradox to fix. This report has been included here for the sake of explaining the issue to future E&CC developers and Linux players.
# The following bug report was filed on the Paradox forums by co-discoverer Sculas, and has been included for posterity.
# -- Sage (Oct 22, 2026)

# https://forum.paradoxplaza.com/forum/threads/mod-redefining-a-vanilla-building-leaves-stale-modifier-registry-entries-causing-a-use-after-free-write-during-loading-crash-on-linux.1946218/
# Integrity​
# I have verified my game files (on Steam)​
# Yes
# 
# I have disabled all mods​
# No
# 
# Required​
# Issue type​
# Crash
# 
# Summary​
# Mod redefining a vanilla building leaves stale modifier registry entries, causing a use-after-free write during loading (crash on Linux)
# 
# Description​
# When a mod defines a building key that already exists in the base game, the engine destroys the old building and its modifier records. It does not remove their entries from a global registry that the modifier: parser fills (variable_values.cpp). Right after loading, a loop in the data initialisation function writes the modifier id into entry->object + 0x1A8 for every entry whose object pointer is not null. That includes the entries of destroyed objects, so this is a use-after-free write.
# 
# What the write hits depends on the allocator. On Linux it lands on live objects and the game crashes at the end of loading. On Windows the same writes land on memory nobody uses and nothing visible happens. A Windows 11 test of the same playset did not crash, presumably because the operating system memory allocators differ.
# 
# This is not specific to Wine or Proton. A crash bundle posted in https://forum.paradoxplaza.com/foru...e-to-memory-allocation-issues-likely.1945225/ is from the native Linux build (4.5.2, Steam, one mod called Ethics and Civics Classic). Its stack trace ends in CTriggerCollectionBase::CheckScopeSupportRecursivePrintError, called from CScriptedTrigger::postValidate, CTriggerDatabase::postValidate and CRandomNameDatabase::FillDatabase. That is the same function that crashed in some of my dumps (stellaris.exe+0x3465D0 in the Windows build). The error.log in that bundle says that building_hab_capital, building_hab_major_capital and building_hab_system_capital "already exist, using the one at" ethic_00_capital_buildings.txt. These are the same buildings my reproducer replaces.
# 
# Replacing a definition is supported behaviour. The engine logs "Object with key: X already exists, using the one at ..." and keeps the later definition, and every mod that overrides a vanilla building depends on that. A mod author cannot avoid the bug except by never replacing a key. It also cannot be shown with mods disabled, because the trigger is a second definition of an existing key and all 498 building keys in 4.5.2 are defined once. The attached mod is the game's own unedited text for two buildings, defined a second time.
# 
# Root cause in detail (Windows build 4.5.2, image base 0x140000000):
# 1. The parser for modifier:<name> expressions (RVA 0x34C110) creates a global registry the first time it runs. The registry pointer is at RVA 0x3114ED8, with an array of entry pointers at +0x08 and the count at +0x14. For every reference it allocates a 0x68 byte entry. The first field of the entry is a pointer to the record object being parsed.
# 2. The record class destructor (RVA 0x34AAE0) resets the vptr to the base vtable (0x2344D10) and frees two std::string members. It does not touch the registry.
# 3. The loop in the data initialisation function (RVA 0x1B91F0) looks up each entry's name and runs "mov rax,[rsi] ; mov [rax+1A8h],edx" at RVA 0x1B9CFB. It only skips entries whose object pointer is zero.
# 4. With the attached mod, 14 entries point at destroyed objects. They all come from vanilla building_hab_major_capital and building_hab_system_capital in common/buildings/00_capital_buildings.txt. In one run the registry had 485 entries. 483 had a normal vptr. One had the base vtable that the destructor leaves behind, and one pointed at memory that now held ASCII text.
# 5. I confirmed this with gdb and with a diagnostic DLL. Clearing the registry entries when their object is destroyed removes exactly those 14 entries, and the crash goes away.
# 
# What the stray write overwrites: The post-load loop stores a modifier id, which is a small number. I caught one of these stores in a run with a larger mod set, not the attached mod. I made the memory page holding a trigger's children array read-only, and the first write to it faulted at stellaris.exe+0x1B9CFB. At that point rax pointed into a freed block and the value being stored was 0x1770E. The children slot had held a valid pointer 8 seconds earlier. The trigger loop at trigger.cpp:499 then calls a virtual function on that child without checking it, and faults when it reads the small number. With the attached mod, the dumps show other ids, such as 0xB0D0.
# 
# Suggested fix: In the destructor of the record class, remove every registry entry whose first field is this, or set that field to null, because the post-load loop already skips null. The same clean-up could go in the code that replaces an existing object ("already exists, using the one at ..."). A more robust design would tie registration to the owner's lifetime, or store a handle that is checked before use. In debug builds, the post-load loop could assert that the object's vptr is not the base vtable.
# 
# Disclosure: I did most of the work (testing, debugging, etc.) myself, but an LLM was used to help with the writing of this report.
# 
# Expected Result​
# The second definition replaces the first and the game finishes loading. When the old building's records are destroyed, their entries are also removed from the modifier registry, so nothing writes into freed memory.
# 
# Actual Result​
# The registry keeps 14 entries that point at destroyed objects, and the post-load loop writes a modifier id into them. On Linux (Proton) the game crashes at about 100% loading with EXCEPTION_ACCESS_VIOLATION, reading a small number that is a modifier id. Where it crashes depends on heap layout.
# 
# The 8 attached dumps (Windows build under Proton with the repro mod) fall on three places:
# stellaris.exe+0x346653, 1 dump, read of 0xB0D9, in the trigger loop mentioned above
# stellaris.exe+0x1BB7F50, 5 dumps, read of 0xB0E0, a recursive node destroy routine called with node = 0xB0D0
# ntdll.dll+0x2CA77, 2 dumps, read of 0xB0CE, inside the heap allocator
# 0xB0D0 is the id of the modifier job_soldier_or_warrior_add and 0xB0D9 is habitat_max_districts_add. Both belong to the 14 stale entries.
# 
# error.log on every start contains these two lines:
# Object with key: building_hab_major_capital already exists, using the one at file: common/buildings/zz_repro.txt line: 14
# Object with key: building_hab_system_capital already exists, using the one at file: common/buildings/zz_repro.txt line: 160
# 
# Steps to Reproduce​
# 1. Unzip stellaris_repro_mod.zip (attached) into "Documents/Paradox Interactive/Stellaris/mod/".
# 2. Open the Paradox launcher and create a playset that contains only "Repro: redefine the habitat capitals".
# 3. Start the game. Start it several times, because whether it crashes depends on heap layout.
# 4. Open logs/error.log. The two "already exists" lines above appear on every start.
# 5. On Linux the game crashes at about 100% loading in some of the runs.
# 6. On Windows the game most likely loads fine, because the stale writes land on unused memory. Run stellaris.exe with the attached mod under full page heap (gflags /p /enable stellaris.exe /full, or Application Verifier). I expect to see 14 access violations at stellaris.exe+0x1B9CFB, one for each stale entry. Without a tool, set a breakpoint on that instruction and check whether the object being written to (rax) has the base-class vtable 0x142344D10. That means the object has already been destroyed.
# 
# Game version​
# 4.5.2 Cygnus, commit caa600af4934c86bd6c1862632e2500e7d6163f6
# 
# Additional​
# Operating System​
# Linux
# 
# Platform​
# Steam
# 
# Repro Rate​
# Intermittent (2/3)
# 
# Attachment​
# View attachment crash_dumps.zipView attachment stellaris_repro_mod.zipView attachment error_log_excerpt.txt# 

### Special thanks to the following users for submitting bug reports to the Linux Crash channel on our Discord:
# Camille1358
# comrade_nord
# CrazyUncleSole
# DarthDemetrius
# Geolykt
# hexsaber
# McMoto
# Nakonin
# Salthin
# SanCastXP
# TakeOnLuna
# !WerTerKer12