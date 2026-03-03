# Elyo AI – Full-Stack Project Blueprint

## Directory Structure

```
elyo-ai/
├── README.md
├── .env.local.example
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── supabase/
│   └── migrations/
│       ├── 001_schema.sql
│       ├── 002_rls.sql
│       └── 003_seed.sql
├── src/
│   ├── app/
│   │   ├── (marketing)/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx                  ← Landing page
│   │   │   ├── product/page.tsx
│   │   │   ├── demo/page.tsx
│   │   │   ├── privacy/page.tsx
│   │   │   └── terms/page.tsx
│   │   ├── (product)/
│   │   │   ├── layout.tsx                ← Sidebar + Topbar
│   │   │   ├── app/
│   │   │   │   ├── page.tsx              ← Dashboard
│   │   │   │   ├── onboarding/page.tsx
│   │   │   │   ├── agent/page.tsx
│   │   │   │   ├── workflows/
│   │   │   │   │   ├── page.tsx          ← Workflow list
│   │   │   │   │   └── [id]/page.tsx     ← Builder
│   │   │   │   ├── analytics/page.tsx
│   │   │   │   ├── achievements/page.tsx
│   │   │   │   ├── team/page.tsx
│   │   │   │   └── settings/page.tsx
│   │   └── api/
│   │       ├── agent/route.ts
│   │       ├── auth/callback/route.ts
│   │       ├── integrations/connect/route.ts
│   │       ├── workflows/
│   │       │   ├── route.ts
│   │       │   └── [id]/simulate/route.ts
│   │       ├── mood/route.ts
│   │       └── analytics/route.ts
│   ├── components/
│   │   ├── ui/                           ← Design system primitives
│   │   │   ├── Button.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── Badge.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Modal.tsx
│   │   │   ├── Skeleton.tsx
│   │   │   └── ThemeToggle.tsx
│   │   ├── marketing/
│   │   │   ├── Hero.tsx
│   │   │   ├── Features.tsx
│   │   │   ├── HowItWorks.tsx
│   │   │   ├── Integrations.tsx
│   │   │   ├── Pricing.tsx
│   │   │   ├── Testimonials.tsx
│   │   │   └── Footer.tsx
│   │   ├── app/
│   │   │   ├── Sidebar.tsx
│   │   │   ├── Topbar.tsx
│   │   │   ├── WorkspaceSwitcher.tsx
│   │   │   ├── MoodCheckin.tsx
│   │   │   ├── DashboardCard.tsx
│   │   │   ├── AgentChat.tsx
│   │   │   ├── WorkflowBuilder.tsx
│   │   │   ├── WorkflowNode.tsx
│   │   │   ├── AnalyticsChart.tsx
│   │   │   └── AchievementCard.tsx
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── client.ts                 ← Browser client
│   │   │   ├── server.ts                 ← Server client
│   │   │   └── middleware.ts
│   │   ├── agent/
│   │   │   ├── planner.ts               ← Intent extraction
│   │   │   ├── tools.ts                 ← Tool definitions + stubs
│   │   │   └── loop.ts                  ← Tool-calling loop
│   │   ├── analytics.ts
│   │   ├── achievements.ts
│   │   ├── seed.ts                      ← New-user seeding
│   │   └── validators.ts                ← Zod schemas
│   ├── hooks/
│   │   ├── useWorkspace.ts
│   │   ├── useMood.ts
│   │   ├── useAgent.ts
│   │   └── useTheme.ts
│   └── types/
│       └── index.ts
```

---

## Key Source Files

### `.env.local.example`
```env
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
LLM_PROVIDER_API_KEY=your-llm-key
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

### `supabase/migrations/001_schema.sql`
```sql
-- Enable UUID extension
create extension if not exists "pgcrypto";

-- Profiles
create table profiles (
  user_id uuid primary key references auth.users(id) on delete cascade,
  name text,
  role text default 'member',
  timezone text default 'UTC',
  work_hours jsonb default '{"start": "09:00", "end": "17:00"}'::jsonb,
  mood_preferences jsonb default '{}'::jsonb,
  hourly_rate numeric default 75,
  updated_at timestamptz default now()
);

-- Workspaces
create table workspaces (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  owner_id uuid references auth.users(id),
  created_at timestamptz default now()
);

