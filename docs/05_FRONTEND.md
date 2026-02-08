# 🎨 Frontend: React + Inertia.js + TypeScript

> **Framework:** React 19  
> **Language:** TypeScript (Strict Mode)  
> **Build Tool:** Vite 7  
> **Integration:** Inertia.js

---

## 📁 Структура Frontend

```
app/frontend/
├── entrypoints/
│   ├── application.js          # Vite entry (CSS imports)
│   ├── application.css         # Tailwind + Global styles
│   └── inertia.jsx             # Inertia App setup
├── pages/                      # Inertia Page Components (1:1 с контроллерами)
│   ├── Auth/
│   │   ├── Login.tsx
│   │   └── Register.tsx
│   ├── Dashboard/
│   │   └── Index.tsx           # Employee dashboard
│   ├── Roadmaps/
│   │   ├── Index.tsx           # Каталог roadmaps
│   │   ├── Show.tsx            # Viewer (read-only)
│   │   └── Fork.tsx            # Выбор roadmap для форка
│   └── Organizations/
│       ├── Roadmaps/
│       │   ├── Index.tsx       # Список roadmaps организации
│       │   ├── New.tsx         # Создание roadmap
│       │   └── Edit.tsx        # Редактор roadmap (Manager/Owner)
│       ├── Employees/
│       │   └── Index.tsx       # Матрица навыков
│       └── Settings/
│           └── Index.tsx
├── components/
│   ├── ui/                     # Базовые компоненты (shadcn/ui style)
│   │   ├── Button.tsx
│   │   ├── Card.tsx
│   │   ├── Input.tsx
│   │   ├── Select.tsx
│   │   ├── Modal.tsx
│   │   ├── Tooltip.tsx
│   │   └── Badge.tsx
│   ├── layout/
│   │   ├── Layout.tsx          # Main layout wrapper
│   │   ├── Navbar.tsx
│   │   ├── Sidebar.tsx
│   │   └── MobileMenu.tsx
│   └── roadmap/                # React Flow компоненты
│       ├── RoadmapViewer.tsx   # Wrapper для React Flow (read-only)
│       ├── RoadmapEditor.tsx   # Wrapper для React Flow (editable)
│       ├── SkillNode.tsx       # Custom node component
│       ├── SkillSidebar.tsx    # Details slide-over panel
│       ├── SkillForm.tsx       # CRUD form для навыка
│       ├── PermitForm.tsx      # Форма ввода данных допуска
│       └── AutoLayoutButton.tsx
├── hooks/
│   ├── useCurrentUser.ts
│   ├── useRoadmapLayout.ts     # Dagre auto-layout wrapper
│   └── useToast.ts
├── types/
│   ├── models.ts               # TypeScript interfaces
│   ├── inertia.d.ts            # Inertia типы
│   └── reactflow.d.ts
└── utils/
    ├── graphLayout.ts          # Dagre helper
    ├── dateFormat.ts
    └── cn.ts                   # classnames helper (для Tailwind)
```

---

## 🧩 Ключевые Компоненты

### `Layout.tsx` (Main Layout)

```tsx
// app/frontend/components/layout/Layout.tsx
import { PropsWithChildren } from 'react';
import { usePage } from '@inertiajs/react';
import Navbar from './Navbar';
import Sidebar from './Sidebar';
import { useToast } from '@/hooks/useToast';
import { PageProps } from '@/types/inertia';

export default function Layout({ children }: PropsWithChildren) {
  const { auth, flash } = usePage<PageProps>().props;
  const { toast } = useToast();
  
  // Показываем flash messages через toast
  useEffect(() => {
    if (flash.success) toast.success(flash.success);
    if (flash.error) toast.error(flash.error);
  }, [flash]);
  
  return (
    <div className="min-h-screen bg-gray-50">
      <Navbar user={auth.user} organization={auth.organization} />
      
      <div className="flex">
        {auth.user && <Sidebar />}
        
        <main className="flex-1 p-6">
          {children}
        </main>
      </div>
    </div>
  );
}
```

---

### `RoadmapViewer.tsx` (Read-Only Graph)

