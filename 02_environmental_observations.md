# Environmental Intelligence in Meliponiculture SaaS: Modeling Contextual Observations with Prisma and Next.js

When building software for **meliponiculture** (stingless bee farming), data goes far beyond hive inspections and honey production. Native bees are deeply sensitive to environmental conditions — weather changes, swarming events, pollen availability, and predator activity directly impact colony health and productivity.

In this article, I'll walk through how we designed and implemented the **Environmental Observations Module** in [ASFControl](https://asfcontrol.com.br), a SaaS platform for managing native stingless bee colonies in Brazil.

---

## The Problem: Context-Blind Apiary Records

Before this module, ASFControl could track *what happened* inside a hive (inspections, harvests, feedings), but had no way to record *why it happened*. Producers often noted observations like "colony seemed stressed" or "unusual swarming activity" in plain text fields, making data analysis impossible.

We needed a structured way to capture four categories of environmental events:

| Category | Examples |
|---|---|
| **Climate** | Temperature, humidity, rainfall, wind speed |
| **Swarming** | Natural swarm events, swarm capture attempts |
| **Pollen** | Flowering plants in bloom, pollen color, abundance |
| **Predators** | Ants, lizards, wax moths, fire incidents |

---

## Database Design

Each observation category has its own dedicated table in PostgreSQL, linked to the `User` entity. This approach avoids a monolithic "observations" table with 30 nullable columns, preserving type safety in Prisma.

```prisma
// prisma/schema.prisma

model ObservacaoClima {
  id          String   @id @default(cuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  data        DateTime
  temperatura Float?
  umidade     Float?
  chuva       Boolean  @default(false)
  descricao   String?
  createdAt   DateTime @default(now())

  @@index([userId])
}

model ObservacaoEnxameacao {
  id           String   @id @default(cuid())
  userId       String
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  data         DateTime
  colmeiaId    String?
  colmeia      Colmeia? @relation(fields: [colmeiaId], references: [id], onDelete: SetNull)
  tipo         String   // "natural" | "captura_tentada" | "captura_sucesso"
  observacoes  String?
  createdAt    DateTime @default(now())

  @@index([userId])
}

model ObservacaoPolen {
  id          String   @id @default(cuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  data        DateTime
  planta      String
  corPolen    String?
  abundancia  String?  // "baixa" | "media" | "alta"
  observacoes String?
  createdAt   DateTime @default(now())

  @@index([userId])
}

model ObservacaoPredador {
  id          String   @id @default(cuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  data        DateTime
  tipo        String   // "formiga" | "lagartixa" | "traça-da-cera" | "incêndio" | "outro"
  gravidade   String?  // "leve" | "moderada" | "grave"
  colmeiaId   String?
  observacoes String?
  createdAt   DateTime @default(now())

  @@index([userId])
}
```

Using `@@index([userId])` ensures efficient queries when filtering observations per user, which is critical in a multi-tenant SaaS.

---

## Weather API Integration

For climate observations, we integrated a real-time weather API to pre-populate the form with current conditions, reducing manual data entry. The integration lives in a dedicated API route:

```typescript
// src/app/api/weather/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getUserFromToken } from '@/lib/auth';

export async function GET(req: NextRequest) {
    const user = await getUserFromToken(req);
    if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

    const { searchParams } = new URL(req.url);
    const lat = searchParams.get('lat');
    const lon = searchParams.get('lon');

    if (!lat || !lon) {
        return NextResponse.json({ error: 'Coordinates required' }, { status: 400 });
    }

    const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m,relative_humidity_2m,precipitation,wind_speed_10m&timezone=America/Sao_Paulo`;

    const res = await fetch(url, { next: { revalidate: 600 } }); // Cache for 10 minutes
    if (!res.ok) {
        return NextResponse.json({ error: 'Weather API unavailable' }, { status: 502 });
    }

    const data = await res.json();
    const current = data.current;

    return NextResponse.json({
        temperatura: current.temperature_2m,
        umidade: current.relative_humidity_2m,
        chuva: current.precipitation > 0,
        vento: current.wind_speed_10m,
    });
}
```

We chose [Open-Meteo](https://open-meteo.com) for its free tier, no API key requirement, and excellent coverage across Brazil. The `revalidate: 600` option uses Next.js's built-in fetch cache to avoid hammering the weather API on rapid refreshes.

---

## Component Architecture: One Form per Observation Category

Rather than building a single mega-form with conditional logic, we created **one focused component per category**. This keeps each form simple, testable, and independently maintainable.

```
src/components/observacoes/
├── ClimaPicker.tsx          ← Reusable weather pre-fill component
├── ObservacaoCard.tsx       ← Generic card for listing any observation type
├── ObservacaoList.tsx       ← Unified list with category filter
├── clima/
│   └── ClimaForm.tsx        ← Climate observation form
├── enxameacao/
│   └── EnxameacaoForm.tsx   ← Swarming event form
├── polen/
│   └── PolenForm.tsx        ← Pollen observation form
└── predadores/
    └── PredadorForm.tsx     ← Predator activity form