-- Workspace Members
create table workspace_members (
  workspace_id uuid references workspaces(id) on delete cascade,
  user_id uuid references auth.users(id) on delete cascade,
  role text default 'member' check (role in ('owner','admin','member')),
  skill_tags text[] default '{}',
  workload_score int default 0,
  primary key (workspace_id, user_id)
);

-- Integrations
create table integrations (
  id uuid primary key default gen_random_uuid(),
  workspace_id uuid references workspaces(id) on delete cascade,
  provider text not null,
  status text default 'disconnected',
  oauth_tokens_encrypted text,
  connected_at timestamptz,
  unique(workspace_id, provider)
);

-- Mood Entries
create table mood_entries (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade,
  date date default current_date,
  score int check (score between 1 and 5),
  notes text,
  created_at timestamptz default now(),
  unique(user_id, date)
);

-- Workflows
create table workflows (
  id uuid primary key default gen_random_uuid(),
  workspace_id uuid references workspaces(id) on delete cascade,
  name text not null,
  json_definition jsonb default '{}'::jsonb,
  is_active boolean default true,
  created_by uuid references auth.users(id),
  created_at timestamptz default now()
);

-- Workflow Runs
create table workflow_runs (
  id uuid primary key default gen_random_uuid(),
  workflow_id uuid references workflows(id) on delete cascade,
  started_at timestamptz default now(),
  status text default 'pending' check (status in ('pending','running','success','failed')),
  input_json jsonb default '{}'::jsonb,
  output_json jsonb default '{}'::jsonb,
  time_saved_minutes int default 0
);

-- Agent Conversations
create table agent_conversations (
  id uuid primary key default gen_random_uuid(),
  workspace_id uuid references workspaces(id) on delete cascade,
  user_id uuid references auth.users(id) on delete cascade,
  title text,
  created_at timestamptz default now()
);

-- Agent Messages
create table agent_messages (
  id uuid primary key default gen_random_uuid(),
  conversation_id uuid references agent_conversations(id) on delete cascade,
  role text check (role in ('user','assistant','tool')),
  content text,
  created_at timestamptz default now(),
  metadata_json jsonb default '{}'::jsonb
);

-- Achievements (global catalogue)
create table achievements (
  key text primary key,
  title text not null,
  description text,
  icon text,
  threshold int default 1
);

-- User Achievements
create table user_achievements (
  user_id uuid references auth.users(id) on delete cascade,
  achievement_key text references achievements(key),
  earned_at timestamptz,
  progress int default 0,
  primary key (user_id, achievement_key)
);

-- Activity Log
create table activity_log (
  id uuid primary key default gen_random_uuid(),
  workspace_id uuid references workspaces(id) on delete cascade,
  user_id uuid references auth.users(id),
  provider text,
  action_type text,
  occurred_at timestamptz default now(),
  metadata_json jsonb default '{}'::jsonb
);

-- Seed achievement catalogue
insert into achievements (key, title, description, icon, threshold) values
  ('first_automation', 'First Automation', 'Run your first workflow', '🤖', 1),
  ('one_hour_saved',   '1 Hour Saved',     'Save 60+ minutes via automations', '⏱️', 60),
  ('streak_3',         'Check-in Streak',  'Log mood 3 days in a row', '🔥', 3),
  ('five_workflows',   'Workflow Architect','Create 5 workflows', '🏗️', 5),
  ('team_player',      'Team Player',       'Route a task to a teammate', '🤝', 1);
```

---

### `supabase/migrations/002_rls.sql`
```sql
-- Helper: is user a member of workspace?
create or replace function is_workspace_member(ws_id uuid)
returns boolean language sql security definer as $$
  select exists (
    select 1 from workspace_members
    where workspace_id = ws_id and user_id = auth.uid()
  );
$$;

-- Helper: is user owner or admin?
create or replace function is_workspace_admin(ws_id uuid)
returns boolean language sql security definer as $$
  select exists (
    select 1 from workspace_members
    where workspace_id = ws_id
      and user_id = auth.uid()
      and role in ('owner','admin')
  );
$$;

alter table profiles enable row level security;
create policy "profiles_self" on profiles using (user_id = auth.uid());

alter table workspaces enable row level security;
create policy "workspaces_member" on workspaces
  using (is_workspace_member(id));

alter table workspace_members enable row level security;
create policy "wm_member_read" on workspace_members
  using (is_workspace_member(workspace_id));
create policy "wm_admin_write" on workspace_members
  for insert with check (is_workspace_admin(workspace_id));

alter table integrations enable row level security;
create policy "integrations_member" on integrations
  using (is_workspace_member(workspace_id));
