# Funktionsbeskrivelse – Sekvensskabelon 16 step

**Platform:** Siemens TIA Portal V21, S7-1500 / S7-1200, WinCC Unified PC
**Sprog:** SCL
**Version:** 2.2 – 2026-11-26

---

## 1. Formål

Skabelonen er en standard til at lave styringssekvenser (trinstyringer) med op til 16 step.
Sekvensens logik (stepskift, timer, log, alarm og HMI) ligger i én fast blok, som genbruges uændret.
Selve procesdelen, dvs. hvad der skal ske i hvert step og hvornår der skal skiftes, skrives i en
kopi af skabelonblokken. Alle sekvenser i et anlæg kommer derfor til at opføre sig ens og er lette
at fejlfinde.

## 2. Opbygning

```
OB1 (Main)
 └─ "MinSekvens_DB"  →  FB_MinSekvens            (skabelon – én kopi pr. sekvens)
                         ├─ Step-regioner         (tekst, timeout, skiftebetingelse pr. step)
                         ├─ Seq : FB_Sekvens16    (sekvensmotor – multi-instans)
                         │    ├─ Program_Alarm    (timeout-alarm til WinCC)
                         │    └─ FC_SekvensLogTekst (loglinjer som tekst)
                         └─ Udgange               (step bits → ventiler/motorer)

 io_Data ◄──► "Sekvens_Data".MinSekvens       (RETAIN: step, steptimer, tællere)
 io_Log  ◄──► "Sekvens_Data".MinSekvens_Log   (RETAIN: log over 50 stepskift)
 io_HMI  ◄──► "Sekvens_HMI".MinSekvens        (HMI: kommandoer og status)
```

| Blok | Type | Beskrivelse |
|---|---|---|
| `typ_SekvensLog` | PLC-datatype | Én linje i step-loggen |
| `typ_SekvensData` | PLC-datatype | Step-nr, steptimer og tællere |
| `typ_SekvensHMI` | PLC-datatype | HMI-interface: kommandoer, status, oversigt og log |
| `FC_SekvensTal` | FC | Hjælpefunktion: tal til tekst med foranstillede nuller |
| `FC_SekvensLogTekst` | FC | Hjælpefunktion: én log-post som tekstlinje |
| `FB_Sekvens16` | FB | Sekvensmotoren. Rettes ikke. |
| `FB_MinSekvens` | FB | Skabelon. Kopiér og ret for hver sekvens. |
| `Sekvens_Data` | Global DB (RETAIN) | Step og log for alle sekvenser |
| `Sekvens_HMI` | Global DB | HMI-interface for alle sekvenser |
| `MinSekvens_DB` | Instans-DB | Instans af skabelonen |

**Hvorfor InOut?** Step-nr, steptimer og log ligger i den globale DB `Sekvens_Data` og ikke i
instans-DB'en. Når der tilføjes ind- eller udgange i skabelonen og instans-DB'en derfor
reinitialiseres ved download, bevares sekvensens step og log.

## 3. Sekvensens forløb

```
          Start (flanke) + Trans[0]
 [Step 0] ─────────────────────────► [Step 1] ──Trans[1]──► [Step 2] ──Trans[2]──► ... [Step n]
  Hvile                                                                                 │
    ▲                                                                       Trans[n] + Naeste=99
    │                                                                       eller step 16 færdig
    └──────────────── Sekvens slut (o_Done) ◄──────────────────────────────────────────┘
                      (ved AutoRestart → direkte til step 1)
```

- **Step 0 – Hvile:** sekvensen er stoppet og alle step bits for step 1–16 er FALSE.
- **Start:** en positiv flanke på Start starter sekvensen i step 1, hvis startbetingelserne
  `Trans[0]` er opfyldt.
- **Stepskift:** når skiftebetingelsen `Trans[n]` for det aktive step er TRUE, skiftes til næste step.
  Kun det aktive steps betingelse bliver brugt.
