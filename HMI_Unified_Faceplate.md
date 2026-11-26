# HMI – WinCC Unified PC: faceplate til FB_Sekvens16

Mål: **én faceplate** (`fp_Sekvens16`) der kan genbruges til alle sekvenser. Hver sekvens
får sin egen instans af faceplaten, koblet til `Sekvens_HMI.<sekvensnavn>`.

Faceplaten bygges af to typer i projektbiblioteket:

| Type | Indhold |
|---|---|
| `fp_SekvensStep` | Én linje i stepoversigten (lampe, nr, tekst, max tid, seneste tid) |
| `fp_Sekvens16` | Hele sekvensbilledet – indeholder 17 × `fp_SekvensStep` |

---

## 1. HMI-tag til sekvensen

1. Generer PLC-blokkene og oversæt PLC'en (så `typ_SekvensHMI` og `Sekvens_HMI` kendes).
2. Unified PC → **HMI tags** → ny tagtabel `Sekvenser`.
3. Træk `Sekvens_HMI.MinSekvens` fra PLC'en ind i tagtabellen. Der oprettes ét struktureret
   HMI-tag af typen `typ_SekvensHMI`, fx `Sekvens_HMI_MinSekvens`.
4. Acquisition cycle: **1 s** er rigeligt. (Knapper skriver uanset cyklus med det samme.)

> Kræver at `Sekvens_HMI` har *Accessible from HMI/OPC UA* (standard ved import).

---

## 2. Faceplate `fp_SekvensStep` (én linje)

Projektbibliotek → **Types** → *Add new type* → **Faceplate** (HMI Unified). Størrelse ca. 460 × 26 px.

### Interface (Tags)

| Navn | Datatype |
|---|---|
| `Aktiv` | Bool |
| `Tekst` | WString |
| `MaxTid_s` | Real |
| `SidsteTid_s` | Real |

### Interface (Properties)

| Navn | Datatype | Bruges til |
|---|---|---|
| `Nr` | Int | Stepnummer der vises (sættes fast på hver instans: 0..16) |

### Elementer

| Element | Egenskab | Dynamisering |
|---|---|---|
| Cirkel (Ø16) | Background color | Tag `Aktiv`: FALSE = grå, TRUE = grøn |
| Tekst "Nr" | Text | Property `Nr`, format 2 cifre |
| Tekst (bred) | Text | Tag `Tekst` |
| I/O-felt (output) | Process value | Tag `MaxTid_s`, format 0 decimaler, enhed "s". Visible: `MaxTid_s > 0` |
| I/O-felt (output) | Process value | Tag `SidsteTid_s`, format 1 decimal, enhed "s" |
| Rektangel (baggrund) | Background color | Tag `Aktiv`: TRUE = lysegrøn markering |

Release-versionér typen (*Release version*).

---

## 3. Faceplate `fp_Sekvens16` (hele billedet)

Ny faceplate-type, ca. 1200 × 700 px.

### Interface

| Art | Navn | Datatype |
|---|---|---|
| Tag | `Seq` | **`typ_SekvensHMI`** (PLC-datatypen – vælg den under *PLC data types*) |
| Property | `Titel` | WString |

### Layout

```
┌──────────────────────────────────────────────────────────────────────────┐
│ {Titel}                           [KØRER] [PAUSE] [GENSTART] [ALARM S03] │  Statuslinje
├──────────────────────────────────────────────────────────────────────────┤
│ Step 03  Opvarm                                   Skift OK ●             │  Aktivt step
│ Tid i step 125.4 s   Max 1800 s   Rest 1674.6 s   [██████░░░░░░]         │
│ Cyklusser 42                                                             │
├────────────────────────────────────┬─────────────────────────────────────┤
│ STEPOVERSIGT  (17 × fp_SekvensStep)│ BETJENING                           │
│                                    │ [START] [RESET] [KVITTÉR]           │
│                                    │ [■] Pause  [■] Step-mode  [■] Auto  │
│                                    │ [NÆSTE STEP]                        │
│                                    │ Tving til: [ 05 ]  [TVING]          │
├────────────────────────────────────┴─────────────────────────────────────┤
│ LOG (nyeste øverst)                                          [SLET LOG]  │
│ (flerlinjet tekstboks)                                                   │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3a. Statuslinje

| Element | Dynamisering |
|---|---|
| Tekst `{Titel}` | Property `Titel` |
| Tekst prefix | `Seq.Prefix` (fx "SPVA12" foran titlen) |
| Lampe "KØRER" | `Seq.Running` → grøn |
| Lampe "PAUSE" | `Seq.Hold` → gul |
| Lampe "GENSTART" | `Seq.RestartHold` → orange, blink |
| Lampe "VENTER" | `Seq.WaitConfirm` → blå (step-mode venter på Næste step) |
| Lampe "ALARM" + I/O-felt | `Seq.TimeoutAlarm` → rød, blink. Tekst: `Seq.TimeoutStep` |

### 3b. Aktivt step

| Element | Tag | Format |
|---|---|---|
| I/O-felt Step | `Seq.Step` | 2 cifre |
| Tekst | `Seq.StepTekst` | |
| Lampe "Skift OK" | `Seq.TransOK` | grøn = betingelse opfyldt (står den grå længe, venter sekvensen på en betingelse) |
| I/O-felt Tid i step | `Seq.StepTid_s` | 1 decimal, "s" |
| I/O-felt Max | `Seq.MaxTid_s` | 0 decimaler. Visible: `Seq.MaxTid_s > 0` |
| I/O-felt Rest | `Seq.RestTid_s` | 1 decimal. Visible: `Seq.MaxTid_s > 0` |
| Bar | Process value `Seq.StepTid_s`, Max value `Seq.MaxTid_s` | Visible: `Seq.MaxTid_s > 0` |
| I/O-felt Cyklusser | `Seq.AntalCyklus` | |

### 3c. Stepoversigt

Indsæt 17 instanser af `fp_SekvensStep` under hinanden (instans n = step 0..16):

| Interface på instans n | Kobles til |
|---|---|
| `Aktiv` | `Seq.StepAktiv[n]` |
| `Tekst` | `Seq.Tekst[n]` |
| `MaxTid_s` | `Seq.MaxTidListe_s[n]` |
| `SidsteTid_s` | `Seq.SidsteTid_s[n]` |
| Property `Nr` | fast værdi n |
| **Visible** (instansens egenskab) | `Seq.StepBrugt[n]` |

Ubrugte step (tom `#Tekst[n]` i PLC'en) skjules derved automatisk.