create policy "integrations_admin_write" on integrations
  for all using (is_workspace_admin(workspace_id));

alter table mood_entries enable row level security;
create policy "mood_self" on mood_entries using (user_id = auth.uid());

alter table workflows enable row level security;
create policy "workflows_member_read" on workflows
  using (is_workspace_member(workspace_id));
create policy "workflows_member_write" on workflows
  for all using (is_workspace_member(workspace_id));

alter table workflow_runs enable row level security;
create policy "runs_member" on workflow_runs
  using (
    workflow_id in (
      select id from workflows where is_workspace_member(workspace_id)
    )
  );

alter table agent_conversations enable row level security;
create policy "conv_member" on agent_conversations
  using (is_workspace_member(workspace_id));

alter table agent_messages enable row level security;
create policy "msg_member" on agent_messages
  using (
    conversation_id in (
      select id from agent_conversations where is_workspace_member(workspace_id)
    )
  );

alter table user_achievements enable row level security;
create policy "ach_self" on user_achievements using (user_id = auth.uid());

alter table activity_log enable row level security;
create policy "log_member" on activity_log
  using (is_workspace_member(workspace_id));
```

---

### `src/lib/supabase/server.ts`
```typescript
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export function createClient() {
  const cookieStore = cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: (cs) => cs.forEach(({ name, value, options }) =>
          cookieStore.set(name, value, options)
        ),
      },
    }
  );
}

export function createServiceClient() {
  const { createClient: _c } = require('@supabase/supabase-js');
  return _c(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  );
}
```

---

### `src/lib/agent/tools.ts`
```typescript
import type { SupabaseClient } from '@supabase/supabase-js';

export const TOOL_DEFINITIONS = [
  { name: 'calendar.getEvents',   description: 'Fetch upcoming calendar events' },
  { name: 'calendar.createEvent', description: 'Create a calendar event' },
  { name: 'gmail.getThreads',     description: 'Fetch recent Gmail threads' },
  { name: 'gmail.sendEmail',      description: 'Send an email via Gmail' },
  { name: 'slack.postMessage',    description: 'Post a Slack message to a channel' },
  { name: 'notion.createPage',    description: 'Create a Notion page' },
  { name: 'workflow.runSimulation', description: 'Simulate a workflow with sample input' },
];

type ToolResult = { success: boolean; data: unknown; mocked: boolean };

export async function executeTool(
  toolName: string,
  params: Record<string, unknown>,
  supabase: SupabaseClient,
  workspaceId: string,
  userId: string
): Promise<ToolResult> {
  // Log every tool call
  await supabase.from('activity_log').insert({
    workspace_id: workspaceId,
    user_id: userId,
    provider: toolName.split('.')[0],
    action_type: toolName,
    metadata_json: { params },
  });

  // Mocked implementations – swap for real SDK calls when tokens are present
  const mocks: Record<string, unknown> = {
    'calendar.getEvents': {
      events: [
        { id: '1', title: 'Team Standup', start: new Date(Date.now() + 3600_000).toISOString() },
        { id: '2', title: 'Product Review', start: new Date(Date.now() + 86400_000).toISOString() },
      ],
    },
    'calendar.createEvent': { id: 'new-evt-1', status: 'confirmed', ...params },
    'gmail.getThreads': {
      threads: [
        { id: 't1', subject: 'Q2 Planning', snippet: 'Following up on the roadmap…' },
        { id: 't2', subject: 'Invoice #1042', snippet: 'Please find attached…' },
      ],
    },
    'gmail.sendEmail': { messageId: 'msg-mock-001', status: 'sent' },
    'slack.postMessage': { ts: Date.now().toString(), channel: params.channel ?? '#general' },
    'notion.createPage': { pageId: 'notion-mock-001', url: 'https://notion.so/mock' },
    'workflow.runSimulation': {
      status: 'success',
      steps_completed: 3,
      time_saved_minutes: 12,
      output: { summary: 'Simulation complete – 3 actions executed successfully.' },
    },
  };

  return { success: true, data: mocks[toolName] ?? { ok: true }, mocked: true };
}
```

---

### `src/app/api/agent/route.ts`
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/server';
import { executeTool, TOOL_DEFINITIONS } from '@/lib/agent/tools';
import { z } from 'zod';

const BodySchema = z.object({
  message: z.string().min(1),
  conversationId: z.string().uuid().optional(),
  workspaceId: z.string().uuid(),
});

export async function POST(req: NextRequest) {
  const supabase = createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const body = BodySchema.parse(await req.json());

  // Upsert conversation
  let convId = body.conversationId;
  if (!convId) {
    const { data } = await supabase.from('agent_conversations').insert({
      workspace_id: body.workspaceId,
      user_id: user.id,
      title: body.message.slice(0, 60),
    }).select('id').single();
    convId = data!.id;
  }

  // Store user message
  await supabase.from('agent_messages').insert({
    conversation_id: convId,
    role: 'user',
    content: body.message,
  });

  // Fetch mood for context
  const { data: mood } = await supabase
    .from('mood_entries')
    .select('score')
    .eq('user_id', user.id)
    .eq('date', new Date().toISOString().split('T')[0])
    .maybeSingle();

  const moodScore = mood?.score ?? 3;
  const toneHint = moodScore <= 2
    ? 'Keep suggestions brief and low-effort.'
    : 'Be thorough and proactive.';

  // Call LLM
  const llmRes = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': process.env.LLM_PROVIDER_API_KEY!,
      'anthropic-version': '2023-06-01',
    },
    body: JSON.stringify({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      system: `You are Elyo, an AI productivity agent. ${toneHint}
