# User Portal Implementation Guide (Month 6)

**Purpose**: Complete implementation guide for the Chat SDK User Portal (SaaS Customer Portal)  
**Technology**: Next.js 14, React 18, TailwindCSS, shadcn/ui, React Query

---

## Overview

The User Portal is a self-service web application for Chat SDK customers to manage their integration, teams, users, and settings.

### Key Features:
- 🔐 Authentication & Authorization
- 👥 Team/Workspace Management
- 🔑 API Key Generation & Management
- 📊 Analytics Dashboard
- 🪝 Webhook Configuration
- 💳 Billing & Subscription
- ⚙️ Settings & Preferences

---

## Setup

### 1. Initialize User Portal (if not done in Month 1)

```bash
cd packages/user-portal
pnpm install
```

### 2. Install Additional Dependencies

```bash
pnpm add @tanstack/react-query recharts date-fns lucide-react
pnpm add -D @types/node
```

### 3. Install shadcn/ui Components

```bash
npx shadcn-ui@latest init
npx shadcn-ui@latest add button card input label select table tabs dialog dropdown-menu
npx shadcn-ui@latest add form toast alert badge avatar separator
```

---

## Project Structure

```
packages/user-portal/
├── app/
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   ├── signup/page.tsx
│   │   ├── forgot-password/page.tsx
│   │   └── layout.tsx
│   ├── (dashboard)/
│   │   ├── dashboard/page.tsx
│   │   ├── teams/
│   │   │   ├── page.tsx
│   │   │   ├── [id]/page.tsx
│   │   │   └── new/page.tsx
│   │   ├── users/page.tsx
│   │   ├── channels/page.tsx
│   │   ├── analytics/page.tsx
│   │   ├── webhooks/
│   │   │   ├── page.tsx
│   │   │   └── [id]/page.tsx
│   │   ├── api-keys/page.tsx
│   │   ├── billing/page.tsx
│   │   ├── settings/page.tsx
│   │   └── layout.tsx
│   ├── api/
│   │   └── auth/[...nextauth]/route.ts
│   ├── layout.tsx
│   └── page.tsx
├── components/
│   ├── ui/              # shadcn/ui components
│   ├── dashboard/
│   │   ├── sidebar.tsx
│   │   ├── header.tsx
│   │   └── stats-card.tsx
│   ├── analytics/
│   │   ├── usage-chart.tsx
│   │   ├── activity-chart.tsx
│   │   └── metrics-grid.tsx
│   ├── teams/
│   │   ├── team-list.tsx
│   │   ├── team-form.tsx
│   │   └── member-list.tsx
│   ├── webhooks/
│   │   ├── webhook-list.tsx
│   │   ├── webhook-form.tsx
│   │   └── webhook-logs.tsx
│   └── shared/
│       ├── loading.tsx
│       └── error-boundary.tsx
├── lib/
│   ├── api-client.ts
│   ├── auth.ts
│   ├── utils.ts
│   └── hooks/
│       ├── use-teams.ts
│       ├── use-analytics.ts
│       └── use-webhooks.ts
├── types/
│   └── index.ts
└── public/
```

---

## Implementation

### 1. Dashboard Layout

**Create `app/(dashboard)/layout.tsx`**:

```typescript
import { Sidebar } from '@/components/dashboard/sidebar';
import { Header } from '@/components/dashboard/header';

export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex h-screen">
      <Sidebar />
      <div className="flex-1 flex flex-col overflow-hidden">
        <Header />
        <main className="flex-1 overflow-y-auto bg-gray-50 p-6">
          {children}
        </main>
      </div>
    </div>
  );
}
```

### 2. Dashboard Home

**Create `app/(dashboard)/dashboard/page.tsx`**:

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';
import { api } from '@/lib/api-client';
import { StatsCard } from '@/components/dashboard/stats-card';
import { UsageChart } from '@/components/analytics/usage-chart';
import { ActivityChart } from '@/components/analytics/activity-chart';
import { Users, MessageSquare, Zap, TrendingUp } from 'lucide-react';

