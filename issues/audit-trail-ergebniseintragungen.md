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
- [ ] In der Ergebnisdetailansicht wird angezeigt: "Eingetragen von [Username] am [Datum/Uhrzeit]" - nur sichtbar für Nutzer mit direktem Turnier-Zugriff (Organisatoren, Schiedsrichter, Admins)
- [ ] Bei Änderungen eines bereits eingetragenen Ergebnisses wird ein vollständiger Änderungsverlauf (History) gespeichert (Wer? Wann? Altes Ergebnis? Neues Ergebnis?)
- [ ] Die Audit-Informationen sind **nicht** im Live-Dashboard für Zuschauer sichtbar
- [ ] Die Audit-Informationen sind **nicht** in CSV-Exporten enthalten
- [ ] Nur Nutzer mit direktem Turnier-Zugriff können die Audit-Informationen in der UI einsehen
- [ ] Die Audit-Daten sind vor nachträglicher Manipulation geschützt (append-only Log)
- [ ] Filter-/Suchfunktion für privilegierte Nutzer: "Zeige mir alle Ergebnisse, die User X eingetragen hat"

## ❓ Open Questions
*(List all unclear aspects, missing information, or decisions that need to be made)*
- ✅ **Geklärt:** Nur Personen mit direktem Zugriff auf das Turnier (Organisatoren, Schiedsrichter, Admins) dürfen sehen, wer das Ergebnis eingetragen hat. Keine Zuschauer über das Live-Dashboard.
- ✅ **Geklärt:** Die Informationen bleiben analog zu den anderen Turnierdaten vorhanden (gleiche Retention-Policy).
- ✅ **Geklärt:** Im CSV-Export sollen diese Informationen **nicht** enthalten sein.
- ✅ **Geklärt:** Es gibt keinen Anwendungsfall für "Ergebnis eingetragen im Namen von" - der Name wird nicht überschrieben.
- Soll es eine Benachrichtigung geben, wenn ein Ergebnis nachträglich geändert wird (z.B. E-Mail an Organisator)?
- Wie soll mit automatisch generierten Ergebnissen umgegangen werden (z.B. bei Forfeit/Walkover)? System-User oder letzter manueller Eingriff?

## 🛠️ Technical Notes & Ideas
- **Frontend / UI:** 
  - Kompakte Darstellung in der Match-/Ergebnisansicht (z.B. kleines Icon mit Tooltip oder Info-Badge)
  - **Wichtig:** Audit-Informationen nur für Nutzer mit Turnier-Zugriff rendern (Permission-Check im Frontend)
  - Live-Dashboard für Zuschauer: Audit-Informationen komplett ausblenden
  - Optionale Detail-Ansicht mit vollständigem Änderungsverlauf (Modal oder Accordion) - nur für berechtigte Nutzer
  - Admin-Dashboard: Übersicht über kürzlich eingetragene/geänderte Ergebnisse mit User-Zuordnung
  
- **Backend / Data:** 
  - Erweiterung des Match/Result-Models um `created_by_user_id`, `created_at`, `updated_by_user_id`, `updated_at`
  - Separate Audit-Log-Tabelle für History: `result_audit_log` (result_id, user_id, timestamp, action_type, old_value, new_value)
  - Middleware/Interceptor, der automatisch bei jeder Result-Mutation die User-Info aus dem aktuellen Auth-Context speichert
  - API-Endpoint für Audit-Log-Abfragen (z.B. GET `/api/matches/{id}/audit-log`) mit Permission-Check
  - **CSV-Export:** Audit-Felder explizit von der Export-Logik ausschließen
  
- **Technical Considerations:** 
  - Performance: Bei großen Turnieren können viele Audit-Einträge entstehen → Index auf `result_id` und `timestamp`
  - Datenschutz: Audit-Daten unterliegen der gleichen Retention-Policy wie andere Turnierdaten (DSGVO-konform)
  - Migrationsstrategie: Bestehende Ergebnisse haben keine User-Info → als "Legacy/Unknown" markieren oder System-User zuweisen?
  - Berechtigungsprüfung: Strikte Trennung zwischen Turnier-Teilnehmern (kein Zugriff) und Organisatoren/Schiedsrichtern/Admins (voller Zugriff)

---
*Created by the Turnonio Feature Agent 🤖🏆*