Available tools: ${TOOL_DEFINITIONS.map(t => t.name).join(', ')}.
When proposing a workflow, return a JSON block tagged \`\`\`workflow ... \`\`\`.`,
      messages: [{ role: 'user', content: body.message }],
    }),
  });

  const llmData = await llmRes.json();
  const assistantText = llmData.content?.[0]?.text ?? 'I could not process that request.';

  // Detect tool calls mentioned in reply (simplified pattern match)
  const toolPattern = /\b(calendar\.getEvents|gmail\.getThreads|slack\.postMessage|notion\.createPage|workflow\.runSimulation)\b/g;
  const toolsFound = [...new Set(assistantText.match(toolPattern) ?? [])];
  const toolResults: Record<string, unknown> = {};

  for (const tool of toolsFound) {
    const res = await executeTool(tool, {}, supabase, body.workspaceId, user.id);
    toolResults[tool] = res.data;
  }

  // Store assistant message
  await supabase.from('agent_messages').insert({
    conversation_id: convId,
    role: 'assistant',
    content: assistantText,
    metadata_json: { tool_calls: toolsFound, tool_results: toolResults },
  });

  return NextResponse.json({
    conversationId: convId,
    reply: assistantText,
    toolResults,
  });
}
```

---

### `src/app/api/workflows/[id]/simulate/route.ts`
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/server';

export async function POST(req: NextRequest, { params }: { params: { id: string } }) {
  const supabase = createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const { input } = await req.json();
  const timeSaved = Math.floor(Math.random() * 20) + 5;

  const { data: run } = await supabase.from('workflow_runs').insert({
    workflow_id: params.id,
    status: 'success',
    input_json: input ?? {},
    output_json: { simulated: true, steps: 3 },
    time_saved_minutes: timeSaved,
  }).select().single();

  return NextResponse.json({ run, timeSavedMinutes: timeSaved });
}
```

---

