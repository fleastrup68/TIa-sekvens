# Funktionsbeskrivelse – Sekvensskabelon 16 step

**Platform:** Siemens TIA Portal V21, S7-1500 / S7-1200
**Sprog:** SCL
**Version:** 1.0 – 2026-11-22

---

## 1. Formål

Skabelonen er en standard til at lave styringssekvenser (trinstyringer) med op til 16 step.
Sekvensens logik (stepskift, timer, log, alarmer) ligger i én fast blok, som genbruges uændret.
Selve procesdelen, dvs. hvad der skal ske i hvert step og hvornår der skal skiftes, skrives i en
kopi af skabelonblokken. Alle sekvenser i et anlæg kommer derfor til at opføre sig ens og er lette
at fejlfinde.

## 2. Opbygning

```
OB1 (Main)
 └─ "MinSekvens_DB"  →  FB_MinSekvens      (skabelon – én kopi pr. sekvens)
                         ├─ Step-regioner   (tekst, timeout, skiftebetingelse pr. step)
                         ├─ Seq : FB_Sekvens16   (sekvensmotor – multi-instans)
                         │    └─ Log[0..49] : typ_SekvensLog
                         └─ Udgange         (step bits → ventiler/motorer)
```

| Blok | Type | Beskrivelse |
|---|---|---|
| `typ_SekvensLog` | PLC-datatype | Én linje i step-loggen |
| `FB_Sekvens16` | FB | Sekvensmotoren. Rettes ikke. |
| `FB_MinSekvens` | FB | Skabelon. Kopiér og ret for hver sekvens. |
| `MinSekvens_DB` | Instans-DB | Data for sekvensen (status, log, konfiguration) |

## 3. Sekvensens forløb

```
          Start (flanke) + Trans[0]
 [Step 0] ─────────────────────────► [Step 1] ──Trans[1]──► [Step 2] ──Trans[2]──► ... [Step n]
  Hvile                                                                                 │
    ▲                                                                       Trans[n] + NextStep=99
    │                                                                       eller step 16 færdig
    └──────────────── Sekvens slut (o_Done) ◄──────────────────────────────────────────┘
                      (ved AutoRestart → direkte til step 1)
```

- **Step 0 – Hvile:** sekvensen er stoppet og alle step bits for step 1–16 er FALSE.
- **Start:** en positiv flanke på `i_Start` starter sekvensen i step 1, hvis startbetingelserne
  `Trans[0]` er opfyldt.
- **Stepskift:** når skiftebetingelsen `Trans[n]` for det aktive step er TRUE, skiftes til næste step.
  Kun det aktive steps betingelse bliver evalueret.
- **Næste step:** normalt n+1. Det kan ændres pr. step med `Config.NextStep[n]`:

  | Værdi | Betydning |
  |---|---|
  | 0 | Næste step (n+1) – standard |
  | 1..16 | Spring til dette step (forgrening/løkke) |
  | 99 | Sekvens slut |

- **Slut:** efter step 16, eller når et step har `NextStep = 99`, er sekvensen gennemført.
  `o_Done` giver en puls, og cyklustælleren tælles op. Derefter går sekvensen til step 0, eller til
  step 1 hvis `i_AutoRestart` er TRUE og startbetingelserne er opfyldt.

## 4. Driftsformer

| Funktion | Indgang | Virkning |
|---|---|---|
| Start | `i_Start` (flanke) | Starter fra step 0. Fortsætter efter genstart-pause. |
| Pause | `i_Hold` | Ingen stepskift. Steptimeren står stille. Step bits forbliver aktive. `o_Hold` = TRUE. |
| Reset | `i_Reset` | Går altid til step 0 og kvitterer alarm. Har højeste prioritet. |
| Step-for-step | `i_StepMode` + `i_StepConfirm` | Skift kræver både en opfyldt betingelse og en flanke på bekræft. `o_WaitConfirm` viser, at sekvensen venter. Til idriftsætning. |
| Spring | `i_JumpTo` (1..16) | Springer straks til det angivne step. Har prioritet over transitionen. 0 = intet spring. |
| Gentag | `i_AutoRestart` | Sekvensen kører forfra uden stop i step 0. |

**Prioritet:** Reset → Pause → Start/Spring → Transition.

## 5. Steptimer

- `o_StepTime` (TIME) og `o_StepTime_s` (REAL, sekunder) angiver tiden i det aktive step.
- Timeren nulstilles ved hvert stepskift og står stille under pause.
- Den måles med `RUNTIME`, så den er uafhængig af cyklustid og OB.
- Timeren går op til ca. 23 døgn.
- Brug i en skiftebetingelse:
  `#Trans[4] := #Seq.o_StepTime >= T#10M;`

## 6. Step bits

- `o_StepBit[0..16]`: præcis én bit er TRUE, nemlig den for det aktive step.
- `o_StepEntry` er TRUE i én cyklus, når et nyt step aktiveres. Den bruges fx til at nulstille
  tællere eller sætte setpunkter.
- Princip: hver aktuator har **én** linje i `REGION Udgange`, hvor de step, den skal være aktiv i,
  er OR'et sammen:
  `#o_M1_Omroerer := #tempUdgangOK AND (#Seq.o_StepBit[2] OR #Seq.o_StepBit[3]);`