```tsx
// app/frontend/components/roadmap/RoadmapViewer.tsx
import { useState, useCallback } from 'react';
import ReactFlow, {
  Node,
  Edge,
  Controls,
  Background,
  MiniMap,
  useNodesState,
  useEdgesState,
} from 'reactflow';
import 'reactflow/dist/style.css';

import SkillNode from './SkillNode';
import SkillSidebar from './SkillSidebar';

const nodeTypes = {
  skillNode: SkillNode,
};

interface Props {
  roadmap: {
    id: number;
    title: string;
    nodes: Node[];
    edges: Edge[];
  };
  userProgress: Record<string, { status: string; expiresAt?: string }>;
}

export default function RoadmapViewer({ roadmap, userProgress }: Props) {
  const [nodes] = useNodesState(roadmap.nodes);
  const [edges] = useEdgesState(roadmap.edges);
  const [selectedSkillId, setSelectedSkillId] = useState<string | null>(null);
  
  const onNodeClick = useCallback((event: any, node: Node) => {
    setSelectedSkillId(node.id);
  }, []);
  
  // Раскрашиваем узлы по статусу
  const nodesWithStatus = nodes.map(node => {
    const progress = userProgress[node.id];
    const status = progress?.status || 'todo';
    
    return {
      ...node,
      data: {
        ...node.data,
        status,
        expiresAt: progress?.expiresAt,
      },
    };
  });
  
  return (
    <div className="relative h-[800px] w-full">
      <ReactFlow
        nodes={nodesWithStatus}
        edges={edges}
        nodeTypes={nodeTypes}
        onNodeClick={onNodeClick}
        fitView
        minZoom={0.5}
        maxZoom={1.5}
      >
        <Controls />
        <Background />
        <MiniMap />
      </ReactFlow>
      
      {selectedSkillId && (
        <SkillSidebar
          skillId={selectedSkillId}
          progress={userProgress[selectedSkillId]}
          onClose={() => setSelectedSkillId(null)}
        />
      )}
    </div>
  );
}
```

---

### `SkillNode.tsx` (Custom Node)

```tsx
// app/frontend/components/roadmap/SkillNode.tsx
import { memo } from 'react';
import { Handle, Position, NodeProps } from 'reactflow';
import { cn } from '@/utils/cn';
import { CheckCircle, Circle, Clock, AlertTriangle } from 'lucide-react';

const SkillNode = ({ data }: NodeProps) => {
  const { label, status, skillType, category, expiresAt } = data;
  
  // Цвета по статусу
  const statusColors = {
    todo: 'bg-gray-100 border-gray-300',
    in_progress: 'bg-yellow-100 border-yellow-400',
    completed: 'bg-green-100 border-green-400',
    expired: 'bg-red-100 border-red-400',
    expiring_soon: 'bg-orange-100 border-orange-400',
  };
  
  // Иконка по статусу
  const StatusIcon = {
    todo: Circle,
    in_progress: Clock,
    completed: CheckCircle,
    expired: AlertTriangle,
    expiring_soon: AlertTriangle,
  }[status] || Circle;
  
  const isPermit = skillType === 'permit';
  
  return (
    <div
      className={cn(
        'px-4 py-3 rounded-lg border-2 shadow-sm min-w-[200px]',
        statusColors[status]
      )}
    >
      <Handle type="target" position={Position.Top} />
      
      <div className="flex items-start gap-2">
        <StatusIcon className="w-5 h-5 mt-0.5 flex-shrink-0" />
        
        <div className="flex-1 min-w-0">
          <div className="font-medium text-sm leading-tight">
            {label}
          </div>
          
          {isPermit && (
            <div className="text-xs text-gray-600 mt-1">
              Допуск
            </div>
          )}
          
          {category && (
            <div className="text-xs text-gray-500 mt-1">
              {category}
            </div>
          )}
          
          {expiresAt && new Date(expiresAt) < new Date() && (
            <div className="text-xs text-red-600 font-medium mt-1">
              Истек
            </div>
          )}
        </div>
      </div>
      
      <Handle type="source" position={Position.Bottom} />
    </div>
  );
};

export default memo(SkillNode);
```

---

### `SkillSidebar.tsx` (Details Panel)

