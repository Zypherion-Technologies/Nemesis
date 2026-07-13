# Nemesis
[![Website](https://img.shields.io/badge/zypherion.tech-1f6feb?logo=googlechrome&logoColor=white)](https://www.zypherion.tech)
[![Discord](https://img.shields.io/badge/Discord-5865F2?logo=discord&logoColor=white)](https://discord.gg/JXx32jKJXY)
[![Telegram](https://img.shields.io/badge/Telegram-26A5E4?logo=telegram&logoColor=white)](https://t.me/zypherion_technologies)
[![X](https://img.shields.io/badge/Follow-000000?logo=x&logoColor=white)](https://x.com/Zypherion_Tech)

windows tool i made cuz i got tired of staring at jlaive forks in malware samples with nothing public to actually rip them apart at runtime.

in short, it's a .net process monitor that hooks the clr at the native layer, tracks reflective assembly loads, and automatically dumps pes directly from memory. it also checks amsi and etw integrity against the original on disk binaries while detecting bypass techniques such as `clr.dll` string patching and direct `LoadFromBuffer` usage.

> where to find samples ? i recommend checking out https://tria.ge (not an ad) and u can download any sample that u want and filter based on a family 

## why the name?

you might be wondering, why the name nemesis?

because it represents the exact purpose of the tool. a nemesis is something that stands as a constant challenge or downfall for an opponent and that's the idea behind this project. 

in greek myth [nemesis](https://en.wikipedia.org/wiki/Nemesis_(mythology)) was the spirit of divine retribution the one who handed out payback to anyone who got too arrogant or thought they were untouchable. bit on the nose for malware that brags about being "fud" if you ask me.

## why this exists

im a malware analyst. if you do this job(hobby for me) long enough you start seeing the same loader chains over and over especially since jlaive (also called crybat) blew up and every script kiddy forked it.

the pattern is stupid simple but annoying as hell to deal with:

```
something.bat  →  obfuscated powershell  →  csharp stub  →  your actual payload
```

you double-click a `.bat` that looks like word salad. cmd kicks off powershell with a wall of junk. powershell decrypts/decompresses a .net stub (aes, gzip, base64). that stub patches amsi + etw, reflectively loads the real exe/dll in memory, and youre done nothing friendly ever hits disk in a useful form.

that bugged me. there isn't really a good **public** tool aimed at fighting this specific chain hooking where the .net payload actually materializes, dumping it before the process eats itself, catching the amsi/etw patches these stubs always do. so i built nemesis.

not a silver bullet. not gonna replace your sandbox. but it gives you something real to run against a suspicious `.bat` on a test box and actually pull artifacts out.

## what it actually does

**launcher** (`Launcher.exe`)
- creates your target suspended (exe, bat, cmd, whatever)
- injects `Nemesis.dll` before main thread runs
- waits for nemesis to say it's ready
- resumes. if the target spawns powershell kids, launcher can inject those too

**nemesis dll** (`Nemesis.dll`)
- hooks clr load paths that jlaive-style stubs actually hit:
  - `nLoadImage`
  - `nLoadFile`
  - `AssemblyNative::LoadFromBuffer` (pattern resolved off nLoadImage)
- when a reflective pe shows up in memory -> queues a dump to `%TEMP%\Nemesis_dumps`
- compares live `amsi.dll` / `ntdll.dll` exports against on disc copies (catches the classic `ret` patch on `AmsiScanBuffer` / `EtwEventWrite`)
- amsi check also looks at clr `.rdata` strings when clr is loaded (some bypasses patch those instead, theres a great vxug paper on it called; `2024-11-21 - New AMSI Bypss Technique Modifying CLRDLL in Memory.pdf` )
- logs to console (colored) + `%TEMP%\Nemesis.log` it has also `ENABLE_VIRTUAL_TERMINAL_PROCESSING` to avoid issues with console being lets say weird :D

basically: let the bat chain run, catch the payload where the crypter actually loads it, and log the evasion tricks on the way.

## Proof: 
<img width="840" height="166" alt="image" src="https://github.com/user-attachments/assets/a25c4b88-772e-4497-b231-5d32e121f876" />
<img width="1071" height="449" alt="image" src="https://github.com/user-attachments/assets/550f1691-3886-41e1-87d0-1288bdd6c41d" />
<img width="1261" height="614" alt="image" src="https://github.com/user-attachments/assets/29237d07-9c33-479b-9747-fddb1b676996" />

## build

need visual studio 2022+ with c++ desktop + masm (x64).

open `Nemesis.slnx`, pick **Release | x64**, build solution.

## run on a real target

this is the actual use case point it at a suspicious bat and see what falls out:

```powershell
cd x64\Release
.\Launcher.exe "C:\path\to\suspicious.bat"
```

extra args after `--` get passed to the target:

```powershell
.\Launcher.exe myapp.exe -- --some-flag
```

custom dll path:

```powershell
.\Launcher.exe --dll C:\path\Nemesis.dll myapp.exe
```

**artifacts:**
- logs → `%TEMP%\Nemesis.log`
- dumped pes → `%TEMP%\Nemesis_dumps`

## what nemesis is / isn't

**is:**
- a runtime analysis helper for jlaive style crypters (bat → ps1 → csharp chains)
- useful on a lab vm when you have a sample and want in memory dumps + amsi/etw telemetry
- open for other analysts to use, extend, complain about

**isn't:**
- an edr replacement
- guaranteed to catch every fork variant (obfuscation changes, new bypasses, e.g patchless amsi/etw beacuse its not made for it as of now and i plan on changing that in future
maybe this will turn into some managed lang toolkit)

## notes

- x64 only for the clr asm detours right now
- cmd.exe hosts often have no clr nemesis signals ready anyway so launcher doesn't hang, real work happens when powershell/dotnet shows up
- rebuild fails with `LNK1104`? something still has nemesis.dll loaded kill the target and rebuild
- `pwsh.exe` vcpkg noise during build is harmless, ignore it

## further reading (the jlaive ecosystem)

if you wanna understand what you're fighting:
- [Trend Micro on BatCloak / Jlaive](https://www.trendmicro.com/en_us/research/23/b/attackers-target-microsoft-users-with-batcloak-engine.html) — the bat → ps1 → csharp layer cake
- [Fortinet on ScrubCrypt](https://www.fortinet.com/blog/threat-research/scrubcrypt-deploys-venomrat-with-arsenal-of-plugins) — jlaive's annoying successor if ur interested what was little diff about scrubcrypt it was the fact that they piped commands out with the pipe sign in .bat it works pretty well...
- [Unprotect.it — ScrubCrypt](https://unprotect.it/technique/scrubcrypt/) — technique breakdown
- https://github.com/backdoorskid/ClrAmsiScanPatcher - finds the .net string as mentioned before i recommended reading that .pdf file :)


## disclaimer

**by using nemesis you accept this.** it's provided **as-is** with **no warranty** — you assume all risk.

you're solely responsible for lawful, authorized use (lab vms, owned systems, explicit permission). to the maximum extent permitted by law, the authors and [zypherion.tech](https://zypherion.tech) **disclaim all liability** for any damages, losses, or legal claims arising from use or misuse. see [LICENSE](LICENSE) for full terms.

## license

**non-commercial / personal / research / hobby use** → [PolyForm Noncommercial 1.0.0](LICENSE)

**commercial use** (selling it, paid product, saas, client work, etc.) → you need a separate license. polyform noncommercial doesn't cover that.

hit me up if you want a commercial license:

**[adam@zypherion.tech / @wd6g(discord) / telegram: @ZypherionTechnologies]**

# todo

> quick note i have to look into compilemethod and fix it as of now its not somewhat needed as crypters dont usually use it at all... since they have to use `asm.load(...)`
> also reminds me i have to check rdata str as i havent test that properly sadly...
> dont do a PR with a dumb code please, u cannot hook n(Native) backends e.g nLoadImage normall u have to preserve registers and its just meh, thats why we use asm. 