### `src/lib/seed.ts`
```typescript
import { createServiceClient } from './supabase/server';

export async function seedNewUser(userId: string, email: string) {
  const sb = createServiceClient();

  // Profile
  await sb.from('profiles').upsert({ user_id: userId, name: email.split('@')[0] });

  // Workspace
  const { data: ws } = await sb.from('workspaces')
    .insert({ name: 'My Workspace', owner_id: userId })
    .select().single();

  // Member
  await sb.from('workspace_members').insert({
    workspace_id: ws!.id, user_id: userId, role: 'owner',
    skill_tags: ['writing', 'planning'], workload_score: 2,
  });

  // Integrations (disconnected stubs)
  await sb.from('integrations').insert([
    { workspace_id: ws!.id, provider: 'google', status: 'disconnected' },
    { workspace_id: ws!.id, provider: 'slack',  status: 'disconnected' },
    { workspace_id: ws!.id, provider: 'notion', status: 'disconnected' },
  ]);

  // Sample workflows
  const wfDefs = [
    {
      name: 'Meeting Follow-up',
      json_definition: {
        nodes: [
          { id: 'n1', type: 'trigger', label: 'Calendar Event Ended', data: {} },
          { id: 'n2', type: 'action',  label: 'Send Follow-up Email', data: { template: 'follow_up' } },
          { id: 'n3', type: 'action',  label: 'Create Notion Summary', data: {} },
        ],
        edges: [{ source: 'n1', target: 'n2' }, { source: 'n2', target: 'n3' }],
      },
    },
    {
      name: 'Inbox Triage',
      json_definition: {
        nodes: [
          { id: 'n1', type: 'trigger',   label: 'Email Received', data: {} },
          { id: 'n2', type: 'condition', label: 'Contains Invoice?', data: { field: 'subject', op: 'contains', value: 'invoice' } },
          { id: 'n3', type: 'action',    label: 'Tag & Archive', data: {} },
          { id: 'n4', type: 'action',    label: 'Forward to Slack', data: { channel: '#finance' } },
        ],
        edges: [
          { source: 'n1', target: 'n2' },
          { source: 'n2', target: 'n3', label: 'Yes' },
          { source: 'n2', target: 'n4', label: 'No' },
        ],
      },
    },
  ];

  for (const wf of wfDefs) {
    const { data: w } = await sb.from('workflows')
      .insert({ ...wf, workspace_id: ws!.id, created_by: userId })
      .select().single();

    // Seed runs for analytics
    await sb.from('workflow_runs').insert([
      { workflow_id: w!.id, status: 'success', time_saved_minutes: 14,
        input_json: {}, output_json: { steps: 3 } },
      { workflow_id: w!.id, status: 'success', time_saved_minutes: 9,
        input_json: {}, output_json: { steps: 2 } },
    ]);
  }

  // Seed mood entries
  for (let i = 4; i >= 0; i--) {
    const d = new Date(); d.setDate(d.getDate() - i);
    await sb.from('mood_entries').insert({
      user_id: userId,
      date: d.toISOString().split('T')[0],
      score: Math.floor(Math.random() * 3) + 3,
    });
  }

  // User achievements
  await sb.from('user_achievements').insert([
    { user_id: userId, achievement_key: 'first_automation', earned_at: new Date().toISOString(), progress: 1 },
    { user_id: userId, achievement_key: 'streak_3',         progress: 2 },
  ]);

  return ws!.id;
}
```

---

### `src/app/(product)/layout.tsx`
```tsx
import Sidebar from '@/components/app/Sidebar';
import Topbar  from '@/components/app/Topbar';
import { createClient } from '@/lib/supabase/server';
import { redirect } from 'next/navigation';

export default async function ProductLayout({ children }: { children: React.ReactNode }) {
  const supabase = createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');

  return (
    <div className="flex h-screen overflow-hidden bg-background">
      <Sidebar />
      <div className="flex flex-col flex-1 overflow-hidden">
        <Topbar user={user} />
        <main className="flex-1 overflow-y-auto p-6">{children}</main>
      </div>
    </div>
  );
}
```

---

### `src/components/app/MoodCheckin.tsx`
```tsx
'use client';
import { useState } from 'react';

const EMOJIS = ['😴','😟','😐','😊','🚀'];

export default function MoodCheckin() {
  const [selected, setSelected] = useState<number | null>(null);
  const [saved, setSaved] = useState(false);

  async function save(score: number) {
    setSelected(score);
    await fetch('/api/mood', {
      method: 'POST',
      body: JSON.stringify({ score }),
      headers: { 'Content-Type': 'application/json' },
    });
    setSaved(true);
  }

  return (
    <div className="rounded-2xl border bg-card p-4 shadow-sm">
      <p className="text-sm font-medium text-muted-foreground mb-3">
        How's your energy today?
      </p>
      {saved ? (
        <p className="text-teal-500 font-semibold text-sm">✓ Logged! Elyo will adapt your suggestions.</p>
      ) : (
        <div className="flex gap-2">
          {EMOJIS.map((e, i) => (
            <button
              key={i}
              onClick={() => save(i + 1)}
              className={`text-2xl p-2 rounded-xl transition hover:scale-110 ${
                selected === i + 1 ? 'bg-teal-100 dark:bg-teal-900 ring-2 ring-teal-500' : ''
              }`}
            >
              {e}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

### `src/app/(marketing)/page.tsx` (Landing – structure)
```tsx
import Hero         from '@/components/marketing/Hero';
import Features     from '@/components/marketing/Features';
import HowItWorks   from '@/components/marketing/HowItWorks';
import Integrations from '@/components/marketing/Integrations';
import Pricing      from '@/components/marketing/Pricing';
import Testimonials from '@/components/marketing/Testimonials';
import Footer       from '@/components/marketing/Footer';