export default function DashboardPage() {
  const { data: stats } = useQuery({
    queryKey: ['dashboard-stats'],
    queryFn: () => api.analytics.getDashboardStats(),
  });

  return (
    <div className="space-y-6">
      <div>
        <h1 className="text-3xl font-bold">Dashboard</h1>
        <p className="text-gray-600">Welcome back! Here's your overview.</p>
      </div>

      {/* Stats Grid */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
        <StatsCard
          title="Total Users"
          value={stats?.totalUsers || 0}
          icon={Users}
          trend="+12%"
          trendUp
        />
        <StatsCard
          title="Messages"
          value={stats?.totalMessages || 0}
          icon={MessageSquare}
          trend="+8%"
          trendUp
        />
        <StatsCard
          title="Active Channels"
          value={stats?.activeChannels || 0}
          icon={Zap}
        />
        <StatsCard
          title="Engagement"
          value={`${stats?.engagementRate || 0}%`}
          icon={TrendingUp}
          trend="+5%"
          trendUp
        />
      </div>

      {/* Charts */}
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <UsageChart />
        <ActivityChart />
      </div>
    </div>
  );
}
```

### 3. Team Management

**Create `app/(dashboard)/teams/page.tsx`**:

```typescript
'use client';

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { api } from '@/lib/api-client';
import { Button } from '@/components/ui/button';
import { TeamList } from '@/components/teams/team-list';
import { TeamForm } from '@/components/teams/team-form';
import { Dialog, DialogContent, DialogTrigger } from '@/components/ui/dialog';
import { Plus } from 'lucide-react';
import { useState } from 'react';

export default function TeamsPage() {
  const [isCreateOpen, setIsCreateOpen] = useState(false);
  const queryClient = useQueryClient();

  const { data: teams, isLoading } = useQuery({
    queryKey: ['teams'],
    queryFn: () => api.teams.list(),
  });

  const createTeam = useMutation({
    mutationFn: (data: any) => api.teams.create(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['teams'] });
      setIsCreateOpen(false);
    },
  });

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <div>
          <h1 className="text-3xl font-bold">Teams</h1>
          <p className="text-gray-600">Manage your workspaces and teams</p>
        </div>
        <Dialog open={isCreateOpen} onOpenChange={setIsCreateOpen}>
          <DialogTrigger asChild>
            <Button>
              <Plus className="mr-2 h-4 w-4" />
              Create Team
            </Button>
          </DialogTrigger>
          <DialogContent>
            <TeamForm
              onSubmit={(data) => createTeam.mutate(data)}
              isLoading={createTeam.isPending}
            />
          </DialogContent>
        </Dialog>
      </div>

      {isLoading ? (
        <div>Loading...</div>
      ) : (
        <TeamList teams={teams || []} />
      )}
    </div>
  );
}
```

### 4. API Keys Management

**Create `app/(dashboard)/api-keys/page.tsx`**:

```typescript
'use client';

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { api } from '@/lib/api-client';
import { Button } from '@/components/ui/button';
import { Card } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Key, Copy, Trash2, Eye, EyeOff } from 'lucide-react';
import { useState } from 'react';
import { toast } from '@/components/ui/use-toast';

