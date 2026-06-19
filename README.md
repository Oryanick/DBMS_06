# DBMS_06 – PostgreSQL in Practice: DDL, DML, and Bulk Import

**Module:** Databases · THGA Bochum  
**Lecturer:** Stephan Bökelmann · <sboekelmann@ep1.rub.de>  
**Repository:** <https://github.com/MaxClerkwell/DBMS_06>  
**Prerequisites:** DBMS_01, DBMS_02, DBMS_03, DBMS_04, DBMS_05, Lecture 06  
**Duration:** 90 minutes

---

## Learning Objectives

After completing this exercise you will be able to:

- Connect to a remote Linux server via **SSH** and work confidently in a
  terminal-only environment
- Verify that a running PostgreSQL instance is available and understand the
  role of the `postgres` system user
- Create a **database user** and a **database** using standard SQL commands
  from the `psql` client
- Re-implement a known relational schema in PostgreSQL by typing **DDL
  statements** interactively in `psql`
- Insert rows **one at a time** using `INSERT` and observe immediate feedback
  from the database engine
- Prepare a **CSV file by hand** and load its contents into a table using
  the standard SQL `COPY` statement
- Formulate and run **three analytical queries** directly in the `psql` shell
- Create and populate a second, independent database entirely from a
  **`.sql` script**

**After completing this exercise you should be able to answer the following questions independently:**

- What is the difference between the `postgres` operating-system user and a
  PostgreSQL database role?
- Why must foreign keys be defined in dependency order when creating tables?
- What are the advantages of `COPY FROM` over individual `INSERT` statements
  for large data volumes?
- What does `\q`, `\dt`, `\d <table>`, and `\c <database>` do inside `psql`?

---

## 0 – Connect to the Server

You can complete this exercise either on the **THGA lecture server** or on
**your own machine** running a Debian-based Linux distribution (Ubuntu, Debian,
WSL2, etc.).

### Option A – Lecture Server (SSH)

```bash
ssh <your-username>@<server-ip>
```

> You should see a shell prompt on the remote machine.
> If you are asked about the host fingerprint, type `yes` and press Enter.

> **Screenshot 1:** Take a screenshot of your terminal showing a successful
> SSH login (the welcome banner and your shell prompt) and insert it here.
>
> `[insert screenshot]`

### Option B – Your Own Machine

Open a terminal. All subsequent commands run locally — skip the `ssh` step.

---

## 1 – Verify PostgreSQL is Available

### Option A – Lecture Server

PostgreSQL is already installed and running on the lecture server. Just
confirm that the service is reachable and check the client version:

```bash
pg_isready
psql --version
```

> `pg_isready` should print something like
> `/var/run/postgresql:5432 - accepting connections`.

### Option B – Your Own Machine

If you are working on your own Debian-based system, install PostgreSQL first:

```bash
sudo apt-get update
sudo apt-get install -y postgresql postgresql-client
```

Then verify:

```bash
pg_isready
psql --version
```

> **Screenshot 2:** Take a screenshot showing the output of both commands.
>
> `![Screenshot2](https://github.com/Oryanick/DBMS_06/blob/master/Screenshot%202.png)


---

## 2 – Create a Database User and a Database

PostgreSQL uses the concept of **roles** for access control. After installation,
only the `postgres` superuser role exists. You will create a dedicated role for
this exercise and assign a new database to it.

Switch to the `postgres` system user to open a superuser session:

```bash
sudo -u postgres psql
```

You should now see the `psql` prompt:

```
psql (16.x)
Type "help" for help.

