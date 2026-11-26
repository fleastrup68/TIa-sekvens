# Sekvensskabelon 16 step – TIA Portal V21 (SCL)

## Filer

| Fil | Indhold |
|---|---|
| `01_typ_SekvensLog.udt` | Datatype for én linje i step-loggen |
| `02_FB_Sekvens16.scl` | Sekvensmotoren – genbruges uændret i alle sekvenser |
| `03_FB_MinSekvens.scl` | **Skabelonen** – kopiér og ret for hver ny sekvens |
| `04_MinSekvens_DB.db` | Instans-DB til skabelonen |

## Import i TIA Portal

1. Projekttræ → PLC → **External source files** → *Add new external file* → vælg alle 4 filer.
2. Højreklik på filerne **i rækkefølge 01 → 04** → *Generate blocks from source*.
3. Kald i OB1 (Main):
   ```
   "MinSekvens_DB"(i_Start := "Start_PB", i_Reset := "Reset_PB", ...);
   ```
4. Kontrollér i interfacet på `FB_MinSekvens`, at multi-instansen `Seq` har Retain = **"Set in IDB"**,
   så RETAIN-indstillingerne inde i `FB_Sekvens16` bruges.

Virker på S7-1500 og S7-1200 (bruger `RUNTIME` og `RD_LOC_T`).

## Sådan bruges skabelonen

Hvert step har sin egen `REGION` i `FB_MinSekvens`:

```
REGION Step 02 - Start omrører
    #Seq.Config.StepText[2] := 'Start omrører';     // tekst til HMI / log
    #Seq.Config.MaxTime[2]  := T#10S;               // timeout-alarm (T#0MS = fra)
    #Trans[2] := #i_M1_Koerer AND #Seq.o_StepTime >= T#2S;   // skiftebetingelse
END_REGION
```

- **Skiftebetingelse:** `#Trans[n]` = betingelse for at *forlade* step n. `#Trans[0]` = startbetingelser.
- **Steptimer:** `#Seq.o_StepTime` (TIME) eller `#Seq.o_StepTime_s` (REAL). Nulstilles ved hvert stepskift, står stille under pause.
- **Step bits:** `#Seq.o_StepBit[n]` – brug dem i `REGION Udgange` til ventiler og motorblokke. `#Seq.o_StepEntry` er en 1-cyklus puls ved indgang i nyt step.
- **Næste step:** `Config.NextStep[n]`: 0 = n+1 (standard), 1..16 = spring, **99 = sekvens slut**. Brugte steps kan altså være færre end 16.
- **Spring:** `#JumpTo := 7;` springer direkte til step 7 (prioritet over transition).

## Funktioner i FB_Sekvens16

| Indgang | Funktion |
|---|---|
| `i_Start` | Flanke: start fra step 0 / fortsæt efter genstart-pause |
| `i_Hold` | Pause – ingen stepskift, steptimer stopper |
| `i_Reset` | Til step 0, kvitterer alarm |
| `i_StepMode` + `i_StepConfirm` | Step-for-step ved idriftsætning |
| `i_AutoRestart` | Efter sidste step → step 1 i stedet for 0 |
| `i_AlarmAck` | Kvittér timeout |

### RETAIN og spændingsgenkomst
Step-nummer, steptimer, cyklustæller og log-løbenummer er **RETAIN**. Efter spændingsgenkomst
står sekvensen i det gemte step med `o_RestartHold = TRUE` (pause). Skabelonen holder alle
aktuatorer slukket indtil operatøren giver **Start** (fortsæt i samme step) eller **Reset** (step 0).

### Log (sidste 50 stepskift)
`MinSekvens_DB.Seq.Log[0..49]` – `[0]` er nyeste. Hver linje: `Nr`, `FraStep`, `TilStep`,
`Aarsag`, `TidIStep`, `Tidsstempel` (lokal tid – husk at sætte PLC'ens tidszone/ur).

| Aarsag | Betydning |
|---|---|
| 1 | Transition |
| 2 | Spring (JumpTo) |
| 3 | Start |
| 4 | Reset |
| 5 | Sekvens slut |

Loggen er ikke RETAIN (sparer retain-hukommelse). Ønskes det, flyt `Log` op i `VAR RETAIN`-sektionen i `FB_Sekvens16` (~1,3 kB pr. sekvens).

## Ny sekvens
1. Kopiér `FB_MinSekvens` → fx `FB_Tank2_Sekvens`, og lav en ny instans-DB.
2. Ret ind-/udgange, step-tekster og `#Trans[n]`.
3. `FB_Sekvens16` røres ikke – læg den gerne i et globalt bibliotek som type.