- Derved kan en aktuator ikke blive styret fra flere steder i programmet.

## 7. Timeout-overvågning

- `Config.MaxTime[n]` angiver den maksimale tid i step n. `T#0MS` betyder ingen overvågning.
- Ved overskridelse sættes `o_TimeoutAlarm`, og `o_TimeoutStep` viser hvilket step.
- Alarmen gives én gang pr. step og kvitteres med `i_AlarmAck` eller `i_Reset`.
- Sekvensen stoppes **ikke** automatisk. Skal den stoppe ved timeout, sættes `o_TimeoutAlarm` på
  `i_Hold` i den kaldende blok.

## 8. Log over de sidste 50 stepskift

Loggen ligger i `Seq.Log[0..49]`. `[0]` er det nyeste skift, og det ældste forsvinder, når der kommer et nyt.

| Felt | Type | Indhold |
|---|---|---|
| `Nr` | DInt | Løbenummer (stiger altid) |
| `FraStep` | Int | Step der blev forladt |
| `TilStep` | Int | Step der blev aktiveret |
| `Aarsag` | Int | 1 = Transition, 2 = Spring, 3 = Start, 4 = Reset, 5 = Sekvens slut |
| `TidIStep` | Time | Hvor længe `FraStep` var aktivt |
| `Tidsstempel` | DTL | Lokal PLC-tid for skiftet |

Loggen kan vises direkte på HMI som en tabel. `TidIStep` gør det let at se, hvilke step der tager
længere tid end normalt.

## 9. Retain og spændingsgenkomst

| Data | Retain |
|---|---|
| Aktivt step | Ja |
| Steptimer | Ja |
| Cyklustæller | Ja |
| Log-løbenummer | Ja |
| Log, alarmer, konfiguration | Nej |

**Ved spændingsgenkomst (eller STOP→RUN) midt i en sekvens:**
1. Sekvensen står i det step, den var i før.
2. `o_RestartHold` = TRUE, og sekvensen står i pause.
3. Skabelonen holder **alle aktuatorer slukket**, så intet starter af sig selv.
4. Operatøren vælger:
   - **Start** → fortsæt i samme step (steptimeren fortsætter fra den gemte værdi)
   - **Reset** → afbryd og gå til step 0

## 10. Interface – FB_Sekvens16

**Indgange**

| Navn | Type | Beskrivelse |
|---|---|---|
| `i_Start` | Bool | Start / fortsæt (flanke) |
| `i_Hold` | Bool | Pause |
| `i_Reset` | Bool | Reset til step 0 |
| `i_StepMode` | Bool | Step-for-step |
| `i_StepConfirm` | Bool | Bekræft stepskift (flanke) |
| `i_AutoRestart` | Bool | Gentag sekvens |
| `i_AlarmAck` | Bool | Kvittér timeout |
| `i_JumpTo` | Int | Spring til step (0 = intet) |
| `i_Trans` | Array[0..16] of Bool | Skiftebetingelser |

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
| `o_RestartHold` | Bool | Pause efter spændingsgenkomst |
| `o_WaitConfirm` | Bool | Venter på bekræft (step-mode) |
| `o_Done` | Bool | Puls: sekvens gennemført |
| `o_CycleCount` | DInt | Antal gennemførte sekvenser |
| `o_TimeoutAlarm` | Bool | Timeout i step |
| `o_TimeoutStep` | Int | Step med timeout |

**Konfiguration (`Seq.Config`)**

| Navn | Type | Beskrivelse |
|---|---|---|
| `NextStep[1..16]` | Int | 0 = n+1, 1..16 = spring, 99 = slut |
| `MaxTime[1..16]` | Time | Max tid pr. step (T#0MS = fra) |
| `StepText[0..16]` | String[40] | Beskrivelse pr. step |

## 11. Eksempelsekvens i skabelonen

| Step | Tekst | Aktuatorer | Skiftebetingelse | Timeout |
|---|---|---|---|---|
| 0 | Hvile | – | Ingen fejl og tank tom | – |
| 1 | Fyld tank | V1 indløb | Niveau høj | 2 min |
| 2 | Start omrører | M1 | M1 kører i 2 s | 10 s |
| 3 | Opvarm | M1, varme | Temperatur OK | 30 min |
| 4 | Omrør 10 min | M1 | Steptimer ≥ 10 min | – |
| 5 | Tøm tank | V2 udløb | Niveau lav + 5 s | 3 min |

Step 5 har `NextStep = 99`, så sekvensen slutter efter step 5. Fejl giver pause, og varmen slukkes
under pause.

## 12. Ny sekvens – arbejdsgang

1. Kopiér `FB_MinSekvens` med et nyt navn (fx `FB_Tank2_Sekvens`) og opret en ny instans-DB.
2. Ret ind- og udgange til anlæggets signaler.
3. Udfyld for hvert step: `StepText`, `MaxTime`, eventuelt `NextStep` og `Trans[n]`.
4. Sæt `NextStep = 99` på det sidste step, der bruges.
5. Tilknyt step bits til aktuatorerne i `REGION Udgange`.
6. Kald instans-DB'en i OB1 og test med step-for-step (`i_StepMode`).
