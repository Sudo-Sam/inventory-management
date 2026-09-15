<template>
  <div class="app-shell">
    <aside class="sidebar" :class="{ collapsed: sidebarCollapsed }">
      <div class="sidebar-brand">
        <h1 v-if="!sidebarCollapsed">{{ t("nav.companyName") }}</h1>
        <span v-else class="brand-mark">{{
          t("nav.companyName").charAt(0)
        }}</span>
      </div>

      <nav class="sidebar-nav">
        <router-link
          v-for="item in navItems"
          :key="item.path"
          :to="item.path"
          class="sidebar-nav-item"
          :class="{ active: $route.path === item.path }"
          :title="sidebarCollapsed ? t(item.label) : null"
        >
          <svg class="nav-icon" viewBox="0 0 24 24" width="20" height="20">
            <path
              :d="getIcon(item.icon)"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
            />
          </svg>
          <span v-if="!sidebarCollapsed" class="nav-label">{{
            t(item.label)
          }}</span>
        </router-link>
      </nav>

      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
        <button
          class="sidebar-toggle"
          @click="toggleSidebar"
          :title="sidebarCollapsed ? 'Expand' : 'Collapse'"
        >
          <svg
            viewBox="0 0 24 24"
            width="18"
            height="18"
            :style="{ transform: sidebarCollapsed ? 'rotate(180deg)' : 'none' }"
          >
            <path
              d="M15 18l-6-6 6-6"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
            />
          </svg>
        </button>
      </div>
    </aside>

    <div class="app-content">
      <FilterBar />
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";
import { api } from "./api";
import { useAuth } from "./composables/useAuth";
import { useI18n } from "./composables/useI18n";
import FilterBar from "./components/FilterBar.vue";
import ProfileMenu from "./components/ProfileMenu.vue";
import ProfileDetailsModal from "./components/ProfileDetailsModal.vue";
import TasksModal from "./components/TasksModal.vue";
import LanguageSwitcher from "./components/LanguageSwitcher.vue";

const { currentUser } = useAuth();
const { t } = useI18n();

const showProfileDetails = ref(false);
const showTasks = ref(false);
const apiTasks = ref([]);
const sidebarCollapsed = ref(
  localStorage.getItem("sidebarCollapsed") === "true",
);

const navItems = [
  { path: "/", label: "nav.overview", icon: "grid" },
  { path: "/inventory", label: "nav.inventory", icon: "box" },
  { path: "/orders", label: "nav.orders", icon: "clipboard" },
  { path: "/spending", label: "nav.finance", icon: "dollar" },
  { path: "/demand", label: "nav.demandForecast", icon: "trending-up" },
  { path: "/reports", label: "nav.reports", icon: "bar-chart" },
  { path: "/backlog", label: "nav.backlog", icon: "list" },
];

const iconPaths = {
  grid: "M3 3h8v8H3V3zm10 0h8v8h-8V3zM3 13h8v8H3v-8zm10 0h8v8h-8v-8z",
  box: "M3 3h18v5H3V3zm0 7h18v9H3v-9zm0 11h18v2H3v-2z",
  clipboard: "M7 3h10v2H7V3zm0 4h10v13H7V7zm2-2h6V2h-6v1z",
  dollar:
    "M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 18c-4.41 0-8-3.59-8-8s3.59-8 8-8 8 3.59 8 8-3.59 8-8 8zm3.5-9c.83 0 1.5-.67 1.5-1.5S16.33 8 15.5 8 14 8.67 14 9.5s.67 1.5 1.5 1.5zm-7 0c.83 0 1.5-.67 1.5-1.5S9.33 8 8.5 8 7 8.67 7 9.5 7.67 11 8.5 11zm3.5 6.5c2.33 0 4.31-1.46 5.11-3.5H6.89c.8 2.04 2.78 3.5 5.11 3.5z",
  "trending-up":
    "M16 6l2.29 2.29-4.88 4.88-4-4L2 16.59 3.41 18 9 12.41l4 4 6.3-6.29L22 12v-6z",
  "bar-chart": "M5 9.2h3V19H5zM10.6 5h2.8v14h-2.8zm5.6 8H19v6h-2.8z",
  list: "M3 13h2v-2H3v2zm0 4h2v-2H3v2zm0-8h2V7H3v2zm4 4h14v-2H7v2zm0 4h14v-2H7v2zM7 7v2h14V7H7z",
};

const getIcon = (name) => iconPaths[name] || iconPaths.grid;

const toggleSidebar = () => {
  sidebarCollapsed.value = !sidebarCollapsed.value;
  localStorage.setItem("sidebarCollapsed", sidebarCollapsed.value);
};

// Merge mock tasks from currentUser with API tasks
const tasks = computed(() => {
  return [...currentUser.value.tasks, ...apiTasks.value];
});

const loadTasks = async () => {
  try {
    apiTasks.value = await api.getTasks();
  } catch (err) {
    console.error("Failed to load tasks:", err);
  }
};

const addTask = async (taskData) => {
  try {
    const newTask = await api.createTask(taskData);
    // Add new task to the beginning of the array
    apiTasks.value.unshift(newTask);
  } catch (err) {
    console.error("Failed to add task:", err);
  }
};

