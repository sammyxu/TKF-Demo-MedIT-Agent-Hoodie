# SQLite Patterns (better-sqlite3)

## Singleton db.ts — Full Template

```typescript
// src/lib/db.ts
import Database from 'better-sqlite3';
import path from 'path';

const DB_PATH = path.join(process.cwd(), 'data', 'app.db');

declare global {
  // eslint-disable-next-line no-var
  var __db: Database.Database | undefined;
}

function getDb(): Database.Database {
  if (!global.__db) {
    global.__db = new Database(DB_PATH);
    global.__db.pragma('journal_mode = WAL');
    global.__db.pragma('foreign_keys = ON');
    createTables(global.__db);
  }
  return global.__db;
}

export function initDb(): Database.Database {
  return getDb();
}

function createTables(db: Database.Database) {
  db.exec(`
    CREATE TABLE IF NOT EXISTS students (
      id         INTEGER PRIMARY KEY AUTOINCREMENT,
      name       TEXT    NOT NULL,
      email      TEXT    NOT NULL UNIQUE,
      role       TEXT    NOT NULL DEFAULT 'student'
                         CHECK(role IN ('student','admin')),
      created_at TEXT    NOT NULL DEFAULT (datetime('now'))
    );

    CREATE TABLE IF NOT EXISTS activities (
      id          INTEGER PRIMARY KEY AUTOINCREMENT,
      student_id  INTEGER NOT NULL REFERENCES students(id) ON DELETE CASCADE,
      type        TEXT    NOT NULL,
      date        TEXT    NOT NULL,
      hours       REAL    NOT NULL CHECK(hours > 0),
      description TEXT,
      supervisor  TEXT,
      status      TEXT    NOT NULL DEFAULT 'pending'
                          CHECK(status IN ('pending','approved','rejected')),
      created_at  TEXT    NOT NULL DEFAULT (datetime('now'))
    );

    CREATE INDEX IF NOT EXISTS idx_activities_student ON activities(student_id);
    CREATE INDEX IF NOT EXISTS idx_activities_status  ON activities(status);
  `);
}
```

---

## Typed Interfaces

Define one interface per entity. Mirror column names exactly:

```typescript
export interface Student {
  id: number;
  name: string;
  email: string;
  role: 'student' | 'admin';
  created_at: string;
}

export interface Activity {
  id: number;
  student_id: number;
  type: string;
  date: string;
  hours: number;
  description: string | null;
  supervisor: string | null;
  status: 'pending' | 'approved' | 'rejected';
  created_at: string;
}

// Input type for create — omit auto-generated fields
export type NewActivity = Omit<Activity, 'id' | 'created_at'>;
export type UpdateActivity = Partial<Omit<Activity, 'id' | 'created_at'>>;
```

---

## CRUD Query Helpers

```typescript
// ── Read ──────────────────────────────────────────────────────────────────

export function getAllActivities(db: Database.Database): Activity[] {
  return db
    .prepare('SELECT * FROM activities ORDER BY created_at DESC')
    .all() as Activity[];
}

export function getActivitiesByStudent(
  db: Database.Database,
  studentId: number
): Activity[] {
  return db
    .prepare('SELECT * FROM activities WHERE student_id = ? ORDER BY created_at DESC')
    .all(studentId) as Activity[];
}

export function getActivityById(
  db: Database.Database,
  id: number
): Activity | undefined {
  return db
    .prepare('SELECT * FROM activities WHERE id = ?')
    .get(id) as Activity | undefined;
}

// ── Create ────────────────────────────────────────────────────────────────

export function createActivity(
  db: Database.Database,
  data: NewActivity
): Activity {
  const stmt = db.prepare(`
    INSERT INTO activities (student_id, type, date, hours, description, supervisor, status)
    VALUES (@student_id, @type, @date, @hours, @description, @supervisor, @status)
  `);
  const result = stmt.run(data);
  return getActivityById(db, result.lastInsertRowid as number)!;
}

// ── Update ────────────────────────────────────────────────────────────────

export function updateActivity(
  db: Database.Database,
  id: number,
  data: UpdateActivity
): Activity | undefined {
  if (Object.keys(data).length === 0) return getActivityById(db, id);
  const setClause = Object.keys(data)
    .map((k) => `${k} = @${k}`)
    .join(', ');
  db.prepare(`UPDATE activities SET ${setClause} WHERE id = @id`).run({ ...data, id });
  return getActivityById(db, id);
}

// ── Delete ────────────────────────────────────────────────────────────────

export function deleteActivity(db: Database.Database, id: number): boolean {
  const result = db.prepare('DELETE FROM activities WHERE id = ?').run(id);
  return result.changes > 0;
}
```

---

## Transactions (for multi-step operations)

```typescript
export function approveActivity(db: Database.Database, id: number, points: number) {
  const tx = db.transaction(() => {
    db.prepare(`UPDATE activities SET status = 'approved' WHERE id = ?`).run(id);
    db.prepare(`
      INSERT INTO points_ledger (student_id, points, source_activity_id)
      SELECT student_id, ?, id FROM activities WHERE id = ?
    `).run(points, id);
  });
  tx();
}
```

---

## Seeding with INSERT OR IGNORE

```typescript
// src/lib/seed.ts
import { initDb } from './db';

const db = initDb();

const seedStudents = db.transaction(() => {
  db.prepare(`
    INSERT OR IGNORE INTO students (id, name, email) VALUES (1, 'Alice Chen', 'alice@example.com')
  `).run();
});

seedStudents();
console.log('Seed complete.');
```

Add to `package.json`:
```json
"scripts": {
  "seed": "npx tsx src/lib/seed.ts"
}
```
