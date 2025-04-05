# 📘 INSTEAD OF Trigger in PL/SQL

## 📌 Einleitung: Was ist ein INSTEAD OF Trigger?

Ein `INSTEAD OF` Trigger wird auf **Views** definiert und **ersetzt eine DML-Operation** (`INSERT`, `UPDATE`, `DELETE`), die eigentlich nicht direkt auf einem View ausgeführt werden kann. Das bedeutet:

- Die Aktion (z. B. ein `INSERT` in den View) wird **nicht vom DBMS ausgeführt**, sondern der Trigger führt **eine benutzerdefinierte Operation** auf den zugrunde liegenden Tabellen aus.
- Das ist besonders nützlich, wenn der View aus mehreren Tabellen besteht oder berechnete/umbenannte Felder enthält.

---

## 🔄 Allgemeines Trigger-Beispiel (Schablone)

```sql
CREATE OR REPLACE TRIGGER trigger_name                     -- 🔸 Trigger wird erstellt oder ersetzt
INSTEAD OF INSERT OR UPDATE OR DELETE ON view_name         -- 🔸 Gilt für eine View – ersetzt DML-Operationen (INSERT/UPDATE/DELETE)
FOR EACH ROW                                               -- 🔸 Trigger wird für jede betroffene Zeile einzeln ausgeführt
DECLARE
    v_variable table.column%TYPE;                          -- 📦 Lokale Variable, z. B. um Werte zwischenzuspeichern
BEGIN
    IF INSERTING THEN                                      -- ➕ Wenn ein INSERT erfolgt
        INSERT INTO base_table (spalte1, spalte2, ...)     -- 🧾 Datensatz in Basistabelle einfügen
        VALUES (:NEW.spalte1, :NEW.spalte2, ...);          --     mit Werten aus der View-Zeile

    ELSIF UPDATING THEN                                    -- 🔁 Wenn ein UPDATE erfolgt
        UPDATE base_table                                  -- 🛠️ Update in der Basistabelle
        SET spalte1 = :NEW.spalte1,
            spalte2 = :NEW.spalte2
        WHERE primary_key = :OLD.primary_key;              -- 🎯 Identifikation über Primärschlüssel

    ELSIF DELETING THEN                                    -- ❌ Wenn ein DELETE erfolgt
        DELETE FROM base_table                             -- 🗑️ Datensatz in Basistabelle löschen
        WHERE primary_key = :OLD.primary_key;
    END IF;

EXCEPTION
    WHEN NO_DATA_FOUND THEN                                -- ❗ Kein Datensatz gefunden
        RAISE_APPLICATION_ERROR(-20001, 'Datensatz nicht gefunden!');
    WHEN OTHERS THEN                                       -- ⚠️ Alle anderen Fehler
        RAISE_APPLICATION_ERROR(-20002, 'Ein unbekannter Fehler ist aufgetreten!');
END;
/
```

---

## 🧠 Aufgabe 1.1 – View mit Abteilungsname (statt `DEPTNO`)
 1.1. Instead of Trigger, typische Beispiele
        Erzeugen Sie einen View in welchem der Mitarbeiter samt dem Namen seiner Abteilung ausgegeben wird (ohne PK der Abteilung).
            bei insert und update soll anhand des eingefügten Namens die korrekte PK<->FK-Beziehung hergestellt werden, oder
            ein Fehler geworfen werden, wenn es keine Abteilung mit dem gegebenen Namen gibt.
        
### 🎯 Ziel der Aufgabe:
Wir möchten einen View erstellen, der **Mitarbeiterdaten mit dem Abteilungsnamen (`DNAME`)** anzeigt, statt mit der technischen `DEPTNO`. 

Der Benutzer soll über den View `INSERT` und `UPDATE` machen dürfen, **ohne die DEPTNO kennen zu müssen**. Der Trigger übernimmt dann automatisch die Umwandlung.

### 👀 View: `empno_with_dept_name`

