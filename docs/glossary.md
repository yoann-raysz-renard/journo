# Glossary — French domain, English code

journo's code, database schema, API payloads and logs are in **English**. The French
vocabulary teachers actually use appears only in the **UI**, resolved through the
translation layer. This file is the agreed mapping between the two.

Settle a term here *before* it reaches a migration or a template. Renaming a table is
cheap; renaming a concept that has leaked into forty files is not.

## Conventions

- **No franglais.** `seance`, `eleveList`, `cahier_journal` are all wrong. If a French word
  ends up in an identifier, it is a bug in this glossary — fix the glossary, then the code.
- **UI strings are never literals.** Every user-facing string is a catalogue key resolved at
  render time, even though `fr` is the only locale that will exist.
- **Identifiers are English; data can be French.** See the boundaries below.
- One concept, one English name. If two names appear for the same thing, this file decides.

## People and structure

| Domain (FR) | Code / schema (EN) | Notes |
| --- | --- | --- |
| enseignant, professeur des écoles | `teacher` | The only account type. |
| élève | `pupil` | Not `student` — this is primary school. |
| classe | `class_group` | `class` is a reserved word in most languages. |
| école | `school` | |
| directeur, directrice | `head_teacher` | Not `director`. |
| niveau (CP, CE1, …) | `level` | A `class_group` can span several — multi-level is the normal case. |
| cycle | `cycle` | Cycles 1–3 group levels. |
| groupe | `pupil_group` | A subset of a class's pupils. Not `group`, which SQL reserves. |
| atelier | `workshop` | The activity a `pupil_group` rotates through. |

## Time

| Domain (FR) | Code / schema (EN) | Notes |
| --- | --- | --- |
| année scolaire | `school_year` | Straddles two calendar years — store both bounds. |
| période, trimestre, semestre | `school_period` | The unit reports are scoped to. Not `period`, which clashes with `java.time.Period`. |
| semaine | `week` | |
| emploi du temps | `timetable` | The spine of the model. |
| créneau | `time_slot` | A recurring weekly slot in the timetable. |
| séance | `lesson` | A concrete occurrence of a slot on a date. |
| vacances | `holiday` | From the Ministry open-data calendar. |
| zone (A, B, C) | `zone` | Determines `school_period` boundaries. |
| jour férié | `public_holiday` | Distinct from school holidays. |

## Teaching content

| Domain (FR) | Code / schema (EN) | Notes |
| --- | --- | --- |
| cahier journal | `daily_log` | |
| matière, discipline | `subject` | |
| domaine (d'un programme) | `strand` | Subdivision of a subject — Français → Lecture, Écriture… Not `domain`, which clashes with "domain model". |
| objectif | `objective` | |
| programme (officiel) | `curriculum` | The national reference data. |
| compétence | `competency` | |

### *Domaine* is overloaded

The French word covers four different things; map by meaning, not by word:

| Where it appears | Meaning | Maps to |
| --- | --- | --- |
| Programmes, cycles 2–3 | Subdivision of a subject | `strand` |
| Maternelle, « 5 domaines d'apprentissage » | Plays the role of a subject | `subject` |
| LSU, *domaine d'enseignement* / *sous-domaine* | Roughly `subject` / `strand` | Export mapping layer only |
| Socle commun, « 5 domaines » | A separate framework | `common_core_domain`, if ever needed |

## Assessment

| Domain (FR) | Code / schema (EN) | Notes |
| --- | --- | --- |
| carnet de notes | `grade_book` | |
| évaluation | `assessment` | The act of assessing. |
| note, résultat | `grade` | The recorded value. |
| échelle de notation | `grading_scale` | %, /10, A/PA/NA, or custom. |
| acquis / en cours d'acquisition / non acquis | `acquired` / `in_progress` / `not_acquired` | Competency state in an `achievement_record`. Keep the three states explicit, not a boolean. |
| moyenne | `average` | |
| carnet de réussites | `achievement_record` | Maternelle-facing counterpart to grades. |
| bilan périodique | `periodic_report` | What LSU consumes. |
| bilan de fin de cycle | `end_of_cycle_report` | |
| livret scolaire (LSU) | `school_report` | Reserve `lsu` for the export format itself. |
| appréciation, commentaire | `comment` | |

## Boundaries — where French legitimately remains

Three places, all of them data or contract rather than identifiers:

1. **UI strings** — the whole point of the translation layer.
2. **Reference data values.** A subject really is named « Questionner le monde ». English
   column names, French values, and no attempt to translate the official programmes.
3. **The LSU export.** Its element and attribute names are fixed by the official spec and
   are French. Confine them to a mapping layer at the export boundary — serialisation tags
   or an explicit translation step — so they never reach the domain model or the schema.

## Deferred terms

Paid-plan vocabulary, listed so that if these features ever arrive the naming is not
improvised. Out of scope for now — see [roadmap-free-plan.md](roadmap-free-plan.md).

| Domain (FR) | Code / schema (EN) |
| --- | --- |
| fiche de préparation | `lesson_plan` |
| progression, programmation | `progression`, `year_plan` |
| registre d'appel | `attendance_register` |
| absence, retard | `absence`, `lateness` |
| prise de rendez-vous | `appointment` |
| budget de classe | `class_budget` |
| trombinoscope | `class_photo_board` |
| pyramide des âges | `age_distribution` |