export default function APIKeysPage() {
  const [showKeys, setShowKeys] = useState<Record<string, boolean>>({});
  const queryClient = useQueryClient();

  const { data: apiKeys } = useQuery({
    queryKey: ['api-keys'],
    queryFn: () => api.apiKeys.list(),
  });

  const generateKey = useMutation({
    mutationFn: (data: { name: string; permissions: string[] }) =>
      api.apiKeys.generate(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['api-keys'] });
      toast({ title: 'API key generated successfully' });
    },
  });

  const revokeKey = useMutation({
    mutationFn: (id: string) => api.apiKeys.revoke(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['api-keys'] });
      toast({ title: 'API key revoked' });
    },
  });

  const copyToClipboard = (key: string) => {
    navigator.clipboard.writeText(key);
    toast({ title: 'Copied to clipboard' });
  };

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <div>
          <h1 className="text-3xl font-bold">API Keys</h1>
          <p className="text-gray-600">Manage your API keys for integration</p>
        </div>
        <Button onClick={() => generateKey.mutate({ name: 'New Key', permissions: ['read', 'write'] })}>
          <Key className="mr-2 h-4 w-4" />
          Generate New Key
        </Button>
      </div>

      <div className="grid gap-4">
        {apiKeys?.map((key: any) => (
          <Card key={key.id} className="p-6">
            <div className="flex items-center justify-between">
              <div className="flex-1">
                <div className="flex items-center gap-3 mb-2">
                  <h3 className="font-semibold">{key.name}</h3>
                  <Badge variant={key.isActive ? 'default' : 'secondary'}>
                    {key.isActive ? 'Active' : 'Revoked'}
                  </Badge>
                </div>
                <div className="flex items-center gap-2 font-mono text-sm">
                  <code className="bg-gray-100 px-2 py-1 rounded">
                    {showKeys[key.id] ? key.key : '••••••••••••••••'}
                  </code>
                  <Button
                    variant="ghost"
                    size="sm"
                    onClick={() => setShowKeys(prev => ({ ...prev, [key.id]: !prev[key.id] }))}
                  >
                    {showKeys[key.id] ? <EyeOff className="h-4 w-4" /> : <Eye className="h-4 w-4" />}
                  </Button>
                  <Button
                    variant="ghost"
                    size="sm"
                    onClick={() => copyToClipboard(key.key)}
                  >
                    <Copy className="h-4 w-4" />
                  </Button>
                </div>
                <p className="text-sm text-gray-500 mt-2">
                  Created {new Date(key.createdAt).toLocaleDateString()}
                </p>
              </div>
              <Button
                variant="destructive"
                size="sm"
                onClick={() => revokeKey.mutate(key.id)}
                disabled={!key.isActive}
              >
                <Trash2 className="h-4 w-4" />
              </Button>
            </div>
          </Card>
        ))}
      </div>
    </div>
  );
}
```

### 5. Webhooks Configuration

**Create `app/(dashboard)/webhooks/page.tsx`**:

```typescript
'use client';

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { api } from '@/lib/api-client';
import { Button } from '@/components/ui/button';
import { WebhookList } from '@/components/webhooks/webhook-list';
import { WebhookForm } from '@/components/webhooks/webhook-form';
import { Dialog, DialogContent, DialogTrigger } from '@/components/ui/dialog';
import { Webhook, Plus } from 'lucide-react';
import { useState } from 'react';

