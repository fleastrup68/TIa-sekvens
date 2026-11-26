# Sekvensskabelon 16 step – TIA Portal V21 (SCL) – version 2.1

## Filer

| Fil | Indhold |
|---|---|
| `01_typ_SekvensLog.udt` | Datatype for én linje i step-loggen |
| `02_typ_SekvensData.udt` | Datatype for step-nr, steptimer og tællere |
| `03_typ_SekvensHMI.udt` | Datatype for HMI-interfacet (kommandoer + status) |
| `04_FC_SekvensTekst.scl` | To hjælpe-FC'er der laver loglinjer som tekst til HMI |
| `05_FB_Sekvens16.scl` | Sekvensmotoren – genbruges **uændret** i alle sekvenser |
| `06_FB_MinSekvens.scl` | **Skabelonen** – kopiér og ret for hver ny sekvens |
| `07_Sekvens_Data.db` | Global DB (RETAIN) med step-nr og log for hver sekvens |
| `08_Sekvens_HMI.db` | Global DB (ikke retain) med HMI-interface for hver sekvens |
| `09_MinSekvens_DB.db` | Instans-DB til skabelonen |

## Import i TIA Portal

1. Projekttræ → PLC → **External source files** → *Add new external file* → vælg alle 9 filer.
2. Højreklik på filerne **i rækkefølge 01 → 09** → *Generate blocks from source*.
3. Kald i OB1 (Main):
   ```
   "MinSekvens_DB"(i_Prefix := 'SPVA12',
                   i_Start := "Start_PB",
                   i_Reset := "Reset_PB",
                   i_Hold  := NOT "Auto",
                   i_NiveauHoej := "LSH_101",
                   ...
                   io_Data := "Sekvens_Data".MinSekvens,
                   io_Log  := "Sekvens_Data".MinSekvens_Log,
                   io_HMI  := "Sekvens_HMI".MinSekvens);
   ```
4. Tjek i `Sekvens_Data` at **Retain** er sat på begge linjer.

Virker på S7-1500 og S7-1200 (bruger `RUNTIME` og `RD_LOC_T`).

## Hvorfor InOut

Step-nr, steptimer og log ligger i den globale DB `Sekvens_Data` – **ikke** i instans-DB'en.
Når du tilføjer en ind-/udgang i `FB_MinSekvens` og downloader, reinitialiseres instans-DB'en,
men step og log bevares. Efter en reinit (eller spændingsgenkomst) står sekvensen i pause i
sit gemte step (`o_RestartHold`) og aktuatorerne er slukket, indtil operatøren giver
**Start** (fortsæt) eller **Reset** (step 0).

Ønsker du ikke loggen i retain-hukommelsen (~4 kB), fjern Retain på `MinSekvens_Log`.

## Sådan skrives et step

Hvert step har sin egen `REGION` i `FB_MinSekvens`. Skriv dit eget navn i overskriften –
foldet sammen læses blokken som en liste over sekvensen.

```
REGION Step 02 - Start omrører
    #Tekst[2]  := 'Start omrører';                    // tekst til HMI / log
    #MaxTid[2] := T#10S;                              // timeout-alarm (udelad = ingen)
    #Trans[2]  := #i_M1_Koerer AND #StepTid >= T#2S;  // skiftebetingelse
END_REGION
```

| Hvad | Hvordan |
|---|---|
| Skiftebetingelse | `#Trans[n]` = betingelse for at *forlade* step n. `#Trans[0]` = startbetingelser |
| Steptimer | `#StepTid` (TIME). Nulstilles ved hvert stepskift, står stille i pause |
| Step bits | `#S[n]` – brug dem i `REGION Udgange` til ventiler og motorblokke |
| Første cyklus i step | `#Seq.o_StepEntry` (1-cyklus puls) |
| Næste step | `#Naeste[n]`: udelad = n+1, 1..16 = spring, **99 = sekvens slut** |
| Ubrugte step | Lad dem stå med tom `#Tekst[n] := ''` (skjules på HMI). Sæt `#Naeste[sidste brugte] := 99` |
| Ingen timeout | Udelad `#MaxTid[n]`-linjen (eller skriv `T#0MS`) |
| Spring | `#JumpTo := 7;` springer direkte til step 7 (prioritet over transition) |
| Reset til 0 | `i_Reset := TRUE` – altid step 0, kvitterer alarm, logges |

## Indgange (program)