```sql
CREATE OR REPLACE VIEW empno_with_dept_name AS
SELECT
    e.empno,              -- 🔢 Mitarbeiter-ID
    e.ename,              -- 🧑 Name
    e.job,                -- 💼 Jobbezeichnung
    e.sal,                -- 💰 Gehalt
    d.dname               -- 🏢 Abteilungsname (statt DEPTNO)
FROM emp e
JOIN dept d ON e.deptno = d.deptno;  -- 🔗 Fremdschlüsselbeziehung
```

### 🛠 Trigger: `trg_emp_with_dname`

```sql
CREATE OR REPLACE TRIGGER trg_emp_with_dname
INSTEAD OF INSERT OR UPDATE ON empno_with_dept_name
FOR EACH ROW
DECLARE
    v_deptno dept.deptno%TYPE;                         -- 📦 Zwischenspeicher für DEPTNO
BEGIN
    SELECT deptno INTO v_deptno
    FROM dept
    WHERE dname = :NEW.dname;                          -- 🔍 Umwandlung dname → deptno

    IF INSERTING THEN
        INSERT INTO emp (empno, ename, job, sal, deptno)
        VALUES (:NEW.empno, :NEW.ename, :NEW.job, :NEW.sal, v_deptno);

    ELSIF UPDATING THEN
        UPDATE emp
        SET ename = :NEW.ename,
            job   = :NEW.job,
            sal   = :NEW.sal,
            deptno = v_deptno
        WHERE empno = :OLD.empno;
    END IF;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20001, 'Abteilung nicht gefunden: ' || :NEW.dname);
END;
/
```

### ✅ Testfälle

```sql
-- Einfügen eines neuen Mitarbeiters mit gültigem Abteilungsnamen
INSERT INTO empno_with_dept_name (empno, ename, job, sal, dname)
VALUES (1234, 'MAX', 'CLERK', 1500, 'SALES');

-- Update des Mitarbeiters mit anderer Abteilung
UPDATE empno_with_dept_name
SET dname = 'RESEARCH', sal = 1600
WHERE empno = 1234;

-- Fehlerfall: ungültiger Abteilungsname
INSERT INTO empno_with_dept_name (empno, ename, job, sal, dname)
VALUES (1235, 'FAILTEST', 'CLERK', 1400, 'NICHTEXISTENT');
```

---

## 🧠 Aufgabe 1.2 – View mit Datum im Format `YYYY-MM-DD`
Ein View, in dem sämtliche Datumswerte bereits im Vorfeld im Format YYYY-MM-DD ausgegeben werden, Update  und Insert sollen möglich sein.
### 🎯 Ziel der Aufgabe:
Der Benutzer soll über einen View auf das `HIREDATE`-Datum im lesbaren Format `YYYY-MM-DD` zugreifen und trotzdem `INSERT` und `UPDATE` durchführen können. Der Trigger übernimmt die Umwandlung in das echte DATE-Format.

### 👀 View: `date_format_view`

```sql
CREATE OR REPLACE VIEW date_format_view AS
SELECT
    empno,                                            -- 🔢 Mitarbeiter-ID
    ename,                                            -- 🧑 Name
    TO_CHAR(hiredate, 'YYYY-MM-DD') AS format_hiredate -- 📅 Datum als Text
FROM emp;
```

### 🔄 Trigger: `trg_format_date`

```sql
CREATE OR REPLACE TRIGGER trg_format_date
INSTEAD OF INSERT OR UPDATE ON date_format_view
FOR EACH ROW
DECLARE
    dateval DATE;                                           -- 📦 Umgewandeltes Datum
BEGIN
    dateval := TO_DATE(:NEW.format_hiredate, 'YYYY-MM-DD'); -- 🔄 Rückkonvertierung ins DATE-Format

    IF INSERTING THEN
        INSERT INTO emp (empno, ename, hiredate)
        VALUES (:NEW.empno, :NEW.ename, dateval);

    ELSIF UPDATING THEN
        UPDATE emp
        SET ename = :NEW.ename,
            hiredate = dateval
        WHERE empno = :OLD.empno;
    END IF;
END;
/
```

