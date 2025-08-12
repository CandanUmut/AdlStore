# AdlStore
New App Store for justice and fairness 
# ─────────────────────────────────────────────────────────────────────────────
# AdilStore Monorepo — v0.1 (MVP Scaffold)
# ─────────────────────────────────────────────────────────────────────────────
# This single document contains a minimal working scaffold for:
# - backend/api (FastAPI)
# - database (Supabase/Postgres schema)
# - developer CLI (manifest validation)
# - example manifest (JSON)
# - Flutter client skeleton (store app that lists apps)
# Copy files to a repo keeping the indicated paths.
# ─────────────────────────────────────────────────────────────────────────────

# ============================================================================
# FILE: README.md
# PATH: ./README.md
# ----------------------------------------------------------------------------
README_MD = r"""
# AdilStore (MVP)

A community-first, ad-free, developer-friendly app store. Cloud-first MVP, future decentralized (IPFS/DHT). PRU-DB-ready semantic search.

## Quickstart

### 1) Backend API
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r backend/requirements.txt
cp backend/.env.example backend/.env
# fill SUPABASE_URL and SUPABASE_ANON/SERVICE KEYS
uvicorn backend.api.main:app --reload
```

### 2) Database (Supabase)
- Create a new Supabase project
- Run SQL from `database/schema.sql` in the SQL editor
- Put keys into `backend/.env`

### 3) Flutter Store Client
```bash
cd apps/store_client
flutter pub get
flutter run -d chrome  # or an Android device
```

## Structure
```
backend/
  api/
    main.py
    models.py
    deps.py
  requirements.txt
  .env.example

database/
  schema.sql

apps/
  store_client/
    pubspec.yaml
    lib/main.dart

devtools/
  cli.py

manifests/
  example.app.json
```

## MVP Endpoints
- `GET /health` — OK
- `GET /apps` — list catalog
- `GET /apps/{app_id}` — app detail
- `POST /ingest/manifest` — validate + upsert app
- `GET /search?q=...` — basic search (PRU stub)

## Notes
- Security: APK signature/hash fields recorded; binary not stored in DB (use Supabase Storage/externals).
- Privacy-first: analytics toggled off by default.
- Transparent ranking: to be implemented in v0.2 per spec.
"""

# ============================================================================
# FILE: backend/requirements.txt
# PATH: ./backend/requirements.txt
# ----------------------------------------------------------------------------
REQUIREMENTS_TXT = r"""
fastapi==0.111.0
uvicorn==0.30.0
pydantic==2.8.2
python-dotenv==1.0.1
supabase==2.5.1
httpx==0.27.0
orjson==3.10.6
"""

# ============================================================================
# FILE: backend/.env.example
# PATH: ./backend/.env.example
# ----------------------------------------------------------------------------
ENV_EXAMPLE = r"""
SUPABASE_URL=https://YOUR-PROJECT.supabase.co
SUPABASE_SERVICE_KEY=YOUR-SERVICE-ROLE-KEY
SUPABASE_ANON_KEY=YOUR-ANON-KEY
PRU_ENDPOINT=http://localhost:5050
"""

# ============================================================================
# FILE: backend/api/models.py
# PATH: ./backend/api/models.py
# ----------------------------------------------------------------------------
MODELS_PY = r"""
from __future__ import annotations
from pydantic import BaseModel, HttpUrl, Field
from typing import List, Optional

class DonationLinks(BaseModel):
    lightning: Optional[str] = None
    evm: Optional[str] = None
    stripe: Optional[str] = None

class BinaryInfo(BaseModel):
    version: str = Field(..., examples=["1.0.3"])
    url: HttpUrl
    sha256: str

class PrivacyInfo(BaseModel):
    tracking: bool = False
    data_collected: List[str] = []

class AppManifest(BaseModel):
    id: str = Field(..., pattern=r"^[a-zA-Z0-9_.-]+$")
    name: str
    summary: str
    tags: List[str] = []
    platforms: List[str] = ["android"]
    binary: BinaryInfo
    permissions: List[str] = []
    screenshots: List[HttpUrl] = []
    privacy: PrivacyInfo = PrivacyInfo()
    donations: Optional[DonationLinks] = None
    developer: Optional[str] = None

class AppRow(BaseModel):
    id: str
    name: str
    summary: str
    tags: List[str]
    platforms: List[str]
    latest_version: str
    developer: Optional[str] = None
    rating: Optional[float] = None
    installs: int = 0
    binary_url: Optional[str] = None
    sha256: Optional[str] = None
"""