const deleteTask = async (taskId) => {
  try {
    // Check if it's a mock task (from currentUser)
    const isMockTask = currentUser.value.tasks.some((t) => t.id === taskId);

    if (isMockTask) {
      // Remove from mock tasks
      const index = currentUser.value.tasks.findIndex((t) => t.id === taskId);
      if (index !== -1) {
        currentUser.value.tasks.splice(index, 1);
      }
    } else {
      // Remove from API tasks
      await api.deleteTask(taskId);
      apiTasks.value = apiTasks.value.filter((t) => t.id !== taskId);
    }
  } catch (err) {
    console.error("Failed to delete task:", err);
  }
};

const toggleTask = async (taskId) => {
  try {
    // Check if it's a mock task (from currentUser)
    const mockTask = currentUser.value.tasks.find((t) => t.id === taskId);

    if (mockTask) {
      // Toggle mock task status
      mockTask.status = mockTask.status === "pending" ? "completed" : "pending";
    } else {
      // Toggle API task
      const updatedTask = await api.toggleTask(taskId);
      const index = apiTasks.value.findIndex((t) => t.id === taskId);
      if (index !== -1) {
        apiTasks.value[index] = updatedTask;
      }
    }
  } catch (err) {
    console.error("Failed to toggle task:", err);
  }
};

onMounted(loadTasks);
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family:
    "Inter",
    -apple-system,
    BlinkMacSystemFont,
    "Segoe UI",
    Roboto,
    Oxygen,
    Ubuntu,
    Cantarell,
    sans-serif;
  background: #f8fafc;
  color: #1e293b;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.app-shell {
  display: flex;
  min-height: 100vh;
  background: #f8fafc;
}

.sidebar {
  width: 240px;
  background: #ffffff;
  border-right: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  transition: width 0.2s ease;
  flex-shrink: 0;
  position: sticky;
  top: 0;
  height: 100vh;
  z-index: 100;
}

.sidebar.collapsed {
  width: 64px;
}

.sidebar-brand {
  padding: 1.5rem 1.25rem;
  border-bottom: 1px solid #f1f5f9;
  display: flex;
  align-items: center;
  justify-content: center;
}

.sidebar-brand h1 {
  margin: 0;
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  text-align: center;
}

.brand-mark {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  padding: 1rem 0.75rem;
  flex: 1;
  overflow-y: auto;
}

.sidebar-nav-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.625rem 0.75rem;
  border-radius: 6px;
  border-left: 3px solid transparent;
  color: #475569;
  text-decoration: none;
  font-weight: 500;
  font-size: 0.938rem;
  transition: all 0.2s ease;
}

.sidebar-nav-item:hover {
  color: #334155;
  background: #f8fafc;
}

.sidebar-nav-item.active {
  color: #2563eb;
  background: #eff6ff;
  border-left-color: #2563eb;
  font-weight: 600;
}

.nav-icon {
  flex-shrink: 0;
  color: currentColor;
}

.nav-label {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.sidebar.collapsed .sidebar-nav-item {
  justify-content: center;
  padding: 0.625rem;
}

.sidebar-footer {
  padding: 1rem 0.75rem;
  border-top: 1px solid #f1f5f9;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.sidebar-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0.5rem;
  border-radius: 6px;
  border: 1px solid #e2e8f0;
  background: #ffffff;
  color: #64748b;
  cursor: pointer;
  transition: all 0.2s ease;
}

.sidebar-toggle:hover {
  background: #f8fafc;
  border-color: #cbd5e1;
  color: #0f172a;
}

.app-content {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
}

.main-content {
  flex: 1;
  width: 100%;
  padding: 1.5rem 2rem;
}

.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.375rem;
  letter-spacing: -0.025em;
}

.page-header p {
  color: #64748b;
  font-size: 0.938rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: white;
  padding: 1.25rem;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
  transition: all 0.2s ease;
}

.stat-card:hover {
  border-color: #cbd5e1;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.06);
}

.stat-label {
  color: #64748b;
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.stat-card.warning .stat-value {
  color: #ea580c;
}

.stat-card.success .stat-value {
  color: #059669;
}

.stat-card.danger .stat-value {
  color: #dc2626;
}

.stat-card.info .stat-value {
  color: #2563eb;
}

.card {
  background: white;
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid #e2e8f0;
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
  border-bottom: 1px solid #e2e8f0;
}

th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  font-weight: 600;
  color: #475569;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: 0.5rem 0.75rem;
  border-top: 1px solid #f1f5f9;
  color: #334155;
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: #f8fafc;
}

.badge {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success {
  background: #d1fae5;
  color: #065f46;
}

.badge.warning {
  background: #fed7aa;
  color: #92400e;
}

.badge.danger {
  background: #fecaca;
  color: #991b1b;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.badge.decreasing {
  background: #fecaca;
  color: #991b1b;
}

.badge.stable {
  background: #e0e7ff;
  color: #3730a3;
}

.badge.high {
  background: #fecaca;
  color: #991b1b;
}

.badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.badge.low {
  background: #dbeafe;
  color: #1e40af;
}

.loading {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}
</style>
