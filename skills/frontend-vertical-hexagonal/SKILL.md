---
name: frontend-vertical-hexagonal
description: Enforces a Vertical Slice / Feature-First architecture with an internal Hexagonal Architecture (Ports and Adapters) structure using React, Next.js, Zustand, React Query, and Axios.
---

# Frontend Vertical-Hexagonal Architecture (CodelyTV Style)

This skill enforces a **Feature-First / Vertical Slice** approach on the outside, and a strict **Hexagonal Architecture (Ports & Adapters)** on the inside of each module, styled exactly after CodelyTV's frontend-hexagonal guidelines and validated via `eslint-plugin-hexagonal-architecture`.

---

## 1. Core Philosophy & ESLint Layer Boundaries

*   **Screaming Architecture**: The code structure shouts what the app does (e.g. `courses`, `tasks`), separating the React framework UI concern from core business logic.
*   **Dependency Inversion**: Outer layers (infrastructure adapters, UI views) depend on the inner layers (domain and application). Inner layers depend on nothing.
*   **Mandatory ESLint Rule Boundaries**: The project must install and configure `eslint-plugin-hexagonal-architecture`.
    *   **Domain (`domain/`)**: Can only import from within the same `domain/` folder or shared domain utilities. It has no external dependencies.
    *   **Application (`application/`)**: Can only import from `domain/` and other `application/` cases. It cannot import from `infrastructure/` or presentation/sections.
    *   **Infrastructure (`infrastructure/`)**: Can import from `domain/`, `application/`, and other `infrastructure/` files.
    *   **Presentation / Sections (`sections/`)**: The React UI layer resides *outside* the modules, allowing components to import from `modules/` without triggering ESLint architecture errors.

### ESLint Configuration (`.eslintrc.js` or equivalent)
```javascript
module.exports = {
  plugins: ["hexagonal-architecture"],
  overrides: [
    {
      files: ["src/modules/**/*.ts", "src/modules/**/*.tsx"],
      rules: {
        "hexagonal-architecture/enforce": ["error"]
      }
    }
  ]
};
```

---

## 2. Directory Blueprint

The app splits logic into `modules/` and React UI into `sections/`:

```text
src/
├── shared/                             # Shared Kernel (Cross-cutting domain and technical concerns)
│   └── domain/
│       └── value-objects/
│           └── Email.ts                # Reusable Value Object across multiple modules
│
├── modules/                            # 1. Core Business Logic (Framework-Agnostic)
│   └── tasks/                          # Vertical Feature Slice
│       ├── domain/                     # Entities, Value Objects & Repository Ports
│       │   ├── Task.ts                 # Task Entity Interface & Coordinator validation
│       │   ├── TaskId.ts               # Value Object: ID Validation
│       │   ├── TaskTitle.ts            # Value Object: Title Validation
│       │   └── TaskRepository.ts       # Repository Port (Interface)
│       ├── application/                # Pure Business Use-Cases (Functions only)
│       │   ├── create/
│       │   │   └── createTask.ts       # Create Use Case Function
│       │   ├── get-all/
│       │   │   └── getAllTasks.ts      # List Use Case Function
│       │   └── delete/
│       │       └── deleteTask.ts       # Delete Use Case Function
│       └── infrastructure/             # Adaptadores (API, Storage, Stores)
│           ├── LocalStorageTaskRepository.ts # LocalStorage Adapter
│           └── AxiosTaskRepository.ts        # Axios API Adapter
│
├── sections/                           # 2. React UI Presentation Layer
│   ├── shared/                         # Shared Design System / Primitives (Buttons, inputs)
│   └── tasks/                          # Tasks-specific components and state
│       ├── TasksContext.tsx            # Context Provider for Dependency Injection
│       ├── TasksList.tsx               # Component for rendering TaskCards
│       ├── TaskCard.tsx                # Card with delete button
│       ├── CreateTaskForm.tsx          # Form for adding a task
│       └── useTaskForm.ts              # Hook handling state and validations of the form
│
└── App.tsx                             # Entry Composition Root (Vite entry) or app/ router shell
```

---

## 3. Communication, State Management & Validation Rules