- **Næste step:** normalt n+1. Det kan ændres pr. step med `#Naeste[n]`:

  | Værdi | Betydning |
  |---|---|
  | 0 (udeladt) | Næste step (n+1) – standard |
  | 1..16 | Spring til dette step (forgrening/løkke) |
  | 99 | Sekvens slut |

- **Slut:** efter step 16, eller når et step har `Naeste = 99`, er sekvensen gennemført.
  `o_Done` giver en puls, og cyklustælleren tælles op. Derefter går sekvensen til step 0, eller til
  step 1 hvis auto-genstart er valgt og startbetingelserne er opfyldt.
- **Ubrugte step:** står i skabelonen med tom tekst (`#Tekst[n] := ''`). De nås aldrig, skjules på
  HMI og kan ikke tvinges til fra HMI.

## 4. Sådan skrives et step

Hvert step har sin egen `REGION` i skabelonen. Overskriften skrives med eget navn, så blokken
foldet sammen læses som en liste over sekvensen.

```
REGION Step 02 - Start omrører
    #Tekst[2]  := 'Start omrører';                    // tekst til HMI / log / alarm
    #MaxTid[2] := T#10S;                              // timeout (udelad = ingen)
    #Trans[2]  := #i_M1_Koerer AND #StepTid >= T#2S;  // skiftebetingelse
END_REGION
```

| Navn | Betydning |
|---|---|
| `#Tekst[n]` | Beskrivelse af step n (maks 40 tegn). Tom = step ikke i brug |
| `#MaxTid[n]` | Max tid i step n før alarm. Udeladt = ingen overvågning |
| `#Naeste[n]` | Kun ved spring eller slut (99) |
| `#Trans[n]` | Betingelse for at forlade step n. `#Trans[0]` = startbetingelser |
| `#StepTid` | Tid i aktivt step (TIME) |
| `#S[n]` | Step bit: TRUE når step n er aktivt |
| `#JumpTo` | Spring direkte til et step fra programmet (fx ved fejl) |

Konfigurationen nulstilles i starten af hver cyklus. Kun det, der står i step-regionerne, gælder.

## 5. Driftsformer

Alle funktioner kan styres både fra programmet (indgange) og fra HMI. De virker side om side (OR).

| Funktion | Indgang | HMI | Virkning |
|---|---|---|---|
| Start | `i_Start` (flanke) | `Cmd_Start` | Starter fra step 0. Fortsætter efter genstart-pause. |
| Pause | `i_Hold` | `Sel_Hold` | Ingen stepskift. Steptimeren står stille. Step bits forbliver aktive. |
| Reset | `i_Reset` | `Cmd_Reset` | Går altid til step 0 og kvitterer alarm. Højeste prioritet. |
| Step-for-step | `i_StepMode` + `i_StepConfirm` | `Sel_StepMode` + `Cmd_StepConfirm` | Skift kræver både opfyldt betingelse og bekræft. Til idriftsætning. |
| Spring | `i_JumpTo` (1..16) | – | Springer straks til step. Prioritet over transition. Ikke under pause. |
| Tvang | `i_ForceStep` + `i_Force` | `ForceStep` + `Cmd_Force` | Sætter sekvensen direkte i et step 0..16 – også fra step 0 og under pause. |
| Gentag | `i_AutoRestart` | `Sel_AutoRestart` | Sekvensen kører forfra uden stop i step 0. |
| Kvittér | `i_AlarmAck` | `Cmd_AlarmAck` | Kvitterer timeout-alarm |

**Prioritet:** Reset → Tvang → Pause → Start / Spring → Transition.

**Tvang fra HMI** er kun tilladt når sekvensen er i pause, step-mode, genstart-pause eller step 0
(`ForceTilladt`), og kun til et step der er i brug. Tvang fra programmet har ingen begrænsning.
Tvang ignorerer skiftebetingelser, og aktuatorerne for det nye step aktiveres med det samme,
medmindre pause er aktiv.

## 6. Steptimer

