<template>
  <div class="app">
    <header class="top-nav">
      <div class="nav-container">
        <div class="logo">
          <h1>{{ t("nav.companyName") }}</h1>
          <span class="subtitle">{{ t("nav.subtitle") }}</span>
        </div>
        <nav class="nav-tabs">
          <router-link to="/" :class="{ active: $route.path === '/' }">
            {{ t("nav.overview") }}
          </router-link>
          <router-link
            to="/inventory"
            :class="{ active: $route.path === '/inventory' }"
          >
            {{ t("nav.inventory") }}
          </router-link>
          <router-link
            to="/orders"
            :class="{ active: $route.path === '/orders' }"
          >
            {{ t("nav.orders") }}
          </router-link>
          <router-link
            to="/spending"
            :class="{ active: $route.path === '/spending' }"
          >
            {{ t("nav.finance") }}
          </router-link>
          <router-link
            to="/demand"
            :class="{ active: $route.path === '/demand' }"
          >
            {{ t("nav.demandForecast") }}
          </router-link>
          <router-link
            to="/restocking"
            :class="{ active: $route.path === '/restocking' }"
          >
            Restocking
          </router-link>
          <router-link
            to="/reports"
            :class="{ active: $route.path === '/reports' }"
          >
            {{ t("nav.reports") }}
          </router-link>
        </nav>
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </header>
    <FilterBar />
    <main class="main-content">
      <router-view />
    </main>

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

<script>
import { ref, onMounted, computed } from "vue";
import { api } from "./api";
import { useAuth } from "./composables/useAuth";
import { useI18n } from "./composables/useI18n";
import FilterBar from "./components/FilterBar.vue";
import ProfileMenu from "./components/ProfileMenu.vue";
import ProfileDetailsModal from "./components/ProfileDetailsModal.vue";
import TasksModal from "./components/TasksModal.vue";
import LanguageSwitcher from "./components/LanguageSwitcher.vue";

export default {
  name: "App",
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher,
  },
  setup() {
    const { currentUser } = useAuth();
    const { t } = useI18n();
    const showProfileDetails = ref(false);
    const showTasks = ref(false);
    const apiTasks = ref([]);

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
          const index = currentUser.value.tasks.findIndex(
            (t) => t.id === taskId,
          );
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
          mockTask.status =
            mockTask.status === "pending" ? "completed" : "pending";
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

    return {
      t,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask,
    };
  },
};
</script>

<style>
/* ============================================================
   DARK NAVY + ORANGE DESIGN SYSTEM
   Navy palette: #0a1628 (body), #0f2040 (nav/cards), #1a3a5c (elevated)
   Orange accent: #f97316 (primary), #ea580c (darker), #fed7aa (tint)
   Text: #ffffff (primary), #94a3b8 / #cbd5e1 (secondary)
   ============================================================ */

:root {
  --bg-base: #0a1628;
  --bg-nav: #0d1e38;
  --bg-card: #0f2040;
  --bg-elevated: #1a3a5c;
  --bg-hover: #152e50;
  --bg-row-hover: #132843;

  --accent-primary: #f97316;
  --accent-dark: #ea580c;
  --accent-light: #fed7aa;
  --accent-faint: rgba(249, 115, 22, 0.12);

  --text-primary: #ffffff;
  --text-secondary: #94a3b8;
  --text-muted: #64748b;

  --border-subtle: rgba(255, 255, 255, 0.08);
  --border-medium: rgba(255, 255, 255, 0.12);

  --status-success-bg: rgba(16, 185, 129, 0.15);
  --status-success-text: #34d399;
  --status-warning-bg: rgba(245, 158, 11, 0.15);
  --status-warning-text: #fbbf24;
  --status-danger-bg: rgba(239, 68, 68, 0.15);
  --status-danger-text: #f87171;
  --status-info-bg: rgba(59, 130, 246, 0.15);
  --status-info-text: #60a5fa;
}

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
  background: var(--bg-base);
  color: var(--text-primary);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

/* ---- Navigation ---- */
.top-nav {
  background: var(--bg-nav);
  border-bottom: 1px solid var(--border-subtle);
  box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.3);
  position: sticky;
  top: 0;
  z-index: 100;
}