export default function WebhooksPage() {
  const [isCreateOpen, setIsCreateOpen] = useState(false);
  const queryClient = useQueryClient();

  const { data: webhooks } = useQuery({
    queryKey: ['webhooks'],
    queryFn: () => api.webhooks.list(),
  });

  const createWebhook = useMutation({
    mutationFn: (data: any) => api.webhooks.create(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['webhooks'] });
      setIsCreateOpen(false);
    },
  });

  const testWebhook = useMutation({
    mutationFn: (id: string) => api.webhooks.test(id),
  });

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <div>
          <h1 className="text-3xl font-bold">Webhooks</h1>
          <p className="text-gray-600">Configure webhook endpoints for events</p>
        </div>
        <Dialog open={isCreateOpen} onOpenChange={setIsCreateOpen}>
          <DialogTrigger asChild>
            <Button>
              <Plus className="mr-2 h-4 w-4" />
              Add Webhook
            </Button>
          </DialogTrigger>
          <DialogContent className="max-w-2xl">
            <WebhookForm
              onSubmit={(data) => createWebhook.mutate(data)}
              isLoading={createWebhook.isPending}
            />
          </DialogContent>
        </Dialog>
      </div>

      <WebhookList
        webhooks={webhooks || []}
        onTest={(id) => testWebhook.mutate(id)}
      />
    </div>
  );
}
```

### 6. Analytics Dashboard

**Create `app/(dashboard)/analytics/page.tsx`**:

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';
import { api } from '@/lib/api-client';
import { Card } from '@/components/ui/card';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Select } from '@/components/ui/select';
import { UsageChart } from '@/components/analytics/usage-chart';
import { ActivityChart } from '@/components/analytics/activity-chart';
import { MetricsGrid } from '@/components/analytics/metrics-grid';
import { useState } from 'react';

export default function AnalyticsPage() {
  const [timeRange, setTimeRange] = useState('7d');

  const { data: analytics } = useQuery({
    queryKey: ['analytics', timeRange],
    queryFn: () => api.analytics.getStats(timeRange),
  });

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <div>
          <h1 className="text-3xl font-bold">Analytics</h1>
          <p className="text-gray-600">Track usage and engagement metrics</p>
        </div>
        <Select value={timeRange} onValueChange={setTimeRange}>
          <option value="24h">Last 24 hours</option>
          <option value="7d">Last 7 days</option>
          <option value="30d">Last 30 days</option>
          <option value="90d">Last 90 days</option>
        </Select>
      </div>

      <MetricsGrid data={analytics} />

      <Tabs defaultValue="usage" className="space-y-4">
        <TabsList>
          <TabsTrigger value="usage">Usage</TabsTrigger>
          <TabsTrigger value="activity">Activity</TabsTrigger>
          <TabsTrigger value="engagement">Engagement</TabsTrigger>
        </TabsList>

        <TabsContent value="usage" className="space-y-4">
          <Card className="p-6">
            <h3 className="text-lg font-semibold mb-4">Message Volume</h3>
            <UsageChart data={analytics?.usage} />
          </Card>
        </TabsContent>

        <TabsContent value="activity" className="space-y-4">
          <Card className="p-6">
            <h3 className="text-lg font-semibold mb-4">User Activity</h3>
            <ActivityChart data={analytics?.activity} />
          </Card>
        </TabsContent>

        <TabsContent value="engagement" className="space-y-4">
          <Card className="p-6">
            <h3 className="text-lg font-semibold mb-4">Engagement Metrics</h3>
            {/* Engagement charts */}
          </Card>
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

---

## API Client Updates

**Update `lib/api-client.ts`** with all portal endpoints:

```typescript
export const api = {
  // ... existing auth endpoints ...
  
  teams: {
    list: () => apiClient.get('/teams'),
    get: (id: string) => apiClient.get(`/teams/${id}`),
    create: (data: any) => apiClient.post('/teams', data),
    update: (id: string, data: any) => apiClient.patch(`/teams/${id}`, data),
    delete: (id: string) => apiClient.delete(`/teams/${id}`),
    members: (id: string) => apiClient.get(`/teams/${id}/members`),
    inviteMember: (id: string, data: any) => apiClient.post(`/teams/${id}/invite`, data),
  },
  
  apiKeys: {
    list: () => apiClient.get('/api-keys'),
    generate: (data: any) => apiClient.post('/api-keys', data),
    revoke: (id: string) => apiClient.delete(`/api-keys/${id}`),
  },
  
  webhooks: {
    list: () => apiClient.get('/webhooks'),
    get: (id: string) => apiClient.get(`/webhooks/${id}`),
    create: (data: any) => apiClient.post('/webhooks', data),
    update: (id: string, data: any) => apiClient.patch(`/webhooks/${id}`, data),
    delete: (id: string) => apiClient.delete(`/webhooks/${id}`),
    test: (id: string) => apiClient.post(`/webhooks/${id}/test`),
    logs: (id: string) => apiClient.get(`/webhooks/${id}/logs`),
  },
  
  analytics: {
    getDashboardStats: () => apiClient.get('/analytics/dashboard'),
    getStats: (timeRange: string) => apiClient.get(`/analytics/stats?range=${timeRange}`),
    getUsage: (timeRange: string) => apiClient.get(`/analytics/usage?range=${timeRange}`),
  },
};
```

---

## Summary

The User Portal provides a complete self-service interface for Chat SDK customers with:

✅ Modern Next.js 14 architecture  
✅ Beautiful UI with shadcn/ui  
✅ Real-time data with React Query  
✅ Comprehensive team management  
✅ API key generation and management  
✅ Webhook configuration and testing  
✅ Analytics and usage tracking  
✅ Billing integration ready  

**Next**: Integrate with backend APIs from Month 6 implementation.