1.  **Framework Decoupling**: Use cases in `application/` are pure functions. They receive repositories/dependencies as parameters.
2.  **Context-Based Dependency Injection (DI)**: In React, we wire the actual repository implementation inside the Context Provider (e.g. `TasksContextProvider` receives `repository` as a prop). This allows clean mocking in tests.
3.  **Cross-Module Stores**: If state managers like Zustand are used, they live in `infrastructure/` of the module. Cross-module access must be read-only.
4.  **Value Object & Validation Guidelines (Avoid File Explosion)**:
    *   **Co-location / Simple Fields**: For simple validations (e.g., non-empty strings, age ranges), define validation helper functions directly inside the main Entity file (e.g., `User.ts`) to avoid creating dozens of tiny files.
    *   **Shared Value Objects**: Reusable Value Objects (e.g., `Email.ts`, `Dni.ts`, `Phone.ts`) shared by multiple slices (like `User` and `Client`) must be placed in `src/shared/domain/value-objects/`.
    *   **Compound Value Objects**: Group closely related primitive fields (e.g. department, province, district) into a single aggregated interface/file (e.g., `UserAddress.ts`) inside the module's `domain/` directory, along with its validations.
    *   **Coordinator**: The main Entity file (e.g., `User.ts`) exports an `ensure[Entity]IsValid` function that orchestrates all local, shared, and compound validations.

---

## 4. Framework Integration: Vite vs. Next.js

Since Vite and Next.js handle routing differently, we keep the routing folders as thin wrapper shells that import views from the `sections/` directory.

### A. Vite Integration
In Vite, routing resides entirely in the client-side bundle (e.g. `react-router-dom`). The routing config maps URLs directly to modular React views.

#### Vite Directory Layout
```text
src/
├── main.tsx                     # Vite application entry point
├── routes.tsx                   # Central router setup
│
├── modules/                     # Domain & application logic
│
└── sections/
    └── tasks/
        └── views/
            └── TasksView.tsx    # Interactive dashboard view
```

#### Vite `routes.tsx` Example
```tsx
import { BrowserRouter, Routes, Route } from "react-router-dom";
import { createLocalStorageTaskRepository } from "./modules/tasks/infrastructure/LocalStorageTaskRepository";
import { TasksContextProvider } from "./sections/tasks/TasksContext";
import { TasksView } from "./sections/tasks/views/TasksView";

const taskRepository = createLocalStorageTaskRepository();

export function AppRoutes() {
  return (
    <BrowserRouter>
      <Routes>
        <Route
          path="/tasks"
          element={
            <TasksContextProvider repository={taskRepository}>
              <TasksView />
            </TasksContextProvider>
          }
        />
      </Routes>
    </BrowserRouter>
  );
}
```

---

### B. Next.js App Router Integration
In Next.js App Router, routing is governed by the `src/app/` directory. The files `page.tsx` act as thin server/routing shells that instantiate adapters and render the client-side views.

#### Next.js App Router Directory Layout
```text
src/
├── app/                         # Next.js App Router (Thin routing shells)
│   ├── layout.tsx               # Root Layout
│   └── tasks/
│       └── page.tsx             # Server page wrapping the Client View with Provider
│
├── modules/                     # Domain & application logic
│
└── sections/
    └── tasks/
        ├── TasksContext.tsx
        └── views/
            └── TasksView.tsx    # Client view marked with "use client"
```

#### Route Shell `src/app/tasks/page.tsx`
```tsx
import { createLocalStorageTaskRepository } from "@/modules/tasks/infrastructure/LocalStorageTaskRepository";
import { TasksContextProvider } from "@/sections/tasks/TasksContext";
import { TasksView } from "@/sections/tasks/views/TasksView";

export const metadata = {
  title: "Tasks Manager",
};

export default function TasksPage() {
  // Wire up adapter
  const repository = createLocalStorageTaskRepository();

  return (
    <TasksContextProvider repository={repository}>
      <TasksView />
    </TasksContextProvider>
  );
}
```

---

## 5. Performance & React Best Practices

1.  **Avoid Barrel Files (`index.ts`)**: Never use barrel files to re-export files within directories. This breaks bundler tree-shaking, importing dead code.
2.  **Streaming & Suspense**: Wrap heavy or asynchronously loaded modular components in `<Suspense>` to prevent rendering waterfalls.
3.  **Strict Component Isolation**: Keep presentation components lightweight. Extract heavy logic blocks or complex forms into dedicated child files in the same section directory to prevent full-screen re-renders.