- `#StepTid` / `o_StepTime` (TIME) og `o_StepTime_s` (REAL, sekunder) angiver tiden i det aktive step.
- Timeren nulstilles ved hvert stepskift og står stille under pause.
- Den måles med `RUNTIME`, så den er uafhængig af cyklustid og OB.
- Timeren går op til ca. 23 døgn og er RETAIN.
- Brug i en skiftebetingelse: `#Trans[4] := #StepTid >= T#10M;`

## 7. Step bits

- `#S[0..16]` (= `o_StepBit`): præcis én bit er TRUE, nemlig den for det aktive step.
- `#Seq.o_StepEntry` er TRUE i én cyklus, når et nyt step aktiveres. Den bruges fx til at nulstille
  tællere eller sætte setpunkter.
- Princip: hver aktuator har **én** linje i `REGION Udgange`, hvor de step, den skal være aktiv i,
  er OR'et sammen:
  `#o_M1_Omroerer := #tempUdgangOK AND (#S[2] OR #S[3] OR #S[4]);`
- Derved kan en aktuator ikke blive styret fra flere steder i programmet.

## 8. Timeout-overvågning og alarm

- `#MaxTid[n]` angiver den maksimale tid i step n. Udeladt eller `T#0MS` betyder ingen overvågning.
- Ved overskridelse sættes `o_TimeoutAlarm`, og `o_TimeoutStep` viser hvilket step.
- Alarmen gives én gang pr. step og kvitteres med Kvittér eller Reset.
- Sekvensen stoppes **ikke** automatisk. Skal den stoppe ved timeout, sættes `o_TimeoutAlarm` på
  pause i den kaldende blok.

**Program_Alarm:** Sekvensmotoren indeholder en `Program_Alarm`, så timeout vises i WinCC Unified's
alarmliste og alarmhistorik uden opsætning af HMI-alarmer.

| Ledsageværdi | Indhold |
|---|---|
| `@1%s@` | Prefix fra indgangen `i_Prefix`, fx `SPVA12` |
| `@2%d@` | Step med timeout |
| `@3%s@` | Steptekst |

Alarmteksten sættes **én gang** i `FB_Sekvens16` (PLC supervisions & alarms):
`@1%s@ Timeout i step @2%d@: @3%s@`, hvilket giver fx **"SPVA12 Timeout i step 3: Opvarm"**.
Alle sekvenser arver teksten og adskilles af deres prefix.

Kræver S7-1500 eller S7-1200 med firmware V4.2 eller nyere.

## 9. Log over de sidste 50 stepskift

Loggen ligger i `Sekvens_Data.MinSekvens_Log[0..49]`. `[0]` er det nyeste skift, og det ældste
forsvinder, når der kommer et nyt.

| Felt | Type | Indhold |
|---|---|---|
| `Nr` | DInt | Løbenummer (stiger altid) |
| `FraStep` | Int | Step der blev forladt |
| `FraTekst` | String[40] | Tekst for det forladte step |
| `TilStep` | Int | Step der blev aktiveret |
| `Aarsag` | Int | 1 = Transition, 2 = Spring, 3 = Start, 4 = Reset, 5 = Sekvens slut, 6 = Tvang |
| `TidIStep` | Time | Hvor længe `FraStep` var aktivt |
| `Tidsstempel` | DTL | Lokal PLC-tid for skiftet |

Til HMI laves loggen også som færdige tekstlinjer i `Sekvens_HMI.MinSekvens.LogTekst[0..49]`:

```
#0012 26.11 14:03:21 S03>S04 00:05:12 Trans  Opvarm
Nr    Dato  Klokken  Fra>Til TidIStep Årsag  Tekst for forladt step
```

Loggen kan slettes fra HMI (`Cmd_SletLog`).

## 10. Retain og spændingsgenkomst

| Data | Hvor | Retain |
|---|---|---|
| Aktivt step | `Sekvens_Data` | Ja |
| Steptimer | `Sekvens_Data` | Ja |
| Cyklustæller, log-løbenummer | `Sekvens_Data` | Ja |
| Log (50 linjer, ca. 4 kB) | `Sekvens_Data` | Ja (kan slås fra) |
| HMI-status og logtekster | `Sekvens_HMI` | Nej – genopbygges automatisk |
| Alarm, flanker | Instans-DB | Nej |