export default function LandingPage() {
  return (
    <main>
      <Hero />
      <Features />
      <HowItWorks />
      <Integrations />
      <Pricing />
      <Testimonials />
      <Footer />
    </main>
  );
}
```

---

### `src/components/marketing/Hero.tsx`
```tsx
export default function Hero() {
  return (
    <section className="relative overflow-hidden bg-gradient-to-br from-slate-950 via-teal-950 to-blue-950 text-white py-28 px-6 text-center">
      {/* Gradient orbs */}
      <div className="absolute -top-32 left-1/2 -translate-x-1/2 w-[600px] h-[600px] rounded-full bg-teal-500/20 blur-3xl pointer-events-none" />
      <div className="relative z-10 max-w-4xl mx-auto">
        <div className="inline-flex items-center gap-2 bg-teal-500/10 border border-teal-500/30 rounded-full px-4 py-1.5 text-sm text-teal-300 mb-6">
          🧠 Now with Predictive Automation Feed
        </div>
        <h1 className="text-5xl md:text-7xl font-extrabold tracking-tight mb-6 bg-gradient-to-r from-white via-teal-200 to-blue-300 bg-clip-text text-transparent">
          Your Mood-Adaptive<br />AI Teammate
        </h1>
        <p className="text-xl text-slate-300 max-w-2xl mx-auto mb-10">
          <strong className="text-white">Elyo AI</strong> combines mood-adaptive suggestions,
          predictive automation, team intelligence routing, a visual workflow builder,
          ROI analytics, and gamified productivity — so your team ships more with less friction.
        </p>
        <div className="flex flex-col sm:flex-row gap-4 justify-center">
          <a href="/app" className="bg-teal-500 hover:bg-teal-400 text-white font-semibold py-3 px-8 rounded-xl transition">
            Start Free Trial
          </a>
          <a href="/demo" className="border border-white/20 hover:bg-white/10 text-white font-semibold py-3 px-8 rounded-xl transition">
            Live Demo ↗
          </a>
        </div>
        {/* Stats row */}
        <div className="mt-16 grid grid-cols-3 gap-8 text-center">
          {[
            { val: '4.2h', label: 'saved per user / day' },
            { val: '94%', label: 'automation success rate' },
            { val: '$840', label: 'avg monthly ROI' },
          ].map(s => (
            <div key={s.label}>
              <div className="text-3xl font-bold text-teal-300">{s.val}</div>
              <div className="text-sm text-slate-400 mt-1">{s.label}</div>
            </div>
          ))}
        </div>
      </div>
    </section>
  );
}
```

---

### `tailwind.config.ts` (excerpt)
```typescript
import type { Config } from 'tailwindcss';

const config: Config = {
  darkMode: 'class',
  content: ['./src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        teal:       { DEFAULT: '#14b8a6', 50: '#f0fdfa', 500: '#14b8a6', 900: '#134e4a' },
        background: 'hsl(var(--background))',
        card:       'hsl(var(--card))',
        'muted-foreground': 'hsl(var(--muted-foreground))',
      },
      fontFamily: { sans: ['Inter', 'system-ui', 'sans-serif'] },
      borderRadius: { '2xl': '1rem', '3xl': '1.5rem' },
    },
  },
  plugins: [],
};
export default config;
```

---

## README

### Setup

```bash
git clone <repo>
cd elyo-ai
cp .env.local.example .env.local
# Fill in env vars
pnpm install
```

### Supabase

```bash
# Install Supabase CLI
supabase link --project-ref <your-ref>
supabase db push          # runs all migrations in order
```

### Run locally

```bash
pnpm dev   # http://localhost:3000
```

### Deploy (Vercel)

```bash
vercel --prod
# Add all env vars in Vercel dashboard
```

### Trigger new-user seed

The seed function is called from `/api/auth/callback/route.ts` after OAuth exchange:

```typescript
import { seedNewUser } from '@/lib/seed';
// After user is confirmed:
await seedNewUser(user.id, user.email!);
```

---

## Pricing Tiers

| Tier | Price | Highlights |
|------|-------|-----------|
| **Starter** | Free | 3 workflows, 1 integration, 100 agent msgs/mo |
| **Pro** | $29/mo | Unlimited workflows, all integrations, ROI analytics, team features |
| **Enterprise** | Custom | SSO, audit log, SLA, dedicated support |