.nav-container {
  max-width: 1600px;
  margin: 0 auto;
  display: flex;
  align-items: center;
  padding: 0 2rem;
  height: 70px;
}

.nav-container > .nav-tabs {
  margin-left: auto;
  margin-right: 1rem;
}

.nav-container > .language-switcher {
  margin-right: 1rem;
}

.logo {
  display: flex;
  align-items: baseline;
  gap: 0.75rem;
}

.logo h1 {
  font-size: 1.375rem;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.025em;
}

.subtitle {
  font-size: 0.813rem;
  color: var(--text-secondary);
  font-weight: 400;
  padding-left: 0.75rem;
  border-left: 1px solid var(--border-medium);
}

.nav-tabs {
  display: flex;
  gap: 0.25rem;
}

.nav-tabs a {
  padding: 0.625rem 1.25rem;
  color: var(--text-secondary);
  text-decoration: none;
  font-weight: 500;
  font-size: 0.938rem;
  border-radius: 6px;
  transition: all 0.2s ease;
  position: relative;
}

.nav-tabs a:hover {
  color: var(--text-primary);
  background: var(--bg-elevated);
}

.nav-tabs a.active {
  color: var(--accent-primary);
  background: var(--accent-faint);
}

.nav-tabs a.active::after {
  content: "";
  position: absolute;
  bottom: -1px;
  left: 0;
  right: 0;
  height: 2px;
  background: var(--accent-primary);
}

/* ---- Main content ---- */
.main-content {
  flex: 1;
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
  padding: 1.5rem 2rem;
}

.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: var(--text-primary);
  margin-bottom: 0.375rem;
  letter-spacing: -0.025em;
}

.page-header p {
  color: var(--text-secondary);
  font-size: 0.938rem;
}

/* ---- Stat cards ---- */
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: var(--bg-card);
  padding: 1.25rem;
  border-radius: 10px;
  border: 1px solid var(--border-subtle);
  transition: all 0.2s ease;
}

.stat-card:hover {
  border-color: var(--border-medium);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
}

.stat-label {
  color: var(--text-secondary);
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.025em;
}

.stat-card.warning .stat-value {
  color: var(--accent-primary);
}

.stat-card.success .stat-value {
  color: var(--status-success-text);
}

.stat-card.danger .stat-value {
  color: var(--status-danger-text);
}

.stat-card.info .stat-value {
  color: var(--status-info-text);
}

/* ---- Generic card ---- */
.card {
  background: var(--bg-card);
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid var(--border-subtle);
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid var(--border-subtle);
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.025em;
}

/* ---- Tables ---- */
.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: var(--bg-elevated);
  border-top: 1px solid var(--border-subtle);
  border-bottom: 1px solid var(--border-subtle);
}

th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  font-weight: 600;
  color: var(--text-secondary);
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: 0.5rem 0.75rem;
  border-top: 1px solid var(--border-subtle);
  color: #cbd5e1;
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: var(--bg-row-hover);
}

/* ---- Badges ---- */
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
  background: var(--status-success-bg);
  color: var(--status-success-text);
}

.badge.warning {
  background: var(--status-warning-bg);
  color: var(--status-warning-text);
}

.badge.danger {
  background: var(--status-danger-bg);
  color: var(--status-danger-text);
}

.badge.info {
  background: var(--status-info-bg);
  color: var(--status-info-text);
}

.badge.increasing {
  background: var(--status-success-bg);
  color: var(--status-success-text);
}

.badge.decreasing {
  background: var(--status-danger-bg);
  color: var(--status-danger-text);
}

.badge.stable {
  background: var(--status-info-bg);
  color: var(--status-info-text);
}

.badge.high {
  background: var(--status-danger-bg);
  color: var(--status-danger-text);
}

.badge.medium {
  background: var(--status-warning-bg);
  color: var(--status-warning-text);
}

.badge.low {
  background: var(--status-info-bg);
  color: var(--status-info-text);
}

/* ---- Utility ---- */
.loading {
  text-align: center;
  padding: 3rem;
  color: var(--text-secondary);
  font-size: 0.938rem;
}

.error {
  background: var(--status-danger-bg);
  border: 1px solid rgba(239, 68, 68, 0.3);
  color: var(--status-danger-text);
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}
</style>