# ============================================================================
# FILE: backend/api/deps.py
# PATH: ./backend/api/deps.py
# ----------------------------------------------------------------------------
DEPS_PY = r"""
import os
from supabase import create_client, Client
from dotenv import load_dotenv

load_dotenv()

SUPABASE_URL = os.environ.get("SUPABASE_URL")
SUPABASE_SERVICE_KEY = os.environ.get("SUPABASE_SERVICE_KEY")

if not SUPABASE_URL or not SUPABASE_SERVICE_KEY:
    raise RuntimeError("Supabase credentials missing in env")

supabase: Client = create_client(SUPABASE_URL, SUPABASE_SERVICE_KEY)
"""

# ============================================================================
# FILE: backend/api/main.py
# PATH: ./backend/api/main.py
# ----------------------------------------------------------------------------
MAIN_PY = r"""
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from .models import AppManifest, AppRow
from .deps import supabase
from typing import List

app = FastAPI(title="AdilStore API", version="0.1.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/apps", response_model=List[AppRow])
def list_apps():
    resp = supabase.table("apps_view").select("*").execute()
    rows = resp.data or []
    return rows

@app.get("/apps/{app_id}", response_model=AppRow)
def get_app(app_id: str):
    resp = supabase.table("apps_view").select("*").eq("id", app_id).limit(1).execute()
    if not resp.data:
        raise HTTPException(404, detail="app not found")
    return resp.data[0]

@app.get("/search", response_model=List[AppRow])
def search(q: str):
    # naive search; PRU-DB integration later
    resp = supabase.rpc("search_apps", {"q": q}).execute()
    return resp.data or []

@app.post("/ingest/manifest", response_model=AppRow)
def ingest_manifest(manifest: AppManifest):
    # upsert app
    app_row = {
        "id": manifest.id,
        "name": manifest.name,
        "summary": manifest.summary,
        "tags": manifest.tags,
        "platforms": manifest.platforms,
        "developer": manifest.developer,
    }
    supabase.table("apps").upsert(app_row).execute()

    ver_row = {
        "app_id": manifest.id,
        "version": manifest.binary.version,
        "binary_url": str(manifest.binary.url),
        "sha256": manifest.binary.sha256,
        "permissions": manifest.permissions,
    }
    supabase.table("app_versions").upsert(ver_row, on_conflict="app_id,version").execute()

    # return combined view row
    resp = supabase.table("apps_view").select("*").eq("id", manifest.id).limit(1).execute()
    return resp.data[0]
"""

# ============================================================================
# FILE: database/schema.sql
# PATH: ./database/schema.sql
# ----------------------------------------------------------------------------
SCHEMA_SQL = r"""
-- AdilStore schema (Postgres/Supabase)
create table if not exists developers (
  id uuid primary key default gen_random_uuid(),
  handle text unique not null,
  display_name text,
  website text,
  created_at timestamptz default now()
);

create table if not exists apps (
  id text primary key,
  name text not null,
  summary text not null,
  tags text[] default array[]::text[],
  platforms text[] default array['android'],
  developer text references developers(handle) on delete set null,
  created_at timestamptz default now()
);

create table if not exists app_versions (
  app_id text references apps(id) on delete cascade,
  version text not null,
  binary_url text not null,
  sha256 text not null,
  permissions text[] default array[]::text[],
  created_at timestamptz default now(),
  primary key(app_id, version)
);

create table if not exists ratings (
  app_id text references apps(id) on delete cascade,
  user_id uuid,
  stars int check (stars between 1 and 5),
  comment text,
  created_at timestamptz default now()
);

create table if not exists installs (
  app_id text references apps(id) on delete cascade,
  country text,
  created_at timestamptz default now()
);

-- materialized view / view for latest versions (simple approach)
create or replace view apps_view as
select a.id,
       a.name,
       a.summary,
       a.tags,
       a.platforms,
       a.developer,
       coalesce(r.avg_rating, null) as rating,
       coalesce(i.installs, 0) as installs,
       v.version as latest_version,
       v.binary_url,
       v.sha256
from apps a
left join lateral (
  select version, binary_url, sha256
  from app_versions v
  where v.app_id = a.id
  order by created_at desc
  limit 1
) v on true
left join (
  select app_id, avg(stars)::float as avg_rating
  from ratings
  group by app_id
) r on r.app_id = a.id
left join (
  select app_id, count(*)::int as installs
  from installs
  group by app_id
) i on i.app_id = a.id;

-- simple text search function (placeholder for PRU)
create or replace function search_apps(q text)
returns setof apps_view language sql stable as $$
  select * from apps_view
  where name ilike '%'||q||'%' or summary ilike '%'||q||'%'
  order by rating desc nulls last, installs desc, name asc
  limit 50;
$$;
"""

