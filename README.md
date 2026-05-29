import React, { useState, useMemo, useCallback } from "react";
import { base44 } from "@/api/base44Client";
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { parseISO, compareAsc, isToday, isFuture } from "date-fns";
import { motion, AnimatePresence } from "framer-motion";
import { Focus } from "lucide-react";
import { toast } from "sonner";

import TopBar from "../components/taskflow/TopBar";
import LeftSidebar from "../components/taskflow/LeftSidebar";
import RightPanel from "../components/taskflow/RightPanel";
import TaskInput from "../components/taskflow/TaskInput";
import TabFilter from "../components/taskflow/TabFilter";
import SortControls from "../components/taskflow/SortControls";
import TaskList from "../components/taskflow/TaskList";
import NotificationBanner from "../components/taskflow/NotificationBanner";
import AnalyticsSection from "../components/taskflow/AnalyticsSection";
import { useTaskReminders } from "../hooks/useTaskReminders";
import { useAuth } from "../lib/AuthContext";

const PRIORITY_ORDER = { High: 0, Medium: 1, Low: 2 };

const VIEW_LABELS = {
  overview: "Overview",
  today: "Today",
  upcoming: "Upcoming",
  all: "All Tasks",
  completed: "Completed",
};

export default function Home() {
  const [search, setSearch] = useState("");
  const [activeView, setActiveView] = useState("overview");
  const [category, setCategory] = useState("All");
  const [sort, setSort] = useState("order");
  const [activeProjectId, setActiveProjectId] = useState(null);
  const [sidebarOpen, setSidebarOpen] = useState(
    typeof window !== 'undefined' ? window.innerWidth >= 768 : true
  );
  const [rightPanelOpen, setRightPanelOpen] = useState(true);
  const [focusMode, setFocusMode] = useState(false);
  const [inputRef, setInputRef] = useState(null);
  const queryClient = useQueryClient();
  const { user } = useAuth();

  const { data: tasks = [] } = useQuery({
    queryKey: ["tasks", user?.email],
    queryFn: () => base44.entities.Task.filter({ created_by: user.email }, "order", 500),
    enabled: !!user?.email,
  });

  const { data: projects = [] } = useQuery({
    queryKey: ["projects", user?.email],
    queryFn: () => base44.entities.Project.filter({ created_by: user.email }, "order", 200),
    enabled: !!user?.email,
  });

  useTaskReminders(tasks);

  const taskQueryKey = ["tasks", user?.email];

  const createMutation = useMutation({
    mutationFn: (data) => base44.entities.Task.create(data),
    onMutate: async (data) => {
      await queryClient.cancelQueries({ queryKey: taskQueryKey });
      const previous = queryClient.getQueryData(taskQueryKey);
      const optimistic = { ...data, id: `temp-${Date.now()}`, created_date: new Date().toISOString() };
      queryClient.setQueryData(taskQueryKey, (old = []) => [...old, optimistic]);
      return { previous };
    },
    onError: (err, _vars, ctx) => {
      queryClient.setQueryData(taskQueryKey, ctx?.previous);
      console.error("Task creation failed:", err);
      toast.error("Failed to save task. Please try again.");
    },
    onSettled: () => queryClient.invalidateQueries({ queryKey: taskQueryKey }),
  });

  const updateMutation = useMutation({
    mutationFn: ({ id, data }) => base44.entities.Task.update(id, data),
    onMutate: async ({ id, data }) => {
      await queryClient.cancelQueries({ queryKey: taskQueryKey });
      const previous = queryClient.getQueryData(taskQueryKey);
      queryClient.setQueryData(taskQueryKey, (old) => old?.map((t) => (t.id === id ? { ...t, ...data } : t)));
      return { previous };
    },
    onError: (_err, _vars, ctx) => queryClient.setQueryData(taskQueryKey, ctx.previous),
    onSettled: () => queryClient.invalidateQueries({ queryKey: taskQueryKey }),
  });

  const deleteMutation = useMutation({
    mutationFn: (id) => base44.entities.Task.delete(id),
    onMutate: async (id) => {
      await queryClient.cancelQueries({ queryKey: taskQueryKey });
      const previous = queryClient.getQueryData(taskQueryKey);
      queryClient.setQueryData(taskQueryKey, (old) => old?.filter((t) => t.id !== id));
      return { previous };
    },
    onError: (_err, _vars, ctx) => queryClient.setQueryData(taskQueryKey, ctx.previous),
    onSettled: () => queryClient.invalidateQueries({ queryKey: taskQueryKey }),
  });

  const handleToggle = useCallback((task) => updateMutation.mutate({ id: task.id, data: { done: !task.done } }), [updateMutation]);
  const handleDelete = useCallback((task) => deleteMutation.mutate(task.id), [deleteMutation]);
  const handleUpdate = useCallback((id, data) => updateMutation.mutate({ id, data }), [updateMutation]);

  const filteredTasks = useMemo(() => {
    let result = [...tasks];

    // Project filter
    if (activeProjectId) result = result.filter((t) => t.project_id === activeProjectId);

    // View-based filter
    if (activeView === "today") result = result.filter((t) => t.due_date && isToday(parseISO(t.due_date)) && !t.done);
    else if (activeView === "upcoming") result = result.filter((t) => t.due_date && isFuture(parseISO(t.due_date)) && !t.done);
    else if (activeView === "completed") result = result.filter((t) => t.done);
    else if (activeView === "all") result = result.filter((t) => !t.done);
    // overview = all tasks (no filter)

    // Category
    if (category !== "All") result = result.filter((t) => t.category === category);

    // Search
    if (search.trim()) {
      const q = search.toLowerCase();
      result = result.filter((t) => t.text.toLowerCase().includes(q) || (t.description || "").toLowerCase().includes(q));
    }

    // Sort
    if (sort === "due_date") {
      result.sort((a, b) => {
        if (!a.due_date && !b.due_date) return 0;
        if (!a.due_date) return 1;
        if (!b.due_date) return -1;
        return compareAsc(parseISO(a.due_date), parseISO(b.due_date));
      });
    } else if (sort === "priority") {
      result.sort((a, b) => (PRIORITY_ORDER[a.priority] ?? 1) - (PRIORITY_ORDER[b.priority] ?? 1));
    } else if (sort === "created_date") {
      result.sort((a, b) => new Date(b.created_date) - new Date(a.created_date));
    } else {
      result.sort((a, b) => (a.order ?? 0) - (b.order ?? 0));
    }
    return result;
  }, [tasks, activeView, category, search, sort, activeProjectId]);

  const handleReorder = useCallback((fromIndex, toIndex) => {
    const reordered = [...filteredTasks];
    const [moved] = reordered.splice(fromIndex, 1);
    reordered.splice(toIndex, 0, moved);
    const updates = reordered.map((task, idx) => ({ id: task.id, data: { order: idx } }));
    queryClient.setQueryData(taskQueryKey, (old) => {
      const mapped = new Map(old.map((t) => [t.id, t]));
      updates.forEach(({ id, data }) => { if (mapped.has(id)) mapped.set(id, { ...mapped.get(id), ...data }); });
      return Array.from(mapped.values()).sort((a, b) => (a.order ?? 0) - (b.order ?? 0));
    });
    updates.forEach(({ id, data }) => base44.entities.Task.update(id, data));
  }, [filteredTasks, queryClient]);

  const activeProject = projects.find((p) => p.id === activeProjectId);
  const pageTitle = activeProject ? activeProject.name : VIEW_LABELS[activeView] || "Tasks";

  // Tab derived from view
  const tabForView = activeView === "completed" ? "completed" : activeView === "all" || activeView === "today" || activeView === "upcoming" ? "active" : "all";

  return (
    <div className="min-h-screen bg-background flex flex-col">
      {/* Top bar */}
      <TopBar
        search={search}
        onSearchChange={setSearch}
        sidebarOpen={sidebarOpen}
        setSidebarOpen={setSidebarOpen}
        tasks={tasks}
      />

      {/* Focus mode overlay */}
      <AnimatePresence>
        {focusMode && (
          <motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }} exit={{ opacity: 0 }}
            className="fixed inset-0 bg-background/95 backdrop-blur-xl z-40 flex flex-col items-center justify-center p-8">
            <div className="max-w-lg w-full space-y-6">
              <div className="text-center mb-8">
                <div className="h-12 w-12 rounded-2xl bg-primary/10 flex items-center justify-center mx-auto mb-3">
                  <Focus className="h-6 w-6 text-primary" />
                </div>
                <h2 className="text-xl font-bold text-foreground">Focus Mode</h2>
                <p className="text-sm text-muted-foreground mt-1">Distractions hidden — stay on task</p>
              </div>
              <TaskList
                tasks={tasks.filter((t) => !t.done && t.priority === "High")}
                onToggle={handleToggle} onDelete={handleDelete} onUpdate={handleUpdate} onReorder={() => {}}
              />
              <button onClick={() => setFocusMode(false)}
                className="w-full py-2.5 text-sm text-muted-foreground hover:text-foreground border border-border rounded-xl transition-colors">
                Exit Focus Mode
              </button>
            </div>
          </motion.div>
        )}
      </AnimatePresence>

      <div className="flex flex-1 overflow-hidden">
        {/* Mobile sidebar drawer (fixed overlay, below md) */}
        <AnimatePresence>
          {sidebarOpen && (
            <motion.div
              key="mobile-overlay"
              initial={{ opacity: 0 }}
              animate={{ opacity: 1 }}
              exit={{ opacity: 0 }}
              transition={{ duration: 0.2 }}
              onClick={() => setSidebarOpen(false)}
              className="fixed inset-0 z-40 bg-black/40 backdrop-blur-sm md:hidden"
            />
          )}
        </AnimatePresence>
        <AnimatePresence>
          {sidebarOpen && (
            <motion.aside
              key="mobile-drawer"
              initial={{ x: -280 }}
              animate={{ x: 0 }}
              exit={{ x: -280 }}
              transition={{ type: "spring", stiffness: 320, damping: 32 }}
              className="fixed top-0 left-0 z-50 w-[260px] h-full bg-card border-r border-border shadow-2xl md:hidden overflow-y-auto"
            >
              <LeftSidebar
                tasks={tasks}
                activeView={activeView}
                onViewChange={(v) => { setActiveView(v); setSidebarOpen(false); }}
                activeProjectId={activeProjectId}
                onSelectProject={(id) => { setActiveProjectId(id); setSidebarOpen(false); }}
              />
            </motion.aside>
          )}
        </AnimatePresence>

        {/* Desktop/tablet sidebar — hidden on mobile */}
        <AnimatePresence initial={false}>
          {sidebarOpen && (
            <motion.aside
              key="sidebar"
              initial={{ width: 0, opacity: 0 }}
              animate={{ width: 240, opacity: 1 }}
              exit={{ width: 0, opacity: 0 }}
              transition={{ type: "spring", stiffness: 300, damping: 30 }}
              className="hidden md:flex flex-shrink-0 border-r border-border bg-card/50 overflow-hidden h-[calc(100vh-64px)] sticky top-16"
            >
              <div className="w-[240px] h-full overflow-y-auto">
                <LeftSidebar
                  tasks={tasks}
                  activeView={activeView}
                  onViewChange={setActiveView}
                  activeProjectId={activeProjectId}
                  onSelectProject={setActiveProjectId}
                />
              </div>
            </motion.aside>
          )}
        </AnimatePresence>

        {/* Main content */}
        <main className="flex-1 min-w-0 overflow-y-auto h-[calc(100vh-64px)]">
          <div className="max-w-3xl mx-auto px-4 sm:px-6 py-4 md:py-6 space-y-4 md:space-y-5">
            <NotificationBanner />

            {/* Page heading */}
            <motion.div initial={{ opacity: 0, y: -8 }} animate={{ opacity: 1, y: 0 }} transition={{ duration: 0.25 }}
              className="flex items-center gap-3">
              {activeProject && <span className="h-3.5 w-3.5 rounded-full flex-shrink-0" style={{ backgroundColor: activeProject.color }} />}
              <h1 className="text-xl md:text-2xl font-bold text-foreground tracking-tight">{pageTitle}</h1>
              <span className="text-xs md:text-sm text-muted-foreground font-normal mt-0.5 bg-muted/60 px-2 py-0.5 rounded-full">
                {filteredTasks.length}
              </span>
            </motion.div>

            {/* Task input */}
            <motion.div initial={{ opacity: 0, y: 10 }} animate={{ opacity: 1, y: 0 }} transition={{ duration: 0.3, delay: 0.05 }}>
              <TaskInput
                onAdd={(data) => {
                  if (!data.text?.trim()) return;
                  createMutation.mutate({ ...data, order: tasks.length, project_id: activeProjectId || undefined });
                }}
                defaultCategory={category !== "All" ? category : undefined}
                projects={projects}
              />
            </motion.div>

            {/* Filters + Sort */}
            {tasks.length > 0 && (
              <motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }} transition={{ duration: 0.25 }} className="space-y-2.5">
                <div className="flex flex-col sm:flex-row gap-2 items-start sm:items-center">
                  <TabFilter
                    tab={activeView === "completed" ? "completed" : activeView === "all" || activeView === "today" || activeView === "upcoming" ? "active" : "all"}
                    setTab={(t) => { if (t === "completed") setActiveView("completed"); else if (t === "active") setActiveView("all"); else setActiveView("overview"); }}
                    category={category} setCategory={setCategory} tasks={tasks}
                  />
                  <div className="sm:ml-auto bg-muted/60 rounded-xl px-2 py-1">
                    <SortControls sort={sort} setSort={setSort} />
                  </div>
                </div>
              </motion.div>
            )}

            {/* Analytics (collapsible) */}
            {tasks.length > 0 && (
              <motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }} transition={{ duration: 0.2, delay: 0.1 }}>
                <AnalyticsSection tasks={tasks} />
              </motion.div>
            )}

            {/* Task list */}
            <TaskList
              tasks={filteredTasks}
              onToggle={handleToggle}
              onDelete={handleDelete}
              onUpdate={handleUpdate}
              onReorder={handleReorder}
            />

            <div className="h-10" />
          </div>
        </main>

        {/* Right Panel — desktop only */}
        <AnimatePresence initial={false}>
          {rightPanelOpen && (
            <motion.aside
              key="right-panel"
              initial={{ width: 0, opacity: 0 }}
              animate={{ width: 272, opacity: 1 }}
              exit={{ width: 0, opacity: 0 }}
              transition={{ type: "spring", stiffness: 300, damping: 30 }}
              className="hidden lg:flex flex-shrink-0 border-l border-border bg-card/30 overflow-hidden h-[calc(100vh-64px)] sticky top-16"
            >
              <div className="w-[272px] h-full overflow-y-auto px-3">
                <RightPanel
                  tasks={tasks}
                  onAddTask={() => document.querySelector('input[placeholder="Add a new task…"]')?.focus()}
                  onFocusMode={() => setFocusMode(true)}
                />
              </div>
            </motion.aside>
          )}
        </AnimatePresence>
      </div>
    </div>
  );
}