```tsx
// app/frontend/components/roadmap/SkillSidebar.tsx
import { useEffect, useState } from 'react';
import { router } from '@inertiajs/react';
import { X } from 'lucide-react';
import Button from '@/components/ui/Button';
import PermitForm from './PermitForm';

interface Props {
  skillId: string;
  progress?: { status: string; expiresAt?: string };
  onClose: () => void;
}

export default function SkillSidebar({ skillId, progress, onClose }: Props) {
  const [skill, setSkill] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    // Fetch skill details (можно кэшировать в props)
    fetch(`/api/skills/${skillId}`)
      .then(res => res.json())
      .then(data => {
        setSkill(data);
        setLoading(false);
      });
  }, [skillId]);
  
  const handleMarkCompleted = () => {
    router.post(`/progress/${skillId}`, { status: 'completed' }, {
      preserveScroll: true,
      onSuccess: () => onClose(),
    });
  };
  
  if (loading) return <div>Загрузка...</div>;
  if (!skill) return null;
  
  return (
    <div className="absolute top-0 right-0 w-96 h-full bg-white shadow-xl border-l overflow-y-auto">
      <div className="sticky top-0 bg-white border-b p-4 flex items-center justify-between">
        <h3 className="font-semibold text-lg">{skill.title}</h3>
        <button onClick={onClose} className="p-1 hover:bg-gray-100 rounded">
          <X className="w-5 h-5" />
        </button>
      </div>
      
      <div className="p-4 space-y-4">
        {/* Описание */}
        {skill.description && (
          <div>
            <h4 className="font-medium text-sm text-gray-700 mb-1">Описание</h4>
            <p className="text-sm text-gray-600">{skill.description}</p>
          </div>
        )}
        
        {/* Ресурсы */}
        {skill.resources && skill.resources.length > 0 && (
          <div>
            <h4 className="font-medium text-sm text-gray-700 mb-2">Материалы</h4>
            <ul className="space-y-1">
              {skill.resources.map((resource, index) => (
                <li key={index}>
                  <a
                    href={resource.url}
                    target="_blank"
                    rel="noopener noreferrer"
                    className="text-sm text-blue-600 hover:underline"
                  >
                    {resource.title}
                  </a>
                </li>
              ))}
            </ul>
          </div>
        )}
        
        {/* Действия */}
        <div className="pt-4 border-t">
          {skill.skillType === 'permit' ? (
            <PermitForm skill={skill} onSuccess={onClose} />
          ) : (
            <div className="space-y-2">
              {progress?.status !== 'completed' ? (
                <>
                  <Button
                    onClick={() => router.post(`/progress/${skillId}`, { status: 'in_progress' })}
                    variant="outline"
                    fullWidth
                  >
                    Начать изучение
                  </Button>
                  <Button
                    onClick={handleMarkCompleted}
                    variant="primary"
                    fullWidth
                  >
                    Отметить изученным
                  </Button>
                </>
              ) : (
                <div className="text-center text-green-600 font-medium">
                  ✓ Изучено
                </div>
              )}
            </div>
          )}
        </div>
      </div>
    </div>
  );
}
```

---

### `PermitForm.tsx` (Форма Допуска)

```tsx
// app/frontend/components/roadmap/PermitForm.tsx
import { useForm } from '@inertiajs/react';
import Button from '@/components/ui/Button';
import Input from '@/components/ui/Input';

interface Props {
  skill: any;
  onSuccess: () => void;
}

export default function PermitForm({ skill, onSuccess }: Props) {
  const { data, setData, post, processing, errors } = useForm({
    certificate_number: '',
    issued_at: '',
    issuing_authority: '',
  });
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    
    post(`/progress/${skill.id}`, {
      preserveScroll: true,
      onSuccess,
    });
  };
  
  return (
    <form onSubmit={handleSubmit} className="space-y-3">
      <h4 className="font-medium text-sm text-gray-700 mb-2">
        Данные допуска
      </h4>
      
      <Input
        label="Номер удостоверения"
        value={data.certificate_number}
        onChange={e => setData('certificate_number', e.target.value)}
        error={errors.certificate_number}
        required
      />
      
      <Input
        type="date"
        label="Дата выдачи"
        value={data.issued_at}
        onChange={e => setData('issued_at', e.target.value)}
        error={errors.issued_at}
        required
      />
      
      <Input
        label="Кем выдано"
        value={data.issuing_authority}
        onChange={e => setData('issuing_authority', e.target.value)}
        error={errors.issuing_authority}
        placeholder="Ростехнадзор МРО №1"
      />
      
      <div className="text-xs text-gray-500">
        Срок действия: {skill.permitTemplate.expiration_months} мес.
      </div>
      
      <Button
        type="submit"
        variant="primary"
        fullWidth
        disabled={processing}
      >
        {processing ? 'Сохранение...' : 'Сохранить'}
      </Button>
    </form>
  );
}
```