**Ved spændingsgenkomst, STOP→RUN eller reinit af instans-DB'en midt i en sekvens:**
1. Sekvensen står i det step, den var i før.
2. `o_RestartHold` = TRUE, og sekvensen står i pause.
3. Skabelonen holder **alle aktuatorer slukket**, så intet starter af sig selv.
4. Operatøren vælger:
   - **Start** → fortsæt i samme step (steptimeren fortsætter fra den gemte værdi)
   - **Reset** → afbryd og gå til step 0
   - **Tvang** → fortsæt i et valgt step

## 11. HMI-interface (`typ_SekvensHMI`)

**Kommandoer HMI → PLC.** `Cmd_xxx` er pulser: HMI sætter bit (SetBitInTag), PLC nulstiller den.

| Tag | Type | Funktion |
|---|---|---|
| `Cmd_Start` | Bool | Start / fortsæt |
| `Cmd_Reset` | Bool | Reset til step 0 |
| `Cmd_StepConfirm` | Bool | Næste step i step-mode |
| `Cmd_AlarmAck` | Bool | Kvittér timeout |
| `Cmd_Force` | Bool | Tving til `ForceStep` |
| `Cmd_SletLog` | Bool | Slet loggen |
| `ForceStep` | Int | Step der skal tvinges til (0..16) |
| `Sel_Hold` | Bool | Kontakt: pause |
| `Sel_StepMode` | Bool | Kontakt: step-for-step |
| `Sel_AutoRestart` | Bool | Kontakt: auto-genstart |

**Status PLC → HMI**

| Tag | Type | Indhold |
|---|---|---|
| `Prefix` | String[20] | Sekvensens prefix |
| `Step` / `StepTekst` | Int / String | Aktivt step og beskrivelse |
| `StepTid_s` | Real | Tid i aktivt step [s] |
| `MaxTid_s` / `RestTid_s` | Real | Max tid og tid til timeout [s] (0 = ingen) |
| `TransOK` | Bool | Skiftebetingelsen for aktivt step er opfyldt |
| `Running`, `Hold`, `RestartHold`, `WaitConfirm` | Bool | Tilstand |
| `TimeoutAlarm` / `TimeoutStep` | Bool / Int | Alarm og hvilket step |
| `ForceTilladt` | Bool | Tvang fra HMI er tilladt nu |
| `AntalCyklus` | DInt | Gennemførte sekvenser |
| `StepAktiv[0..16]` | Bool | Step bits |
| `StepBrugt[0..16]` | Bool | Step i brug (til synlighed) |
| `Tekst[0..16]` | String | Alle steptekster |
| `MaxTidListe_s[0..16]` | Real | Max tid for alle step |
| `SidsteTid_s[0..16]` | Real | Seneste varighed af hvert step |
| `LogTekst[0..49]` | String[100] | Log som tekstlinjer |

Til WinCC Unified PC findes en trin-for-trin vejledning til en genbrugelig faceplate i
`HMI_Unified_Faceplate.md`.

## 12. Interface – FB_Sekvens16

**Indgange**

