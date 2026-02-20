# ✨ Feature: Audit Trail für Ergebniseintragungen

## 📖 User Story
> **As a** Turnonio-Administrator oder Turnierorganisator
> **I want to** nachvollziehen können, welcher Nutzer ein bestimmtes Ergebnis eingetragen oder geändert hat
> **so that** ich bei fehlerhaften oder strittigen Ergebnissen die Verantwortlichkeit klären und Fehlerquellen identifizieren kann.

## 📝 Description
Derzeit werden Ergebnisse in Turnieren erfasst, aber es fehlt die Information, welcher User (Schiedsrichter, Organisator, Admin) das Ergebnis eingetragen hat. Diese Information ist systemseitig verfügbar, wird aber weder persistiert noch in der UI angezeigt. Ein Audit Trail ermöglicht Fehleranalyse, erhöht die Transparenz und hilft bei der Nachverfolgung von Manipulationen oder versehentlichen Falscheingaben.

## 🎯 Acceptance Criteria
*(Please formulate specifically and testably)*
- [ ] Bei jeder Ergebniseintragung wird der eintragende User (ID, Username) mit Zeitstempel in der Datenbank gespeichert
- [ ] In der Ergebnisdetailansicht wird angezeigt: "Eingetragen von [Username] am [Datum/Uhrzeit]"
- [ ] Bei Änderungen eines bereits eingetragenen Ergebnisses wird ein vollständiger Änderungsverlauf (History) gespeichert (Wer? Wann? Altes Ergebnis? Neues Ergebnis?)
- [ ] Organisatoren und Admins können die Audit-Informationen in der UI einsehen
- [ ] Die Audit-Daten sind vor nachträglicher Manipulation geschützt (append-only Log)
- [ ] Filter-/Suchfunktion: "Zeige mir alle Ergebnisse, die User X eingetragen hat"

## ❓ Open Questions
*(List all unclear aspects, missing information, or decisions that need to be made)*
- Sollen auch reguläre Teilnehmer die Information sehen können, wer ein Ergebnis eingetragen hat, oder nur privilegierte Nutzer (Organisatoren, Admins)?
- Wie lange soll die Audit-History aufbewahrt werden? Gibt es ein Retention-Konzept (z.B. nach Turnierablauf x Monate)?
- Soll es eine Benachrichtigung geben, wenn ein Ergebnis nachträglich geändert wird (z.B. E-Mail an Organisator)?
- Gibt es einen Anwendungsfall für "Ergebnis eingetragen im Namen von" (z.B. wenn ein Admin für einen Schiedsrichter einträgt)?
- Soll die Audit-Information auch in Exporten (CSV, PDF, etc.) enthalten sein?
- Wie soll mit automatisch generierten Ergebnissen umgegangen werden (z.B. bei Forfeit/Walkover)? System-User oder letzter manueller Eingriff?

## 🛠️ Technical Notes & Ideas
- **Frontend / UI:** 
  - Kompakte Darstellung in der Match-/Ergebnisansicht (z.B. kleines Icon mit Tooltip oder Info-Badge)
  - Optionale Detail-Ansicht mit vollständigem Änderungsverlauf (Modal oder Accordion)
  - Admin-Dashboard: Übersicht über kürzlich eingetragene/geänderte Ergebnisse mit User-Zuordnung
  
- **Backend / Data:** 
  - Erweiterung des Match/Result-Models um `created_by_user_id`, `created_at`, `updated_by_user_id`, `updated_at`
  - Separate Audit-Log-Tabelle für History: `result_audit_log` (result_id, user_id, timestamp, action_type, old_value, new_value)
  - Middleware/Interceptor, der automatisch bei jeder Result-Mutation die User-Info aus dem aktuellen Auth-Context speichert
  - API-Endpoint für Audit-Log-Abfragen (z.B. GET `/api/matches/{id}/audit-log`)
  
- **Technical Considerations:** 
  - Performance: Bei großen Turnieren können viele Audit-Einträge entstehen → Index auf `result_id` und `timestamp`
  - Datenschutz: Klärung, ob personenbezogene Daten (Usernames) langfristig gespeichert werden dürfen (DSGVO)
  - Migrationsstrategie: Bestehende Ergebnisse haben keine User-Info → als "Legacy/Unknown" markieren oder System-User zuweisen?

---
*Created by the Turnonio Feature Agent 🤖🏆*