# ============================================================================
# FILE: devtools/cli.py
# PATH: ./devtools/cli.py
# ----------------------------------------------------------------------------
CLI_PY = r"""
#!/usr/bin/env python3
import json, sys, hashlib, re
from pathlib import Path

SCHEMA = {
  "required": ["id","name","summary","binary"],
}

ID_RE = re.compile(r"^[a-zA-Z0-9_.-]+$")

def sha256_file(path: Path) -> str:
    h = hashlib.sha256()
    with path.open('rb') as f:
        for chunk in iter(lambda: f.read(8192), b''):
            h.update(chunk)
    return h.hexdigest()


def validate_manifest(p: Path) -> int:
    data = json.loads(p.read_text(encoding='utf-8'))
    for key in SCHEMA["required"]:
        if key not in data:
            print(f"[x] missing: {key}")
            return 2
    if not ID_RE.match(data["id"]):
        print("[x] invalid id format")
        return 2
    print("[ok] manifest valid")
    return 0

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("usage: cli.py manifests/example.app.json")
        sys.exit(2)
    sys.exit(validate_manifest(Path(sys.argv[1])))
"""

# ============================================================================
# FILE: manifests/example.app.json
# PATH: ./manifests/example.app.json
# ----------------------------------------------------------------------------
EXAMPLE_MANIFEST = r"""
{
  "id": "com.example.focus",
  "name": "Focus",
  "summary": "Distraction-free focus timer for deep work.",
  "tags": ["productivity", "focus"],
  "platforms": ["android"],
  "binary": {
    "version": "1.0.0",
    "url": "https://cdn.example.com/focus-1.0.0.apk",
    "sha256": "REPLACE_WITH_SHA256"
  },
  "permissions": ["INTERNET", "POST_NOTIFICATIONS"],
  "screenshots": ["https://cdn.example.com/focus/1.png"],
  "privacy": {"tracking": false, "data_collected": []},
  "donations": {"lightning": "", "evm": "0x...", "stripe": "acct_..."},
  "developer": "umut"
}
"""

# ============================================================================
# FILE: apps/store_client/pubspec.yaml
# PATH: ./apps/store_client/pubspec.yaml
# ----------------------------------------------------------------------------
PUBSPEC = r"""
name: adilstore_client
description: Minimal AdilStore client (MVP)
version: 0.1.0
publish_to: 'none'

environment:
  sdk: '>=3.4.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.2

flutter:
  uses-material-design: true
"""

# ============================================================================
# FILE: apps/store_client/lib/main.dart
# PATH: ./apps/store_client/lib/main.dart
# ----------------------------------------------------------------------------
MAIN_DART = r"""
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;

const apiBase = String.fromEnvironment('ADILSTORE_API', defaultValue: 'http://localhost:8000');

void main() {
  runApp(const AdilStoreApp());
}

class AdilStoreApp extends StatelessWidget {
  const AdilStoreApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'AdilStore',
      home: const CatalogPage(),
      theme: ThemeData(useMaterial3: true),
    );
  }
}

class CatalogPage extends StatefulWidget {
  const CatalogPage({super.key});

  @override
  State<CatalogPage> createState() => _CatalogPageState();
}

class _CatalogPageState extends State<CatalogPage> {
  List<dynamic> apps = [];
  bool loading = true;

  @override
  void initState() {
    super.initState();
    _fetch();
  }

  Future<void> _fetch() async {
    try {
      final res = await http.get(Uri.parse('$apiBase/apps'));
      if (res.statusCode == 200) {
        setState(() { apps = json.decode(res.body); loading = false; });
      } else { setState(() { loading = false; }); }
    } catch (_) { setState(() { loading = false; }); }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('AdilStore')),
      body: loading
          ? const Center(child: CircularProgressIndicator())
          : ListView.separated(
              itemCount: apps.length,
              separatorBuilder: (_, __) => const Divider(height: 1),
              itemBuilder: (context, i) {
                final a = apps[i];
                return ListTile(
                  title: Text(a['name'] ?? ''),
                  subtitle: Text(a['summary'] ?? ''),
                  trailing: Text(a['latest_version'] ?? ''),
                  onTap: () {},
                );
              },
            ),
    );
  }
}
"""

# ─────────────────────────────────────────────────────────────────────────────
# END OF SCAFFOLD
# ─────────────────────────────────────────────────────────────────────────────