---

## 6. Coding Templates: Complete Tasks CRUD

### A. Domain Layer

#### `domain/TaskId.ts`
```typescript
export function isTaskIdValid(id: string): boolean {
  const regexExp = /^[0-9a-fA-F]{8}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{12}$/gi;
  return regexExp.test(id);
}

export function TaskIdNotValidError(id: string): Error {
  return new Error(`Task ID ${id} is not a valid UUID`);
}
```

#### `domain/TaskTitle.ts`
```typescript
export const TITLE_MIN_LENGTH = 3;
export const TITLE_MAX_LENGTH = 150;

export function isTaskTitleValid(title: string): boolean {
  return title.trim().length >= TITLE_MIN_LENGTH && title.trim().length <= TITLE_MAX_LENGTH;
}

export function TaskTitleNotValidError(title: string): Error {
  return new Error(`Task title must be between ${TITLE_MIN_LENGTH} and ${TITLE_MAX_LENGTH} characters`);
}
```

#### `domain/Task.ts`
```typescript
import { isTaskIdValid, TaskIdNotValidError } from "./TaskId";
import { isTaskTitleValid, TaskTitleNotValidError } from "./TaskTitle";

export interface Task {
  id: string;
  title: string;
  completed: boolean;
}

export function ensureTaskIsValid({ id, title }: Task): void {
  if (!isTaskIdValid(id)) {
    throw TaskIdNotValidError(id);
  }
  if (!isTaskTitleValid(title)) {
    throw TaskTitleNotValidError(title);
  }
}
```

#### `domain/TaskRepository.ts`
```typescript
import { Task } from "./Task";

export interface TaskRepository {
  getAll(): Promise<Task[]>;
  save(task: Task): Promise<void>;
  delete(id: string): Promise<void>;
}
```

### B. Application Layer (Pure Use-Cases)

#### `application/create/createTask.ts`
```typescript
import { Task, ensureTaskIsValid } from "../../domain/Task";
import { TaskRepository } from "../../domain/TaskRepository";

export async function createTask(taskRepository: TaskRepository, task: Task): Promise<void> {
  ensureTaskIsValid(task);
  await taskRepository.save(task);
}
```

#### `application/get-all/getAllTasks.ts`
```typescript
import { Task } from "../../domain/Task";
import { TaskRepository } from "../../domain/TaskRepository";

export async function getAllTasks(taskRepository: TaskRepository): Promise<Task[]> {
  return await taskRepository.getAll();
}
```

#### `application/delete/deleteTask.ts`
```typescript
import { TaskRepository } from "../../domain/TaskRepository";

export async function deleteTask(taskRepository: TaskRepository, id: string): Promise<void> {
  await taskRepository.delete(id);
}
```

### C. Infrastructure Layer

#### `infrastructure/LocalStorageTaskRepository.ts`
```typescript
import { Task } from "../domain/Task";
import { TaskRepository } from "../domain/TaskRepository";

const STORAGE_KEY = "tasks";

export function createLocalStorageTaskRepository(): TaskRepository {
  return {
    getAll,
    save,
    delete: deleteTask,
  };
}

async function getAll(): Promise<Task[]> {
  if (typeof window === "undefined") return [];
  const raw = localStorage.getItem(STORAGE_KEY);
  return raw ? JSON.parse(raw) : [];
}

async function save(task: Task): Promise<void> {
  const tasks = await getAll();
  const index = tasks.findIndex((t) => t.id === task.id);
  if (index >= 0) {
    tasks[index] = task;
  } else {
    tasks.push(task);
  }
  localStorage.setItem(STORAGE_KEY, JSON.stringify(tasks));
}

async function deleteTask(id: string): Promise<void> {
  const tasks = await getAll();
  const filtered = tasks.filter((t) => t.id !== id);
  localStorage.setItem(STORAGE_KEY, JSON.stringify(filtered));
}
```

### D. Sections Layer (React UI & DI)