| Navn | Type | Beskrivelse |
|---|---|---|
| `i_Prefix` | String[20] | Prefix i alarmtekst og på HMI |
| `i_Start` | Bool | Start / fortsæt (flanke) |
| `i_Hold` | Bool | Pause |
| `i_Reset` | Bool | Reset til step 0 |
| `i_StepMode` | Bool | Step-for-step |
| `i_StepConfirm` | Bool | Bekræft stepskift (flanke) |
| `i_AutoRestart` | Bool | Gentag sekvens |
| `i_AlarmAck` | Bool | Kvittér timeout |
| `i_JumpTo` | Int | Spring til step (0 = intet) |
| `i_ForceStep` | Int | Tvang: step 0..16 |
| `i_Force` | Bool | Tvang: udfør (flanke) |
| `i_Trans` | Array[0..16] of Bool | Skiftebetingelser |
| `i_Naeste` | Array[1..16] of Int | Næste step: 0 = n+1, 1..16 = spring, 99 = slut |
| `i_MaxTid` | Array[1..16] of Time | Max tid pr. step (T#0MS = fra) |
| `i_Tekst` | Array[0..16] of String[40] | Beskrivelse pr. step |

**Udgange**

| Navn | Type | Beskrivelse |
|---|---|---|
| `o_Step` | Int | Aktivt step |
| `o_StepBit` | Array[0..16] of Bool | Step bits |
| `o_StepEntry` | Bool | Puls ved nyt step |
| `o_StepTime` | Time | Steptimer |
| `o_StepTime_s` | Real | Steptimer i sekunder |
| `o_StepText` | String[40] | Tekst for aktivt step |
| `o_Running` | Bool | Sekvens kører |
| `o_Hold` | Bool | Pause aktiv |
| `o_RestartHold` | Bool | Pause efter spændingsgenkomst / reinit |
| `o_WaitConfirm` | Bool | Venter på bekræft (step-mode) |
| `o_Done` | Bool | Puls: sekvens gennemført |
| `o_CycleCount` | DInt | Antal gennemførte sekvenser |
| `o_TimeoutAlarm` | Bool | Timeout i step |
| `o_TimeoutStep` | Int | Step med timeout |

**InOut**

| Navn | Type | Kobles til |
|---|---|---|
| `io_Data` | `typ_SekvensData` | `Sekvens_Data.<navn>` |
| `io_Log` | Array[0..49] of `typ_SekvensLog` | `Sekvens_Data.<navn>_Log` |
| `io_HMI` | `typ_SekvensHMI` | `Sekvens_HMI.<navn>` |

## 13. Eksempelsekvens i skabelonen

| Step | Tekst | Aktuatorer | Skiftebetingelse | Timeout |
|---|---|---|---|---|
| 0 | Hvile | – | Ingen fejl og tank tom | – |
| 1 | Fyld tank | V1 indløb | Niveau høj | 2 min |
| 2 | Start omrører | M1 | M1 kører i 2 s | 10 s |
| 3 | Opvarm | M1, varme | Temperatur OK | 30 min |
| 4 | Omrør 10 min | M1 | Steptimer ≥ 10 min | – |
| 5 | Tøm tank | V2 udløb | Niveau lav + 5 s | 3 min |

Step 5 har `Naeste = 99`, så sekvensen slutter efter step 5. Step 6–16 er ikke i brug.
Fejl giver pause, og varmen slukkes under pause.

## 14. Import og ny sekvens – arbejdsgang

**Første gang:**
1. Importér kildefilerne 01 → 09 som *External source files* og kør *Generate blocks from source*
   i rækkefølge.
2. Sæt alarmteksten på `statPA_Timeout` i `FB_Sekvens16`.
3. Kontrollér at begge linjer i `Sekvens_Data` er Retain.

**For hver ny sekvens:**
1. Kopiér `FB_MinSekvens` med et nyt navn (fx `FB_Tank2_Sekvens`) og opret en ny instans-DB.
2. Tilføj `Tank2 : "typ_SekvensData"` og `Tank2_Log : Array[0..49] of "typ_SekvensLog"` (Retain)
   i `Sekvens_Data`, og `Tank2 : "typ_SekvensHMI"` i `Sekvens_HMI`.
3. Ret ind- og udgange til anlæggets signaler.
4. Udfyld for hvert step: `#Tekst`, evt. `#MaxTid`, evt. `#Naeste` og `#Trans[n]`.
5. Sæt `#Naeste[sidste brugte step] := 99`.
6. Tilknyt step bits `#S[n]` til aktuatorerne i `REGION Udgange`.
7. Kald instans-DB'en i OB1 med `i_Prefix`, `io_Data`, `io_Log` og `io_HMI`.
8. Indsæt en ny instans af faceplaten `fp_Sekvens16` på HMI og kobl den til `Sekvens_HMI.Tank2`.
9. Test med step-for-step.
