# Next.js App Router API & Component Patterns

## Collection Route — src/app/api/<entity>/route.ts

```typescript
import { NextResponse } from 'next/server';
import { initDb, getAllActivities, createActivity, type NewActivity } from '@/lib/db';

export async function GET() {
  try {
    const db = initDb();
    const data = getAllActivities(db);
    return NextResponse.json({ data });
  } catch {
    return NextResponse.json({ error: 'Failed to fetch activities' }, { status: 500 });
  }
}

export async function POST(request: Request) {
  try {
    const body = await request.json() as Partial<NewActivity>;
    const { student_id, type, date, hours } = body;

    if (!student_id || !type || !date || !hours) {
      return NextResponse.json(
        { error: 'student_id, type, date, and hours are required' },
        { status: 400 }
      );
    }

    const db = initDb();
    const data = createActivity(db, {
      student_id,
      type,
      date,
      hours,
      description: body.description ?? null,
      supervisor: body.supervisor ?? null,
      status: 'pending',
    });
    return NextResponse.json({ data }, { status: 201 });
  } catch {
    return NextResponse.json({ error: 'Failed to create activity' }, { status: 500 });
  }
}
```

---

## Item Route — src/app/api/<entity>/[id]/route.ts

```typescript
import { NextResponse } from 'next/server';
import {
  initDb,
  getActivityById,
  updateActivity,
  deleteActivity,
  type UpdateActivity,
} from '@/lib/db';

type RouteContext = { params: Promise<{ id: string }> };

export async function GET(_req: Request, { params }: RouteContext) {
  const { id } = await params;
  const db = initDb();
  const data = getActivityById(db, Number(id));
  if (!data) return NextResponse.json({ error: 'Not found' }, { status: 404 });
  return NextResponse.json({ data });
}

export async function PUT(request: Request, { params }: RouteContext) {
  try {
    const { id } = await params;
    const body = await request.json() as UpdateActivity;
    const db = initDb();
    const data = updateActivity(db, Number(id), body);
    if (!data) return NextResponse.json({ error: 'Not found' }, { status: 404 });
    return NextResponse.json({ data });
  } catch {
    return NextResponse.json({ error: 'Failed to update activity' }, { status: 500 });
  }
}

// PATCH — same shape as PUT, kept separate for semantic clarity
export async function PATCH(request: Request, { params }: RouteContext) {
  return PUT(request, { params });
}

export async function DELETE(_req: Request, { params }: RouteContext) {
  const { id } = await params;
  const db = initDb();
  const deleted = deleteActivity(db, Number(id));
  if (!deleted) return NextResponse.json({ error: 'Not found' }, { status: 404 });
  return new NextResponse(null, { status: 204 });
}
```

---

## Client-Side CRUD Hook — src/hooks/use<Entity>.ts

```typescript
'use client';

import { useState, useEffect, useCallback } from 'react';
import type { Activity, NewActivity, UpdateActivity } from '@/lib/db';

interface UseCrudResult {
  items: Activity[];
  loading: boolean;
  error: string | null;
  create: (data: NewActivity) => Promise<void>;
  update: (id: number, data: UpdateActivity) => Promise<void>;
  remove: (id: number) => Promise<void>;
  reload: () => Promise<void>;
}

export function useActivities(): UseCrudResult {
  const [items, setItems] = useState<Activity[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const reload = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const res = await fetch('/api/activities');
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const { data } = await res.json() as { data: Activity[] };
      setItems(data);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load');
    } finally {
      setLoading(false);
    }
  }, []);

  const create = useCallback(async (data: NewActivity) => {
    const res = await fetch('/api/activities', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    if (!res.ok) {
      const { error } = await res.json() as { error: string };
      throw new Error(error);
    }
    await reload();
  }, [reload]);

  const update = useCallback(async (id: number, data: UpdateActivity) => {
    const res = await fetch(`/api/activities/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    if (!res.ok) {
      const { error } = await res.json() as { error: string };
      throw new Error(error);
    }
    await reload();
  }, [reload]);

  const remove = useCallback(async (id: number) => {
    const res = await fetch(`/api/activities/${id}`, { method: 'DELETE' });
    if (!res.ok && res.status !== 404) throw new Error(`HTTP ${res.status}`);
    await reload();
  }, [reload]);

  useEffect(() => { void reload(); }, [reload]);

  return { items, loading, error, create, update, remove, reload };
}
```

---

## Form Component Pattern — src/components/<Entity>Form.tsx

```tsx
'use client';

import { useState, type FormEvent } from 'react';
import type { NewActivity } from '@/lib/db';

interface Props {
  onSubmit: (data: NewActivity) => Promise<void>;
  onCancel?: () => void;
}

export function ActivityForm({ onSubmit, onCancel }: Props) {
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setSubmitting(true);
    setError(null);

    const fd = new FormData(e.currentTarget);
    try {
      await onSubmit({
        student_id: Number(fd.get('student_id')),
        type: fd.get('type') as string,
        date: fd.get('date') as string,
        hours: Number(fd.get('hours')),
        description: (fd.get('description') as string) || null,
        supervisor: (fd.get('supervisor') as string) || null,
        status: 'pending',
      });
      e.currentTarget.reset();
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Submission failed');
    } finally {
      setSubmitting(false);
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      {error && <p role="alert" aria-live="assertive">{error}</p>}
      {/* fields go here — copy from original HTML */}
      <button type="submit" disabled={submitting} aria-busy={submitting}>
        {submitting ? 'Submitting…' : 'Submit'}
      </button>
      {onCancel && <button type="button" onClick={onCancel}>Cancel</button>}
    </form>
  );
}
```

---

## Preserving Original HTML Styles

Copy `<style>` blocks from the HTML verbatim into `src/app/globals.css`, after the Tailwind directives:

```css
/* src/app/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* ── Pasted from original HTML ─────────────────────────────────── */
/* paste original <style> block content here */
```

For class-heavy HTML, keep all original `className` values on the JSX elements — don't swap them for Tailwind utilities.