#### `sections/tasks/TasksContext.tsx`
```tsx
import React, { createContext, useContext, useEffect, useState } from "react";
import { Task } from "../../modules/tasks/domain/Task";
import { TaskRepository } from "../../modules/tasks/domain/TaskRepository";
import { getAllTasks } from "../../modules/tasks/application/get-all/getAllTasks";
import { createTask } from "../../modules/tasks/application/create/createTask";
import { deleteTask } from "../../modules/tasks/application/delete/deleteTask";

interface TasksContextState {
  tasks: Task[];
  loading: boolean;
  error: string | null;
  addNewTask: (title: string) => Promise<void>;
  removeTask: (id: string) => Promise<void>;
}

export const TasksContext = createContext({} as TasksContextState);

export const TasksContextProvider = ({
  children,
  repository,
}: React.PropsWithChildren<{ repository: TaskRepository }>) => {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const fetchTasks = async () => {
    try {
      setLoading(true);
      setError(null);
      const data = await getAllTasks(repository);
      setTasks(data);
    } catch (e: any) {
      setError(e.message || "Failed to load tasks");
    } finally {
      setLoading(false);
    }
  };

  const addNewTask = async (title: string) => {
    try {
      setError(null);
      const newTask: Task = {
        id: crypto.randomUUID(),
        title,
        completed: false,
      };
      await createTask(repository, newTask);
      await fetchTasks();
    } catch (e: any) {
      setError(e.message);
    }
  };

  const removeTask = async (id: string) => {
    try {
      setError(null);
      await deleteTask(repository, id);
      await fetchTasks();
    } catch (e: any) {
      setError(e.message || "Failed to delete task");
    }
  };

  useEffect(() => {
    fetchTasks();
  }, []);

  return (
    <TasksContext.Provider value={{ tasks, loading, error, addNewTask, removeTask }}>
      {children}
    </TasksContext.Provider>
  );
};

export const useTasksContext = () => useContext(TasksContext);
```

#### `sections/tasks/TasksList.tsx`
```tsx
import React from "react";
import { useTasksContext } from "./TasksContext";
import { TaskCard } from "./TaskCard";

export function TasksList() {
  const { tasks, loading, error } = useTasksContext();

  if (loading) return <div>Cargando tareas...</div>;
  if (error) return <div style={{ color: "red" }}>Error: {error}</div>;
  if (tasks.length === 0) return <div>No hay tareas pendientes.</div>;

  return (
    <div className="tasks-list">
      {tasks.map((task) => (
        <TaskCard key={task.id} task={task} />
      ))}
    </div>
  );
}
```

#### `sections/tasks/TaskCard.tsx`
```tsx
import React from "react";
import { Task } from "../../modules/tasks/domain/Task";
import { useTasksContext } from "./TasksContext";

export function TaskCard({ task }: { task: Task }) {
  const { removeTask } = useTasksContext();

  return (
    <div className="task-card" style={{ display: "flex", gap: "12px", margin: "8px 0" }}>
      <span>{task.title}</span>
      <button onClick={() => removeTask(task.id)}>Eliminar</button>
    </div>
  );
}
```

#### `sections/tasks/CreateTaskForm.tsx`
```tsx
import React, { useState } from "react";
import { useTasksContext } from "./TasksContext";

export function CreateTaskForm() {
  const { addNewTask } = useTasksContext();
  const [title, setTitle] = useState("");

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!title.trim()) return;
    await addNewTask(title);
    setTitle("");
  };

  return (
    <form onSubmit={handleSubmit} style={{ margin: "16px 0" }}>
      <input
        type="text"
        placeholder="Nueva tarea..."
        value={title}
        onChange={(e) => setTitle(e.target.value)}
      />
      <button type="submit">Agregar</button>
    </form>
  );
}
```

#### `sections/tasks/views/TasksView.tsx`
```tsx
"use client";

import React from "react";
import { CreateTaskForm } from "../CreateTaskForm";
import { TasksList } from "../TasksList";

export function TasksView() {
  return (
    <div className="tasks-view" style={{ padding: "24px", maxWidth: "600px", margin: "0 auto" }}>
      <h1>🍍 Tasks Planner (Codely Hexagonal Style)</h1>
      <CreateTaskForm />
      <TasksList />
    </div>
  );
}
```