> Skal listen lukke hullerne når et step i midten er skjult, brug en **Flex-/Stack-container**
> hvis din V21 har den, ellers lad blot rækkerne stå i fast position.

### 3d. Betjening

Kommandoer er **pulser** – HMI sætter bit, PLC nulstiller selv. Kontakter holdes af HMI.

| Knap | Event *Press* (eller *Click left mouse button*) | Enabled / Visible |
|---|---|---|
| START | System function **SetBitInTag** → `Seq.Cmd_Start` | |
| RESET | **SetBitInTag** → `Seq.Cmd_Reset` | |
| KVITTÉR | **SetBitInTag** → `Seq.Cmd_AlarmAck` | Visible: `Seq.TimeoutAlarm` |
| NÆSTE STEP | **SetBitInTag** → `Seq.Cmd_StepConfirm` | Enabled: `Seq.WaitConfirm` |
| TVING | **SetBitInTag** → `Seq.Cmd_Force` | Enabled: `Seq.ForceTilladt` |
| SLET LOG | **SetBitInTag** → `Seq.Cmd_SletLog` | Evt. kun for brugerrolle "Service" |

| Kontakt (Switch / Toggle) | Tag |
|---|---|
| Pause | `Seq.Sel_Hold` |
| Step-mode | `Seq.Sel_StepMode` |
| Auto-genstart | `Seq.Sel_AutoRestart` |

| I/O-felt (input) | Tag | Grænser |
|---|---|---|
| Tving til step | `Seq.ForceStep` | Min 0, max 16 |

Tvang beskyttes i PLC'en: den udføres kun når `ForceTilladt` (pause, step-mode, genstart-pause
eller step 0) og kun til et step der er i brug. Giv gerne TVING, RESET og SLET LOG en
**brugerrolle** (*Security* → *Authorization* på knappen).

### 3e. Log

PLC'en leverer 50 færdige linjer i `Seq.LogTekst[0..49]` (`[0]` = nyeste):

```
#0012 26.11 14:03:21 S03>S04 00:05:12 Trans  Opvarm
```

**Anbefalet:** én flerlinjet **Text box** (fast skrifttype, fx Consolas 12, så kolonnerne flugter,
read-only, lodret scrollbar). Dynamisér *Text* med et script, trigger = tag `Seq.LogTekst[0]`:

```javascript
export function TextBox_Log_Text_Trigger(item) {
    let s = '';
    for (let i = 0; i < 50; i++) {
        const linje = Tags('Seq.LogTekst[' + i + ']').Read();
        if (linje !== '') {
            s += linje + '\n';
        }
    }
    return s;
}
```

> Tjek tag-stien i script-editoren: træk `Seq.LogTekst[0]` ind fra interfacet, så indsætter
> editoren den korrekte skrivemåde for faceplate-tags i din V21.

**Alternativ uden script:** 50 (eller 20) tekstfelter under hinanden, hvert dynamiseret med
`Seq.LogTekst[n]`. Mere klikarbejde, men ingen scripts.

Release-versionér `fp_Sekvens16`.

---

## 4. Brug i et skærmbillede

1. Træk `fp_Sekvens16` ind på et screen.
2. Interface-tag `Seq` → HMI-tagget `Sekvens_HMI_MinSekvens`.
3. Property `Titel` → fx `"Tank 1 – Opvarmning"`.

Ny sekvens = ny linje i `Sekvens_HMI` (PLC), nyt HMI-tag, ny faceplate-instans. Intet at tegne om.

---

## 5. Test i simulering (PLCSIM Advanced + Unified PC Runtime)

1. START → step 1, lampen i stepoversigten flytter sig, loggen får en linje "Start".
2. Pause → tid i step står stille, lampe PAUSE.
3. Step-mode → når "Skift OK" er grøn, lyser VENTER og NÆSTE STEP bliver aktiv.
4. Tving: mens den kører uden pause er TVING grå; slå Pause til → TVING aktiv.
5. Download PLC med *reinitialize* → GENSTART-lampe, aktuatorer slukket, log bevaret.
6. Step med `#MaxTid` → lad tiden løbe ud → ALARM, KVITTÉR.

---

## 6. Alarmer i Unified's alarmvisning

Timeout-alarmen kommer fra `Program_Alarm` i `FB_Sekvens16` (se LAES_MIG.md, afsnit *Alarm*)
og vises automatisk som fx **"SPVA12 Timeout i step 3: Opvarm"**.

- Læg et **Alarm control** på screenet (eller i faceplaten) – der skal ikke oprettes HMI-alarmer.
- Filtrér evt. på prefix i alarm control'ens filter (tekst starter med "SPVA12"), hvis faceplaten
  kun skal vise sin egen sekvens' alarmer.