### ✅ Testfälle

```sql
-- Neuer Mitarbeiter mit formatiertem Datum
INSERT INTO date_format_view (empno, ename, format_hiredate)
VALUES (9955, 'DATUMTEST', '2025-04-04');

-- Update des Einstellungsdatums
UPDATE date_format_view
SET format_hiredate = '2025-05-01'
WHERE empno = 9955;

-- Kontrolle der Darstellung
SELECT * FROM date_format_view WHERE empno = 9955;
```

---
# 📘 INSTEAD OF Trigger in PL/SQL – Zusammenfassung

## 🧩 Übersicht der Aufgaben 1.1 und 1.2 – INSTEAD OF Trigger

| Aufgabe  | Thema                              | Ziel                                                                 |
|----------|------------------------------------|----------------------------------------------------------------------|
| 1.1      | Abteilungsname statt DEPTNO im View| Benutzer sieht DNAME, Trigger wandelt zu DEPTNO um                  |
| 1.2      | Datum als Text anzeigen (YYYY-MM-DD)| Benutzer sieht/ändert Datum als Text, Trigger wandelt zurück in DATE |

---

## 🟢 Aufgabe 1.1 – View mit DNAME statt DEPTNO

### 🔍 View-Idee
Benutzer sieht Abteilungsname (`dname`) statt der Fremdschlüsselspalte `deptno`.  
Der Trigger macht `INSERT` und `UPDATE` über diesen View möglich, indem er den Namen in die Nummer umwandelt.

### ✅ Vorteile
- Benutzer muss keine technischen Schlüssel kennen
- Fehlerbehandlung, falls `dname` ungültig ist

### 🧠 Triggerlogik
- `SELECT deptno INTO ... FROM dept WHERE dname = :NEW.dname`
- Bei INSERT: Werte + `deptno` in Tabelle `emp` einfügen
- Bei UPDATE: bestehende Zeile aktualisieren

---

## 🟢 Aufgabe 1.2 – Datum als Text anzeigen

### 🔍 View-Idee
Datum `hiredate` soll im Format `YYYY-MM-DD` dargestellt und bearbeitet werden.

### ✅ Vorteile
- Benutzer sieht klar lesbares Datum
- `INSERT` und `UPDATE` ohne manuelles Parsen möglich

### 🧠 Triggerlogik
- `TO_DATE(:NEW.format_hiredate, 'YYYY-MM-DD')` konvertiert Eingabe in echtes DATE
- Werte werden in `emp` gespeichert

---

## 🧪 Vergleich der Trigger

| Merkmal                          | Aufgabe 1.1                                | Aufgabe 1.2                                   |
|----------------------------------|--------------------------------------------|-----------------------------------------------|
| Art des Views                    | `dname` statt `deptno`                     | Datum als Text (format_hiredate)              |
| Komplexität                      | mittel                                     | eher gering                                   |
| Konvertierung                    | Name → Nummer (via SELECT)                 | Text → DATE (via TO_DATE)                     |
| Fehlerbehandlung                 | Ja, bei ungültigem `dname`                 | Nicht notwendig, da DATE-Format festgelegt    |
| Trigger-Typ                      | `INSTEAD OF INSERT OR UPDATE`              | `INSTEAD OF INSERT OR UPDATE`                 |
| Vorteile                         | Benutzerfreundlich, lesbar, sicheres Mapping | Benutzerfreundliches Datumsformat             |

---

## 📦 Fazit – Wann INSTEAD OF Trigger verwenden?

| Wenn …                                                                 | Dann nutze …        |
|------------------------------------------------------------------------|----------------------|
| … der View aus mehreren Tabellen besteht                              | `INSTEAD OF` Trigger |
| … Felder im View nicht direkt änderbar sind (z. B. berechnete Felder) | `INSTEAD OF` Trigger |
| … Benutzer nur logische Namen sehen soll (statt IDs)                  | `INSTEAD OF` Trigger |
| … Formatierung (z. B. von Datum) im View sichtbar sein soll           | `INSTEAD OF` Trigger |