| Indgang | Funktion |
|---|---|
| `i_Prefix` | Prefix (maks 20 tegn), fx `'SPVA12'`. Står først i alarmteksten og vises på HMI |
| `i_Start` | Flanke: start fra step 0 / fortsæt efter genstart-pause |
| `i_Hold` | Pause – ingen stepskift, steptimer stopper |
| `i_Reset` | Til step 0, kvitterer alarm |
| `i_StepMode` + `i_StepConfirm` | Step-for-step ved idriftsætning |
| `i_AutoRestart` | Efter sidste step → step 1 i stedet for 0 |
| `i_AlarmAck` | Kvittér timeout |
| `i_ForceStep` + `i_Force` | **Tvang**: flanke på `i_Force` sætter sekvensen i step `i_ForceStep` (0..16) – også fra step 0, under pause og i genstart-pause. Logges med årsag 6 |

Programmets indgange og HMI'ens kommandoer virker side om side (OR).

## HMI-interface (`Sekvens_HMI.MinSekvens`)

### Kommandoer HMI → PLC

| Tag | Type | HMI-element | Funktion |
|---|---|---|---|
| `Cmd_Start` | Bool | Knap, *SetBit* | Start / fortsæt |
| `Cmd_Reset` | Bool | Knap, *SetBit* | Reset til step 0 |
| `Cmd_StepConfirm` | Bool | Knap, *SetBit* | Næste step i step-mode |
| `Cmd_AlarmAck` | Bool | Knap, *SetBit* | Kvittér timeout |
| `Cmd_Force` | Bool | Knap, *SetBit* | Tving til `ForceStep` |
| `Cmd_SletLog` | Bool | Knap, *SetBit* (gerne med bekræft) | Slet loggen |
| `ForceStep` | Int | I/O-felt 0..16 | Step der skal tvinges til |
| `Sel_Hold` | Bool | Kontakt, *InvertBit* | Pause |
| `Sel_StepMode` | Bool | Kontakt, *InvertBit* | Step-for-step |
| `Sel_AutoRestart` | Bool | Kontakt, *InvertBit* | Auto-genstart |

`Cmd_xxx` er pulser: HMI sætter bit med *SetBit*, PLC nulstiller den selv i næste cyklus.

**Tvang fra HMI** virker kun når `ForceTilladt = TRUE` (pause, step-mode, genstart-pause
eller step 0). Brug `ForceTilladt` til at gøre Tving-knappen synlig/aktiv.

### Status PLC → HMI

| Tag | Type | Indhold |
|---|---|---|
| `Step` / `StepTekst` | Int / String | Aktivt step og beskrivelse |
| `StepTid_s` | Real | Tid i aktivt step [s] |
| `MaxTid_s` / `RestTid_s` | Real | Max tid og tid til timeout [s] (0 = ingen overvågning) |
| `TransOK` | Bool | Skiftebetingelsen for aktivt step er opfyldt |
| `Running` `Hold` `RestartHold` `WaitConfirm` | Bool | Tilstand |
| `TimeoutAlarm` / `TimeoutStep` | Bool / Int | Alarm og hvilket step |
| `ForceTilladt` | Bool | Tvang er tilladt nu |
| `AntalCyklus` | DInt | Gennemførte sekvenser |
| `StepAktiv[0..16]` | Bool | Step bits til stepoversigt |
| `StepBrugt[0..16]` | Bool | Step i brug (tekst ikke tom) – til at skjule ubrugte step |
| `Tekst[0..16]` | String | Alle steptekster |
| `MaxTidListe_s[0..16]` | Real | Max tid for alle step |
| `SidsteTid_s[0..16]` | Real | Seneste varighed af hvert step |
| `LogTekst[0..49]` | String[100] | Log som færdige tekstlinjer, `[0]` = nyeste |

Loglinje-format:
```
#0012 26.11 14:03:21 S03>S04 00:05:12 Trans  Opvarm
Nr    Dato  Klokken  Fra>Til TidIStep Årsag  Tekst for forladt step
```

### Forslag til skærmbillede