postgres=#
```

Create your role (replace `<your-username>` with your actual Unix login name
so that you can connect without specifying `-U` later):

```sql
CREATE ROLE <your-username> WITH LOGIN PASSWORD '<choose-a-password>';
```

Create the database and assign ownership to your new role:

```sql
CREATE DATABASE bibliothek_<your-username> OWNER <your-username>;
```

Verify:

```sql
SELECT rolname FROM pg_roles WHERE rolname = '<your-username>';
SELECT datname, pg_catalog.pg_get_userbyid(datdba) AS owner
FROM   pg_database
WHERE  datname = 'bibliothek_<your-username>';
```

Exit the superuser session:

```sql
\q
```

> **Screenshot 3:** Take a screenshot showing the `CREATE ROLE`, `CREATE DATABASE`,
> and both `SELECT` results inside the `postgres=#` session.
>
> ![Screenshot3](https://github.com/Oryanick/DBMS_06/blob/master/Screenshot%203.png)

---

## 3 – Connect as Your Own User

```bash
psql -U <your-username> -d bibliothek_<your-username>
```

You should see:

```
psql (16.x)
Type "help" for help.

bibliothek=>
```

> From this point on, every `psql` session in this exercise connects with
> `psql -U <your-username> -d bibliothek_<your-username>` unless stated otherwise.

---

## 4 – Create the Schema

You will now recreate the **municipal library** schema from DBMS_05 in
PostgreSQL. Type each statement individually and press Enter to execute it.
Do not paste all statements at once.

The schema consists of four tables:

| Table      | Description                                      |
|------------|--------------------------------------------------|
| `buch`     | Books, identified by ISBN                        |
| `exemplar` | Physical copies of a book                        |
| `mitglied` | Library members                                  |
| `ausleihe` | Lending transactions (copy ↔ member)             |

Type and execute the following statements one by one:

```sql
CREATE TABLE buch (
    isbn              TEXT           PRIMARY KEY,
    titel             TEXT           NOT NULL,
    erscheinungsjahr  INTEGER        NOT NULL,
    verlag            TEXT           NOT NULL,
    tagesgebuehr      NUMERIC(6,2)   NOT NULL CHECK (tagesgebuehr > 0)
);
```

```sql
CREATE TABLE exemplar (
    exemplar_id  INTEGER  PRIMARY KEY,
    isbn         TEXT     NOT NULL,
    standort     TEXT     NOT NULL,
    FOREIGN KEY (isbn) REFERENCES buch(isbn)
        ON DELETE RESTRICT ON UPDATE CASCADE
);
```

```sql
CREATE TABLE mitglied (
    mitglied_id     INTEGER      PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    nachname        TEXT         NOT NULL,
    vorname         TEXT         NOT NULL,
    geburtsdatum    DATE         NOT NULL,
    email           TEXT         NOT NULL UNIQUE,
    beitritt_datum  DATE         NOT NULL DEFAULT CURRENT_DATE
);
```

```sql
CREATE TABLE ausleihe (
    ausleihe_id      INTEGER  PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    exemplar_id      INTEGER  NOT NULL,
    mitglied_id      INTEGER  NOT NULL,
    ausleihe_datum   DATE     NOT NULL,
    rueckgabe_datum  DATE,
    FOREIGN KEY (exemplar_id) REFERENCES exemplar(exemplar_id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    FOREIGN KEY (mitglied_id) REFERENCES mitglied(mitglied_id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    CHECK (rueckgabe_datum IS NULL OR rueckgabe_datum >= ausleihe_datum)
);
```

Verify that all four tables were created:

```sql
\dt
```

Inspect the structure of one table:

```sql
\d ausleihe
```

> **Screenshot 4:** Take a screenshot showing the output of `\dt` and
> `\d ausleihe`.
>
> ![Screenshot4](https://github.com/Oryanick/DBMS_06/blob/master/Screenshot%204.png)

### Questions for Section 4

**Question 4.1:** You had to create `buch` before `exemplar`, and `exemplar`
and `mitglied` before `ausleihe`. Why does this order matter? What error would
PostgreSQL report if you tried to create `ausleihe` first?

> *Your answer:* Die Reihenfolge ist wichtig, weil Foreign Keys nur auf Tabellen zeigen können, die schon existieren. exemplar verweist auf buch (isbn), ausleihe verweist auf exemplar (exemplar_id) und mitglied (mitglied_id). Wenn man ausleihe zuerst erstellt, existieren die Tabellen, auf die verwiesen wird, noch nicht.
Dann kommt in PostgreSQL ein Fehler wie:
ERROR: relation “exemplar” does not exist
ERROR: relation “mitglied” does not exist


**Question 4.2:** The `mitglied_id` and `ausleihe_id` columns use
`GENERATED ALWAYS AS IDENTITY`. What does this mean? What happens if you try to
supply a value explicitly with `INSERT INTO mitglied (mitglied_id, ...) VALUES (5, ...)`?

> *Your answer:* GENERATED ALWAYS AS IDENTITY bedeutet, dass PostgreSQL die ID automatisch erzeugt, vergleichbar mit AUTO_INCREMENT. Die Datenbank vergibt die Werte selbst. Wenn man versucht, selbst eine ID einzutragen:
INSERT INTO mitglied (mitglied_id, ...) VALUES (5, ...), kommt ein Fehler: ERROR: cannot insert into column “mitglied_id”, außer man verwendet ausdrücklich OVERRIDING SYSTEM VALUE.

**Question 4.3:** `tagesgebuehr` is defined as `NUMERIC(6,2)` while a simpler
`REAL` would also hold decimal numbers. Give a concrete example of an arithmetic
result that would differ between the two types when calculating a lending fee.

> *Your answer:* NUMERIC(6,2) speichert exakt (keine Rundungsfehler), REAL arbeitet mit Gleitkommazahlen, speichert Werte basierend auf dem Binärsystem, nicht im dezimalsystem und kann ungenau sein.
Zum Beispiel:
tagesgebuehr = 0.10
Anzahl Tage = 3

NUMERIC:
0.10 * 3 = 0.30 (exakt)

REAL:
0.10 * 3 = 0.30000000000000004

Der Unterschied wird vor allem bei mehreren Berechnungen oder Summen sichtbar.

---

## 5 – Insert Rows One at a Time

Insert each of the following rows by typing the `INSERT` statement manually and
executing it individually. Watch the `INSERT 0 1` confirmation after each one.

**Books:**

```sql
INSERT INTO buch VALUES ('978-3-423-08733-2', 'Steppenwolf', 1927, 'dtv', 0.50);
```

```sql
INSERT INTO buch VALUES ('978-3-518-36893-4', 'Homo Faber', 1957, 'Suhrkamp', 0.50);
```

```sql
INSERT INTO buch VALUES ('978-3-257-20456-6', 'Der Vorleser', 1995, 'Diogenes', 0.75);
```

```sql
INSERT INTO buch VALUES ('978-3-596-18296-4', 'Das Parfum', 1985, 'Fischer', 0.75);
```

```sql
INSERT INTO buch VALUES ('978-3-423-13571-9', 'Die Verwandlung', 1915, 'dtv', 0.30);
```

**Copies:**

```sql
INSERT INTO exemplar VALUES (1, '978-3-423-08733-2', 'A-01-3');
```

```sql
INSERT INTO exemplar VALUES (2, '978-3-423-08733-2', 'A-01-4');
```

```sql
INSERT INTO exemplar VALUES (3, '978-3-518-36893-4', 'A-02-1');
```

```sql
INSERT INTO exemplar VALUES (4, '978-3-257-20456-6', 'B-01-7');
```

```sql
INSERT INTO exemplar VALUES (5, '978-3-596-18296-4', 'B-02-2');
```

```sql
INSERT INTO exemplar VALUES (6, '978-3-423-13571-9', 'A-03-1');
```

**Members** — omit `mitglied_id` (it is generated) and omit `beitritt_datum`
for the first two members to test the `DEFAULT`:

```sql
INSERT INTO mitglied (nachname, vorname, geburtsdatum, email)
VALUES ('Berger', 'Jonas', '2001-04-12', 'jonas.berger@mail.de');
```

```sql
INSERT INTO mitglied (nachname, vorname, geburtsdatum, email)
VALUES ('Hartmann', 'Lea', '1998-07-08', 'lea.hartmann@example.com');
```

```sql
INSERT INTO mitglied (nachname, vorname, geburtsdatum, email, beitritt_datum)
VALUES ('Sommer', 'Klara', '1985-11-30', 'klara.sommer@web.de', '2019-03-15');
```

After all inserts, verify the row count:

```sql
SELECT COUNT(*) FROM buch;
SELECT COUNT(*) FROM exemplar;
SELECT COUNT(*) FROM mitglied;
```

> **Screenshot 5:** Take a screenshot showing the three `COUNT(*)` results.
>
> ![Screenshot5](https://github.com/Oryanick/DBMS_06/blob/master/Screenshot%205.png)

Exit `psql`:

```sql
\q
```

---

## 6 – Create a CSV File and Load It with COPY

You will now add the lending records (`ausleihe`) not by hand, but by preparing
a CSV file and loading it with the standard SQL `COPY` statement.

### Step 1 – Write the CSV File

Open a new file in the terminal:

```bash
vim ausleihe.csv
```

Enter Insert mode with `i` and type the following content exactly — one row
per line, fields separated by commas, no header row:

```
1,2026-05-01,2026-05-10
2,2026-05-05,
3,2026-05-12,
6,2026-04-20,2026-04-28
```

The columns correspond to: `exemplar_id`, `ausleihe_datum`, `rueckgabe_datum`.
An empty trailing field means `NULL`.

Save and exit: press `Esc`, then type `:wq` and press Enter.

Verify the file:

```bash
cat ausleihe.csv
```

> Note: `mitglied_id` is not in the CSV. You will specify the target columns
> in the `COPY` statement and use a fixed value via a subsequent `UPDATE`, or
> you can extend the CSV to include the member assignment. The approach below
> adds `mitglied_id` to the CSV as a fourth column (member IDs: 1, 2, 1, 3).
> Recreate the file accordingly:

```
1,1,2026-05-01,2026-05-10
3,2,2026-05-05,
4,1,2026-05-12,
6,3,2026-04-20,2026-04-28
```

Columns: `exemplar_id`, `mitglied_id`, `ausleihe_datum`, `rueckgabe_datum`.

### Step 2 – Load the CSV

Connect to the database:

```bash
psql -U <your-username> -d bibliothek
```

Load the file using `COPY`. The path must be absolute:

```sql
COPY ausleihe (exemplar_id, mitglied_id, ausleihe_datum, rueckgabe_datum)
FROM '/home/<your-username>/ausleihe.csv'
WITH (FORMAT csv, NULL '');
```

> The `NULL ''` option tells PostgreSQL to interpret an empty field as `NULL`.

Verify:

```sql
SELECT * FROM ausleihe;
```

> **Screenshot 6:** Take a screenshot showing the full output of `SELECT * FROM ausleihe`.
>
>![Screenshot6](https://github.com/Oryanick/DBMS_06/blob/master/Screenshot%206.png)

### Questions for Section 6

**Question 6.1:** `COPY FROM` requires an absolute path on the server's
filesystem. What is the difference between server-side `COPY` and a
client-side import? In which scenario would you need the client-side variant?

> *Your answer:* COPY FROM liest die Datei direkt auf dem Server. Das bedeutet, der PostgreSQL Server muss Zugriff auf den angegebenen Dateipfad haben. \copy dagegen wird im psql-Client ausgeführt. Dabei liest der Client die Datei lokal ein und schickt die Daten an den Server. Der clientseitigen COPY wird vor allem dann benötigt, wenn man keinen direkten Zugriff auf das Server Dateisystem hat bzw. keine entsprechenden Rechte besitzt.

**Question 6.2:** The `NULL ''` option maps empty CSV fields to `NULL`.
What would happen without this option if the `rueckgabe_datum` field is empty?

> *Your answer:* Ohne die Angabe NULL'' interpretiert PostgreSQL leere Felder aus der CSV nicht automatisch als NULL. Das kann dazu führen, dass leere Werte entweder als leere Strings gespeichert werden oder je nach Spaltentyp zu Importfehlern führen, da das Format nicht eindeutig als NULL erkannt wird.

**Question 6.3:** `ausleihe_id` is `GENERATED ALWAYS AS IDENTITY` and was not
included in the CSV or the `COPY` column list. How does PostgreSQL handle the
missing value? What would happen if you tried to include `ausleihe_id` in the
`COPY` column list with explicit values?

> *Your answer:* Die Spalte ausleihe_id wird automatisch durch PostgreSQL generiert, da sie als GENERATED ALWAYS AS IDENTITY definiert ist. Wird sie beim COPY nicht angegeben, könnte die Datenbank selbstständig passende Werte erzeugen. Wenn man versuchen würde, eigene Werte in diese Spalte zu schreiben, würde PostgreSQL einen Fehler ausgeben, da GENERATED ALWAYS keine manuellen Eingaben erlaubt. Das wäre nur mit OVERRIDING SYSTEM VALUE möglich.

---

## 7 – Three Queries in the psql Shell

Type and execute each query individually in the `psql` shell. Activate
formatted output first:

```sql
\pset format aligned
\pset border 1
```

---

### Query 1 – Currently Open Loans

List all loans that have not yet been returned: show the member's full name
(last name, first name), the book title, and the number of days the book has
been borrowed so far (today minus `ausleihe_datum`).

```sql
SELECT m.nachname,
       m.vorname,
       b.titel,
       CURRENT_DATE - a.ausleihe_datum AS tage_ausgeliehen
FROM   ausleihe   a
JOIN   exemplar   e ON e.exemplar_id = a.exemplar_id
JOIN   buch       b ON b.isbn        = e.isbn
JOIN   mitglied   m ON m.mitglied_id = a.mitglied_id
WHERE  a.rueckgabe_datum IS NULL
ORDER BY tage_ausgeliehen DESC;
```

> *Describe the result: how many open loans are there, and which member has
> held a book the longest?*

> Es gibt 2 offene Ausleihen. Das Mitglied Hartmann, Lea hält das Buch Homo Faber am längsten ausgeliehen (45 Tage), während Berger, Jonas das Buch Der Vorleser seit 38 Tagen ausgeliehen hat.

---

### Query 2 – Loans per Member

For each member, show their full name, the total number of loans, and the
number of loans that are still open. Sort descending by total loans.

```sql
SELECT m.nachname,
       m.vorname,
       COUNT(*)                                          AS ausleihen_gesamt,
       COUNT(*) FILTER (WHERE a.rueckgabe_datum IS NULL) AS noch_offen
FROM   mitglied m
JOIN   ausleihe a ON a.mitglied_id = m.mitglied_id
GROUP BY m.mitglied_id, m.nachname, m.vorname
ORDER BY ausleihen_gesamt DESC;
```

> *Which member has the most loans? What does `FILTER (WHERE ...)` do here
> compared to a `CASE WHEN` expression?*

Das Mitglied Berger, Jonas hat die meisten Kredite mit insgesamt 2 Ausleihen.
- FILTER (WHERE ...) bedeutet: Es wird nur für diese Bedingung gezählt. FILTER ist lesbarer,kompakter, nah am SQL-standard und wird direkt in Aggregatfunktion integriert. 
- CASE WHEN müsste man manuell als Ausdruck bauen ZB: SUM(CASE WHEN rueckgabe_datum IS NULL THEN 1 ELSE 0 END)


---

### Query 3 – Books That Have Never Been Borrowed

Return the title and publisher of every book for which no lending record
exists anywhere in `ausleihe` (regardless of return date).

```sql
SELECT b.titel,
       b.verlag
FROM   buch b
WHERE  NOT EXISTS (
    SELECT 1
    FROM   exemplar e
    JOIN   ausleihe a ON a.exemplar_id = e.exemplar_id
    WHERE  e.isbn = b.isbn
);
```

> *Which books appear in the result? Verify the result manually against the
> data you entered.*

Das Buch Das Parfum(Fischer) erscheint im Ergebnis. Dieses Buch wurde zwar in buch angelegt, aber es existiert kein Eintrag in ausleihe, der auf dieses Buch verweist. Alle anderen Bücher wurden mindestens einmal ausgeliehen.

> **Screenshot 7:** Take a screenshot showing the output of all three queries
> in sequence in the `psql` shell.
>
> ![Screenshot7](https://github.com/Oryanick/DBMS_06/blob/master/Screenshot%207.png)

### Questions for Section 7

**Question 7.1:** Query 1 joins four tables. In what order must the joins be
performed to always produce a correct result, and does the join order affect
correctness or only performance?

> *Your answer:* Die Verknüpfungen folgen logisch den Fremdschlüsselbeziehungen in der Datenbank. Dabei ergibt sich die Kette aus den Tabellenbeziehungen: ausleihe → exemplar → buch → mitglied
Das bedeutet konkret: aus der Ausleihe wird über exemplar_id auf exemplar zugegriffen, von dort über isbn auf buch und über mitglied_id auf mitglied. Für die Korrektheit des Ergebnisses ist die Reihenfolge der JOINs jedoch nicht entscheidend, solange alle JOIN Bedingungen korrekt angegeben sind. SQL arbeitet deklarativ, das heißt, es beschreibt nur das gewünschte Ergebnis und nicht die genaue Ausführungsreihenfolge. Die tatsächliche Reihenfolge wird vom Query Optimizer bestimmt und kann sich je nach Ausführungsplan ändern. Der Unterschied liegt daher nur in der Performance, nicht in der Korrektheit.

**Question 7.2:** Query 2 groups by `m.mitglied_id` in addition to the name
columns. Why is grouping by the primary key necessary even though names appear
unique in the sample data?

> *Your answer:* Die Gruppierung nach m.mitglied_id ist notwendig, weil SQL verlangt, dass alle nicht aggregierten Spalten entweder im GROUP BY enthalten sind oder funktional davon abhängen. Auch wenn die Namen im Beispiel eindeutig erscheinen, kann SQL das nicht voraussetzen. Es ist möglich, dass mehrere Mitglieder denselben Vornamen oder Nachnamen haben. Nur die mitglied_id ist garantiert eindeutig und eindeutig einem Datensatz zugeordnet. Durch die Gruppierung nach dem Primärschlüssel wird sichergestellt, dass jede Aggregation eindeutig pro Mitglied erfolgt und das Ergebnis unabhängig von Datenannahmen korrekt bleibt.

**Question 7.3:** Query 3 uses `NOT EXISTS` with a correlated subquery. Rewrite
the query using `EXCEPT` and verify that both variants return the same result.
Write your rewritten query here:

> *Your rewritten query:*
```sql
SELECT b.titel, b.verlag
FROM buch b

EXCEPT

SELECT b.titel, b.verlag
FROM buch b
JOIN exemplar e ON e.isbn = b.isbn
JOIN ausleihe a ON a.exemplar_id = e.exemplar_id
```

Exit `psql`:

```sql
\q
```

---

## 8 – A Second Database from a Script

You will now create a new, independent database for a different domain: a
small **cinema programme** that stores films, screenings, and seat reservations.

### Step 1 – Create the Script File

```bash
vim kino.sql
```

Enter the following content:

```sql
-- Create the database (run this outside a transaction, directly in psql)
-- The database is created manually; this script sets up the schema and data.

CREATE TABLE film (
    film_id     INTEGER      PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    titel       TEXT         NOT NULL,
    laenge_min  INTEGER      NOT NULL CHECK (laenge_min > 0),
    fsk         INTEGER      NOT NULL CHECK (fsk IN (0, 6, 12, 16, 18))
);

CREATE TABLE vorstellung (
    vorstellung_id  INTEGER   PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    film_id         INTEGER   NOT NULL,
    beginn          TIMESTAMP NOT NULL,
    saal            TEXT      NOT NULL,
    FOREIGN KEY (film_id) REFERENCES film(film_id)
        ON DELETE RESTRICT ON UPDATE CASCADE
);

CREATE TABLE reservierung (
    reservierung_id  INTEGER  PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    vorstellung_id   INTEGER  NOT NULL,
    sitzplatz        TEXT     NOT NULL,
    name_gast        TEXT     NOT NULL,
    FOREIGN KEY (vorstellung_id) REFERENCES vorstellung(vorstellung_id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    UNIQUE (vorstellung_id, sitzplatz)
);

-- Sample data
INSERT INTO film (titel, laenge_min, fsk) VALUES
    ('Metropolis',         153, 0),
    ('Das Boot',           149, 16),
    ('Lola rennt',          81, 12),
    ('Der Untergang',      156, 12),
    ('Goodbye Lenin!',     121, 6);

INSERT INTO vorstellung (film_id, beginn, saal) VALUES
    (1, '2026-07-01 20:00', 'Saal 1'),
    (2, '2026-07-01 18:00', 'Saal 2'),
    (3, '2026-07-02 21:00', 'Saal 1'),
    (4, '2026-07-02 19:30', 'Saal 3'),
    (5, '2026-07-03 17:00', 'Saal 2');

INSERT INTO reservierung (vorstellung_id, sitzplatz, name_gast) VALUES
    (1, 'A1', 'Mertens, Paul'),
    (1, 'A2', 'Mertens, Paul'),
    (1, 'B5', 'Fischer, Ruth'),
    (2, 'C3', 'Wagner, Erik'),
    (3, 'A1', 'Schulze, Lena'),
    (3, 'D7', 'Schulze, Lena'),
    (4, 'B2', 'Braun, Otto'),
    (5, 'A3', 'Klein, Marie');
```

Save and exit: `Esc`, `:wq`, Enter.

### Step 2 – Create the Database

```bash
sudo -u postgres psql -c "CREATE DATABASE kino OWNER <your-username>;"
```

### Step 3 – Run the Script

```bash
psql -U <your-username> -d kino -f kino.sql
```

> You should see a sequence of `CREATE TABLE` and `INSERT` confirmations.

> **Screenshot 8:** Take a screenshot showing the script execution output.
>
> ![Screenshot8](https://github.com/Oryanick/DBMS_06/blob/master/Screenshot%208.png)

---

## 9 – Inspect the New Database

Connect to the `kino` database:

```bash
psql -U <your-username> -d kino
```

List all tables:

```sql
\dt
```

Inspect the `reservierung` table structure:

```sql
\d reservierung
```

Examine the data:

```sql
SELECT * FROM film;
```

```sql
SELECT v.vorstellung_id,
       f.titel,
       v.beginn,
       v.saal
FROM   vorstellung v
JOIN   film        f ON f.film_id = v.film_id
ORDER BY v.beginn;
```

```sql
SELECT f.titel,
       COUNT(r.reservierung_id) AS reservierungen
FROM   film         f
JOIN   vorstellung  v ON v.film_id         = f.film_id
LEFT JOIN reservierung r ON r.vorstellung_id = v.vorstellung_id
GROUP BY f.film_id, f.titel
ORDER BY reservierungen DESC;
```

> **Screenshot 9:** Take a screenshot showing the output of all three
> `SELECT` statements.
>
> ![Screenshot9](https://github.com/Oryanick/DBMS_06/blob/master/Screenshot%209.png)

### Questions for Section 9

**Question 9.1:** The `reservierung` table has a `UNIQUE (vorstellung_id, sitzplatz)`
constraint. What does this prevent, and at which level is this constraint
enforced — application or database?

> *Your answer:* Die Einschränkung UNIQUE (vorstellung_id, sitzplatz) sorgt dafür, dass ein bestimmter Sitzplatz innerhalb einer konkreten Vorstellung nur einmal vergeben werden kann. Das heißt konkret: Ein Platz darf in derselben Vorstellung nicht doppelt reserviert werden, in anderen Vorstellungen aber schon. Diese Regel wird direkt auf Datenbankebene durchgesetzt und nicht in der Anwendung. Dadurch kann keine Software oder kein Benutzer sie umgehen. Selbst bei mehreren gleichzeitigen Zugriffen bleibt die Datenintegrität erhalten, weil die Datenbank das zentral absichert.

**Question 9.2:** The third query uses `LEFT JOIN` between `vorstellung` and
`reservierung`. What would be different about the result if you used `JOIN`
(inner join) instead? Which films would disappear from the result and why?

> *Your answer:* Der Unterschied liegt darin, wie mit fehlenden Daten umgegangen wird. Ein LEFT JOIN gibt alle Filme zurück, auch wenn sie keine Reservierungen haben. In diesem Fall erscheinen die Reservierungsdaten dann als NULL. Ein INNER JOIN zeigt dagegen nur Filme, zu denen es mindestens eine Reservierung gibt. Filme ohne Buchungen werden komplett ausgeblendet. Das bedeutet konkret: Bei einem INNER JOIN würden alle Filme ohne Reservierungen aus dem Ergebnis verschwinden, also auch solche, die zwar existieren, aber noch nicht gebucht wurden. Man kann sagen, dass LEFT JOIN alles zeigt, INNER JOIN aber nur die aktiven Datensätze mit Beziehung.

**Question 9.3:** `ON DELETE CASCADE` was chosen for `reservierung.vorstellung_id`,
but `ON DELETE RESTRICT` for `vorstellung.film_id`. Justify both choices in
terms of the domain.

> *Your answer:* ON DELETE CASCADE bei reservierung.vorstellung_id bedeutet, dass alle Reservierungen automatisch gelöscht werden, wenn eine Vorstellung entfernt wird. Dadurch bleiben keine verwaisten Datensätze übrig, die auf nichts mehr zeigen würden. ON DELETE RESTRICT bei vorstellung.film_id verhindert dagegen, dass ein Film gelöscht wird, solange noch Vorstellungen existieren. Das schützt historische und abhängige Daten davor, versehentlich verloren zu gehen. Insgesamt sorgt CASCADE eher für automatische Aufräumlogik, während RESTRICT als Schutzmechanismus für zentrale Daten fungiert.

Exit `psql`:

```sql
\q
```

---

## 10 – Reflection

**Question A – Server vs. embedded database:**  
SQLite (DBMS_05) and PostgreSQL (this exercise) are both relational databases,
but they operate very differently. Name two concrete differences you experienced
in this exercise — in terms of setup, access control, or SQL behaviour.

> *Your answer:* Zwei zentrale Unterschiede aus der Praxis sind die Architektur und die Benutzerverwaltung. SQLite ist eine eingebettete Datenbank. Das bedeutet, sie läuft direkt in der Anwendung und benötigt keinen separaten Server. Es ist keine Installation eines Datenbankservers notwendig. PostgreSQL hingegen ist ein Client-Server-System. Dabei läuft der Datenbankserver im Hintergrund und wird separat betrieben. Der Zugriff erfolgt entweder lokal über localhost oder über ein Netzwerk. Ein weiterer wichtiger Unterschied ist die Zugriffskontrolle. SQLite besitzt keine echte Benutzer- oder Rollenverwaltung. Es gibt keine Rechteverwaltung auf Datenbankebene. PostgreSQL hingegen arbeitet mit Rollen und Benutzerrechten. Man kann Benutzer mit LOGIN-Rechten erstellen, Eigentümer festlegen und Zugriffe gezielt steuern. Außerdem können mehrere Benutzer gleichzeitig sauber verwaltet werden.

**Question B – COPY vs. INSERT:**  
You inserted the `buch` and `exemplar` rows one at a time, and the `ausleihe`
rows via `COPY`. For a real import of 50,000 rows, which approach would you
choose and why? What is the main operational cost of individual `INSERT`
statements at scale?

> *Your answer:* Für große Datenmengen wie 50.000 Zeilen würde man eindeutig COPY verwenden. Der Hauptgrund ist die deutlich bessere Performance. COPY ist speziell für Massendatenimporte optimiert und verursacht deutlich weniger Overhead pro Datensatz. Bei INSERT sieht das anders aus, jede Zeile wird einzeln verarbeitet, was viele einzelne Operationen und Transaktionen erzeugt. Dadurch entsteht ein hoher Overhead und der Import wird bei großen Datenmengen deutlich langsamer.

**Question C – Role model:**  
You created a dedicated role with `LOGIN` and a password. The `postgres`
superuser also exists. What is the security principle behind creating a
separate role instead of always connecting as `postgres`?

> *Your answer:* Der wichtigste Sicherheitsaspekt ist das sogenannte Least-Privilege-Prinzip. Das bedeutet, jeder Benutzer bekommt nur die Rechte, die er wirklich benötigt. Das hat mehrere Vorteile. Zum einen werden versehentliche Änderungen an wichtigen Systemdaten verhindert. Zum anderen gibt es eine klare Trennung zwischen Administrator (postgres Superuser) und normalen Benutzern. Dadurch wird die Kontrolle über die Datenbank deutlich sicherer und strukturierter. Der postgres Superuser sollte daher nicht im normalen Arbeitsalltag verwendet werden, da er vollständige Rechte über das gesamte System besitzt.

**Question D – Script-driven setup:**  
The `kino.sql` script creates the schema and inserts data in one run. What
is the advantage of this approach over typing the statements interactively?
Name one situation where an interactive approach is still preferable.

> *Your answer:* Der größte Vorteil von Skripten wie kino.sql ist die Reproduzierbarkeit. Das bedeutet, dass die Datenbank jederzeit exakt gleich aufgebaut werden kann. Außerdem sind Skripte schnell ausführbar, gut versionierbar und ideal für automatisierte Deployments. Die interaktive Nutzung hat dagegen andere Vorteile. Sie eignet sich besonders gut zum Debuggen, zum schrittweisen Testen von SQL Befehlen und allgemein zum Lernen und Experimentieren. Es lässt sich kurz sagen, dass Skripte ideal für produktive und wiederholbare Setups sind, während die interaktive Nutzung für Entwicklung und Verständnis besser geeignet ist.

---

## Further Reading

- [PostgreSQL 16 – `CREATE ROLE`](https://www.postgresql.org/docs/current/sql-createrole.html)
- [PostgreSQL 16 – `COPY`](https://www.postgresql.org/docs/current/sql-copy.html)
- [PostgreSQL 16 – `psql` Reference](https://www.postgresql.org/docs/current/app-psql.html)
- [PostgreSQL 16 – Data Types](https://www.postgresql.org/docs/current/datatype.html)
- [PostgreSQL 16 – Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
- Lecture 06 handout