---

## 🎨 Tailwind Configuration

```js
// tailwind.config.js
export default {
  content: [
    './app/frontend/**/*.{js,jsx,ts,tsx}',
    './app/views/**/*.html.erb',
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
        },
      },
    },
  },
  plugins: [],
};
```

---

## 📦 package.json (Dependencies)

```json
{
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "bin/vite dev",
    "build": "bin/vite build",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "@inertiajs/react": "^2.3.13",
    "@vitejs/plugin-react": "^5.1.3",
    "react": "^19.2.4",
    "react-dom": "^19.2.4",
    "reactflow": "^11.11.0",
    "dagre": "^0.8.5",
    "@headlessui/react": "^2.0.0",
    "lucide-react": "^0.300.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.2.0",
    "date-fns": "^3.0.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@types/dagre": "^0.7.52",
    "autoprefixer": "^10.4.24",
    "postcss": "^8.5.6",
    "tailwindcss": "^3.4.19",
    "typescript": "^5.3.0",
    "vite": "^7.3.1",
    "vite-plugin-ruby": "^5.1.2"
  }
}
```

---

## 📘 TypeScript Типы

```ts
// app/frontend/types/models.ts
export interface User {
  id: number;
  email: string;
  full_name: string;
  role: 'employee' | 'manager' | 'owner';
}

export interface Organization {
  id: number;
  name: string;
  slug: string;
  plan_type: 'trial' | 'starter' | 'professional' | 'enterprise';
}

export interface Skill {
  id: number;
  key: string;
  title: string;
  description?: string;
  skill_type: 'skill' | 'permit' | 'milestone';
  category_label?: string;
  category_color?: string;
  permit_template?: PermitTemplate;
  resources?: Resource[];
}

export interface PermitTemplate {
  id: number;
  title: string;
  code: string;
  expiration_months: number;
  country_code: string;
}

export interface UserProgress {
  id: number;
  skill_id: number;
  status: 'todo' | 'in_progress' | 'completed' | 'expired' | 'expiring_soon';
  certificate_number?: string;
  issued_at?: string;
  expires_at?: string;
}

export interface Roadmap {
  id: number;
  title: string;
  slug: string;
  description?: string;
  visibility: 'public' | 'private' | 'unlisted';
  nodes: Node[];
  edges: Edge[];
}

export interface Resource {
  title: string;
  url: string;
  type: 'video' | 'article' | 'document';
}
```

```ts
// app/frontend/types/inertia.d.ts
import { User, Organization } from './models';

export interface PageProps {
  auth: {
    user: User | null;
    organization: Organization | null;
  };
  flash: {
    success?: string;
    error?: string;
  };
}
```

---

## 🎣 Custom Hooks

```ts
// app/frontend/hooks/useRoadmapLayout.ts
import dagre from 'dagre';
import { Node, Edge } from 'reactflow';

const NODE_WIDTH = 200;
const NODE_HEIGHT = 80;

export function useRoadmapLayout() {
  const calculateLayout = (nodes: Node[], edges: Edge[]) => {
    const dagreGraph = new dagre.graphlib.Graph();
    dagreGraph.setDefaultEdgeLabel(() => ({}));
    dagreGraph.setGraph({
      rankdir: 'TB',
      ranksep: 100,
      nodesep: 50,
    });
    
    nodes.forEach(node => {
      dagreGraph.setNode(node.id, { width: NODE_WIDTH, height: NODE_HEIGHT });
    });
    
    edges.forEach(edge => {
      dagreGraph.setEdge(edge.source, edge.target);
    });
    
    dagre.layout(dagreGraph);
    
    const layoutedNodes = nodes.map(node => {
      const position = dagreGraph.node(node.id);
      return {
        ...node,
        position: {
          x: position.x - NODE_WIDTH / 2,
          y: position.y - NODE_HEIGHT / 2,
        },
      };
    });
    
    return { nodes: layoutedNodes, edges };
  };
  
  return { calculateLayout };
}
```

---

**Следующий документ:** `06_FEATURES.md`