```
┌──────────────────────────────────────────────────────────────────────────┐
│ SEKVENS: Tank 1                      [KØRER] [PAUSE] [GENSTART] [ALARM]  │
├──────────────────────────────────────────────────────────────────────────┤
│ Step:  03   Opvarm                              Skift OK: ●              │
│ Tid i step:  125.4 s     Max: 1800.0 s     Rest: 1674.6 s                │
│ Cyklusser: 42                                                            │
├────────────────────────────────────┬─────────────────────────────────────┤
│ STEPOVERSIGT                       │ BETJENING                           │
│ ● 00 Hvile - klar til start        │ [ START ]  [ RESET ]  [ KVITTÉR ]   │
│ ● 01 Fyld tank       max  120 s  95│                                     │
│ ● 02 Start omrører   max   10 s   3│ ( ) Pause   ( ) Step-mode           │
│ ◉ 03 Opvarm          max 1800 s    │ ( ) Auto-genstart                   │
│ ● 04 Omrør 10 min              600 │ [ NÆSTE STEP ]  (kun i step-mode)   │
│ ● 05 Tøm tank        max  180 s  42│                                     │
│   ...                              │ Tving til step: [ 05 ]  [ TVING ]   │
│                                    │ (aktiv når ForceTilladt)            │
├────────────────────────────────────┴─────────────────────────────────────┤
│ LOG (nyeste øverst)                                          [SLET LOG]  │
│ #0012 26.11 14:03:21 S02>S03 00:00:03 Trans  Start omrører               │
│ #0011 26.11 14:03:18 S01>S02 00:01:35 Trans  Fyld tank                   │
│ ...                                                                      │
└──────────────────────────────────────────────────────────────────────────┘
```

- Stepoversigt: 17 linjer med lampe (`StepAktiv[n]`), tekstfelt (`Tekst[n]`), `MaxTidListe_s[n]`
  og `SidsteTid_s[n]`. Skjul gerne linjer hvor teksten starter med "Step " (ubrugte).
- Log: 50 tekstfelter (eller en liste/tabel i WinCC Unified) på `LogTekst[0..49]`.
- **WinCC Unified PC:** se [HMI_Unified_Faceplate.md](HMI_Unified_Faceplate.md) for trin-for-trin faceplate.
- Laves skærmen som **faceplate** med et interface af typen `typ_SekvensHMI`, kan den
  genbruges til alle sekvenser – kobl blot faceplaten til `Sekvens_HMI.<sekvensnavn>`.

## Alarm (Program_Alarm)

`FB_Sekvens16` indeholder en `Program_Alarm` for timeout. Den vises i WinCC Unified's
alarmliste og historik uden HMI-alarmopsætning – alle sekvenser får den automatisk.

**Opsætning – én gang, efter import:**

1. Åbn `FB_Sekvens16` → fanen **PLC supervisions & alarms** (eller interfacet → `statPA_Timeout`
   → *Properties*).
2. Alarmtekst:
   ```
   @1%s@ Timeout i step @2%d@: @3%s@
   ```
3. Alarmklasse: fx **Warnings** (uden kvittering) eller **Errors** (med kvittering i Unified).
4. Oversæt PLC og HMI. I Unified-forbindelsen skal *Central alarm management* / PLC-alarmer
   være slået til (standard for S7-1500).

| Ledsageværdi | Indhold |
|---|---|
| `@1%s@` | `i_Prefix`, fx `SPVA12` |
| `@2%d@` | Step med timeout |
| `@3%s@` | Steptekst |

Resultat: **`SPVA12 Timeout i step 3: Opvarm`**

Alarmen kommer ved timeout og går ved Kvittér (`i_AlarmAck` / HMI) eller Reset.
Kræver S7-1500, eller S7-1200 med firmware der understøtter `Program_Alarm` (V4.2+).

## Log (sidste 50 stepskift)

Rådata: `Sekvens_Data.MinSekvens_Log[0..49]` – `[0]` er nyeste. Hver linje: `Nr`, `FraStep`,
`FraTekst`, `TilStep`, `Aarsag`, `TidIStep`, `Tidsstempel` (lokal tid – husk PLC'ens ur/tidszone).

| Aarsag | Betydning |
|---|---|
| 1 | Transition |
| 2 | Spring (JumpTo) |
| 3 | Start |
| 4 | Reset |
| 5 | Sekvens slut |
| 6 | Tvang |

## Ny sekvens

1. Kopiér `FB_MinSekvens` → fx `FB_Tank2_Sekvens`, og lav en ny instans-DB.
2. Tilføj i `Sekvens_Data`: `Tank2 : "typ_SekvensData"` og
   `Tank2_Log : Array[0..49] of "typ_SekvensLog"` (begge Retain).
3. Tilføj i `Sekvens_HMI`: `Tank2 : "typ_SekvensHMI"`.
4. Ret ind-/udgange, step-tekster og `#Trans[n]`.
5. `FB_Sekvens16` og FC'erne røres ikke – læg dem gerne i et globalt bibliotek som typer.
