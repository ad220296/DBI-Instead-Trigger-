# DBI-Instead-Trigger-


# 🧠 INSTEAD OF Trigger in SQL

---

## 📚 Allgemeines Beispiel – INSTEAD OF Trigger (Vorlage)

```sql
create or replace trigger trigger_name                     -- 🔸 Trigger wird erstellt oder ersetzt
instead of insert or update or delete on view_name         -- 🔸 Gilt für eine View – ersetzt DML-Operationen (INSERT/UPDATE/DELETE)
for each row                                               -- 🔸 Trigger wird für jede betroffene Zeile einzeln ausgeführt
declare
    v_variable table.column%type;                          -- 📦 Lokale Variable, z. B. zum Zwischenspeichern
begin
    if inserting then                                      -- ➕ Wenn ein INSERT erfolgt
        insert into base_table (spalte1, spalte2, ...)
        values (:new.spalte1, :new.spalte2);               -- 🧾 INSERT in Basistabelle

    elsif updating then                                    -- 🔁 Wenn ein UPDATE erfolgt
        update base_table
        set spalte1 = :new.spalte1,
            spalte2 = :new.spalte2
        where primary_key = :old.primary_key;              -- 🎯 Identifikation über Primärschlüssel

    elsif deleting then                                    -- ❌ Wenn ein DELETE erfolgt
        delete from base_table
        where primary_key = :old.primary_key;              -- 🗑️ Zeile löschen
    end if;

exception
    when no_data_found then                                -- ❗ Kein passender Datensatz gefunden
        raise_application_error(-20001, 'Datensatz nicht gefunden!');

    when others then                                       -- ⚠️ Allgemeiner Fehler
        raise_application_error(-20002, 'Ein unbekannter Fehler ist aufgetreten!');
end;
/
```

---

## 🧠 Aufgabe 1.1 – View mit Abteilungsname (statt deptno)

### 🎯 Ziel

Ein View zeigt **Mitarbeiterdaten mit `dname`** (statt `deptno`).  
Über den View sollen INSERT und UPDATE möglich sein.  
Ein INSTEAD OF Trigger übernimmt intern die Zuordnung von `dname → deptno`.

---

### 👀 View erstellen

```sql
create or replace view emp_with_deptno_name as
select
    e.ename,                       -- 🧑 Name
    e.empno,                       -- 🔢 Mitarbeiterschlüssel
    e.sal,                         -- 💰 Gehalt
    d.dname                        -- 🏢 Abteilungsname
from emp e
join dept d on e.deptno = d.deptno; -- 🔗 FK-Verbindung EMP → DEPT
```

---

### 🔄 Trigger zur Verarbeitung von Insert/Update

```sql
create or replace trigger trg_insert_emp_with_dname
instead of insert or update on emp_with_deptno_name
for each row
declare
    v_deptno dept.deptno%type;     -- 📦 Speichert die echte DEPTNO
begin
    select deptno into v_deptno
    from dept
    where dname = :new.dname;

    if inserting then
        insert into emp (empno, ename, sal, deptno)
        values (:new.empno, :new.ename, :new.sal, v_deptno);

    elsif updating then
        update emp
        set ename = :new.ename,
            sal = :new.sal,
            deptno = v_deptno
        where empno = :old.empno;
    end if;

exception
    when no_data_found then
        raise_application_error(-20001, 'Abteilung nicht gefunden: ' || :new.dname);
end;
/
```

---

### 🧪 Testfälle für View `emp_with_deptno_name`

```sql
-- ✅ Insert mit gültigem dname
insert into emp_with_deptno_name (empno, ename, sal, dname)
values (9999, 'Testuser', 3000, 'SALES');

-- ❌ Insert mit ungültigem dname
insert into emp_with_deptno_name (empno, ename, sal, dname)
values (9998, 'Fehltest', 3000, 'NICHTEXISTENT');

-- ✅ Insert eines weiteren gültigen Mitarbeiters
insert into emp_with_deptno_name (empno, ename, sal, dname)
values (1001, 'Hans', 2800, 'SALES');

-- 🔁 Update: Gehalt + Abteilung ändern
update emp_with_deptno_name
set sal = 3500, dname = 'RESEARCH'
where empno = 1001;

-- ❌ Update mit ungültiger Abteilung
update emp_with_deptno_name
set dname = 'NICHTEXISTENT'
where empno = 1001;
```

---

## 🧠 Aufgabe 1.2 – View mit formatiertem Datum (YYYY-MM-DD)

### 🎯 Ziel

Ein View zeigt das `hiredate`-Datum im Format `YYYY-MM-DD`.  
INSERT und UPDATE sollen über den View möglich sein.  
Ein Trigger konvertiert das Textdatum zurück in SQL `DATE`.

---

### 👀 View mit formatiertem Datum

```sql
create or replace view date_format_view as
select
    empno,                                             -- 🔢 Mitarbeiter-ID
    ename,                                             -- 🧑 Name
    to_char(hiredate, 'YYYY-MM-DD') as format_hiredate -- 📅 Datum als Text
from emp;
```

---

### 🔄 Trigger zur Rückkonvertierung

```sql
create or replace trigger trg_format_date 
instead of insert or update on date_format_view
for each row
declare
    dateval date;  -- 📦 Speichert das echte Datum
begin
    dateval := to_date(:new.format_hiredate, 'YYYY-MM-DD');

    if inserting then
        insert into emp (empno, ename, hiredate)
        values (:new.empno, :new.ename, dateval);

    elsif updating then
        update emp
        set ename = :new.ename,
            hiredate = dateval
        where empno = :old.empno;
    end if;
end;
/
```

---

### 🧪 Testfälle für View `date_format_view`

```sql
-- ✅ Insert mit gültigem Datum
insert into date_format_view (empno, ename, format_hiredate)
values (9955, 'DATUMTEST', '2025-04-04');

-- 🔁 Update des Datums
update date_format_view
set format_hiredate = '2025-05-01'
where empno = 9955;

-- 📋 Kontrolle
select * from date_format_view where empno = 9955;
```

---