```

The `ClimaPicker` component handles geolocation and weather pre-filling:

```tsx
// src/components/observacoes/ClimaPicker.tsx
"use client";

import { useState } from "react";

interface WeatherData {
    temperatura: number;
    umidade: number;
    chuva: boolean;
}

interface ClimaPicker {
    onWeatherLoaded: (data: WeatherData) => void;
}

export function ClimaPicker({ onWeatherLoaded }: ClimaPicker) {
    const [loading, setLoading] = useState(false);

    const fetchWeather = () => {
        if (!navigator.geolocation) return;
        setLoading(true);

        navigator.geolocation.getCurrentPosition(async ({ coords }) => {
            const res = await fetch(
                `/api/weather?lat=${coords.latitude}&lon=${coords.longitude}`
            );
            if (res.ok) {
                const data = await res.json();
                onWeatherLoaded(data);
            }
            setLoading(false);
        });
    };

    return (
        <button
            type="button"
            onClick={fetchWeather}
            disabled={loading}
            aria-label="Preencher condições climáticas atuais"
        >
            {loading ? "Carregando clima..." : "🌤 Usar clima atual"}
        </button>
    );
}
```

When the user taps "Usar clima atual", the component requests geolocation, fetches current conditions from our `/api/weather` proxy, and calls `onWeatherLoaded` to pre-fill the form fields. This reduces friction and ensures climate records are accurate.

---

## API Route Pattern

All four observation categories follow the same RESTful pattern:

```typescript
// src/app/api/observacoes/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/db';
import { getUserFromToken } from '@/lib/auth';

export async function GET(req: NextRequest) {
    const user = await getUserFromToken(req);
    if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

    const { searchParams } = new URL(req.url);
    const tipo = searchParams.get('tipo') ?? 'clima';
    const page = Number(searchParams.get('page') ?? 1);
    const limit = 20;

    const modelMap = {
        clima: prisma.observacaoClima,
        enxameacao: prisma.observacaoEnxameacao,
        polen: prisma.observacaoPolen,
        predadores: prisma.observacaoPredador,
    } as const;

    const model = modelMap[tipo as keyof typeof modelMap];
    if (!model) {
        return NextResponse.json({ error: 'Invalid observation type' }, { status: 400 });
    }

    const [items, total] = await Promise.all([
        (model as any).findMany({
            where: { userId: user.id },
            orderBy: { data: 'desc' },
            skip: (page - 1) * limit,
            take: limit,
        }),
        (model as any).count({ where: { userId: user.id } }),
    ]);

    return NextResponse.json({ items, total, page, totalPages: Math.ceil(total / limit) });
}
```

Using a `modelMap` approach keeps the route DRY while maintaining full Prisma type safety per model.

---

## Outcome

The Environmental Observations module transforms ASFControl from a simple hive manager into an **environmental intelligence platform**. Producers can now correlate hive performance with external factors like weather patterns, seasonal pollen availability, and predator activity — generating the kind of dataset that supports both better farm decisions and scientific research on native bee conservation.

---

## Stack

- **Next.js 15** (App Router, Route Handlers, Server Components)
- **Prisma ORM** + **PostgreSQL 15**
- **Open-Meteo API** (free, no key required)
- **TypeScript** (strict mode throughout)
- **Geolocation API** (native browser)

---

*Built as part of [ASFControl](https://asfcontrol.com.br) — the digital infrastructure for Brazilian meliponiculture.*
