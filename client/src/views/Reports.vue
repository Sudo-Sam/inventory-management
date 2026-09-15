<template>
  <div class="reports">
    <div class="page-header">
      <h2>{{ t("reports.title") }}</h2>
      <p>{{ t("reports.description") }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t("common.loading") }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Quarterly Performance -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t("reports.quarterly.title") }}</h3>
        </div>
        <div class="table-container">
          <table class="reports-table">
            <thead>
              <tr>
                <th>{{ t("reports.quarterly.quarter") }}</th>
                <th>{{ t("reports.quarterly.totalOrders") }}</th>
                <th>{{ t("reports.quarterly.totalRevenue") }}</th>
                <th>{{ t("reports.quarterly.avgOrderValue") }}</th>
                <th>{{ t("reports.quarterly.fulfillmentRate") }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="q in quarterlyData" :key="q.quarter">
                <td>
                  <strong>{{ q.quarter }}</strong>
                </td>
                <td>{{ q.total_orders }}</td>
                <td>{{ formatCurrency(q.total_revenue, currentCurrency) }}</td>
                <td>
                  {{ formatCurrency(q.avg_order_value, currentCurrency) }}
                </td>
                <td>
                  <span :class="getFulfillmentClass(q.fulfillment_rate)">
                    {{ q.fulfillment_rate }}%
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Monthly Trends Chart -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t("reports.monthlyTrend.title") }}</h3>
        </div>
        <div class="chart-container">
          <div class="bar-chart">
            <div
              v-for="month in monthlyData"
              :key="month.month"
              class="bar-wrapper"
            >
              <div class="bar-container">
                <div
                  class="bar"
                  :style="{ height: getBarHeight(month.revenue) + 'px' }"
                  :title="formatCurrency(month.revenue, currentCurrency)"
                ></div>
              </div>
              <div class="bar-label">{{ formatMonth(month.month) }}</div>
            </div>
          </div>
        </div>
      </div>

      <!-- Month-over-Month Comparison -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t("reports.monthlyAnalysis.title") }}</h3>
        </div>
        <div class="table-container">
          <table class="reports-table">
            <thead>
              <tr>
                <th>{{ t("reports.monthlyTrend.month") }}</th>
                <th>{{ t("reports.monthlyAnalysis.orders") }}</th>
                <th>{{ t("reports.monthlyTrend.revenue") }}</th>
                <th>{{ t("reports.monthlyAnalysis.change") }}</th>
                <th>{{ t("reports.monthlyAnalysis.growthRate") }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(month, index) in monthlyData" :key="month.month">
                <td>
                  <strong>{{ formatMonth(month.month) }}</strong>
                </td>
                <td>{{ month.order_count }}</td>
                <td>{{ formatCurrency(month.revenue, currentCurrency) }}</td>
                <td>
                  <span
                    v-if="index > 0"
                    :class="
                      getChangeClass(
                        month.revenue,
                        monthlyData[index - 1].revenue,
                      )
                    "
                  >
                    {{
                      getChangeValue(
                        month.revenue,
                        monthlyData[index - 1].revenue,
                        currentCurrency,
                      )
                    }}
                  </span>
                  <span v-else>-</span>
                </td>
                <td>
                  <span
                    v-if="index > 0"
                    :class="
                      getChangeClass(
                        month.revenue,
                        monthlyData[index - 1].revenue,
                      )
                    "
                  >
                    {{
                      getGrowthRate(
                        month.revenue,
                        monthlyData[index - 1].revenue,
                      )
                    }}
                  </span>
                  <span v-else>-</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Summary Stats -->
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-label">
            {{ t("reports.summary.totalRevenueYTD") }}
          </div>
          <div class="stat-value">
            {{ formatCurrency(totalRevenue, currentCurrency) }}
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-label">
            {{ t("reports.summary.avgMonthlyRevenue") }}
          </div>
          <div class="stat-value">
            {{ formatCurrency(avgMonthlyRevenue, currentCurrency) }}
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-label">
            {{ t("reports.summary.totalOrdersYTD") }}
          </div>
          <div class="stat-value">{{ totalOrders }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t("reports.summary.bestQuarter") }}</div>
          <div class="stat-value">{{ bestQuarter }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch, computed } from "vue";
import { api } from "../api";
import { useI18n } from "../composables/useI18n";
import { useFilters } from "../composables/useFilters";

export default {
  name: "Reports",
  setup() {
    const { t, currentCurrency } = useI18n();
    const {
      selectedPeriod,
      selectedLocation,
      selectedCategory,
      selectedStatus,
    } = useFilters();

    const loading = ref(true);
    const error = ref(null);
    const quarterlyData = ref([]);
    const monthlyData = ref([]);
    const totalRevenue = ref(0);
    const avgMonthlyRevenue = ref(0);
    const totalOrders = ref(0);
    const bestQuarter = ref("");

    // Format currency based on current currency
    const formatCurrency = (value, currency) => {
      const formatted = value.toLocaleString("en-US", {
        minimumFractionDigits: 0,
        maximumFractionDigits: 0,
      });
      return currency === "JPY" ? "¥" + formatted : "$" + formatted;
    };

    // Format month string (YYYY-MM) to readable format
    const formatMonth = (monthStr) => {
      const parts = monthStr.split("-");
      const monthIndex = parseInt(parts[1]) - 1;
      const monthNames = [
        "Jan",
        "Feb",
        "Mar",
        "Apr",
        "May",
        "Jun",
        "Jul",
        "Aug",
        "Sep",
        "Oct",
        "Nov",
        "Dec",
      ];
      return monthNames[monthIndex] + " " + parts[0];
    };

    // Load orders and derive report data
    const loadData = async () => {
      try {
        loading.value = true;
        error.value = null;

        // Fetch all orders with current filters
        const filters = {
          warehouse: selectedLocation.value,
          category: selectedCategory.value,
          status: selectedStatus.value,
          month: selectedPeriod.value,
        };

        const orders = await api.getOrders(filters);

        // Generate quarterly data from orders
        const quarterlyMap = {};
        const monthlyMap = {};

        orders.forEach((order) => {
          const orderDate = new Date(order.order_date);
          if (isNaN(orderDate.getTime())) return;

          const month = orderDate.getMonth() + 1;
          const year = orderDate.getFullYear();
          const quarter = Math.ceil(month / 3);
          const quarterKey = `Q${quarter} ${year}`;
          const monthKey = `${year}-${String(month).padStart(2, "0")}`;

          // Initialize quarter if needed
          if (!quarterlyMap[quarterKey]) {
            quarterlyMap[quarterKey] = {
              quarter: quarterKey,
              total_orders: 0,
              total_revenue: 0,
              fulfillment_rate: 95,
            };
          }

          // Initialize month if needed
          if (!monthlyMap[monthKey]) {
            monthlyMap[monthKey] = {
              month: monthKey,
              order_count: 0,
              revenue: 0,
            };
          }

          // Aggregate data
          quarterlyMap[quarterKey].total_orders++;
          quarterlyMap[quarterKey].total_revenue += order.total_value;
          monthlyMap[monthKey].order_count++;
          monthlyMap[monthKey].revenue += order.total_value;
        });

        // Convert maps to sorted arrays
        quarterlyData.value = Object.values(quarterlyMap).sort((a, b) => {
          const aQuarter = parseInt(a.quarter.match(/\d+/)[0]);
          const aYear = parseInt(a.quarter.match(/\d{4}/)[0]);
          const bQuarter = parseInt(b.quarter.match(/\d+/)[0]);
          const bYear = parseInt(b.quarter.match(/\d{4}/)[0]);
          return aYear - bYear || aQuarter - bQuarter;
        });

        monthlyData.value = Object.values(monthlyMap).sort((a, b) =>
          a.month.localeCompare(b.month),
        );

        // Calculate average order value for quarters
        quarterlyData.value.forEach((q) => {
          if (q.total_orders > 0) {
            q.avg_order_value = Math.round(q.total_revenue / q.total_orders);
          } else {
            q.avg_order_value = 0;
          }
        });

        // Calculate summary stats
        calculateSummaryStats();
      } catch (err) {
        error.value = t("common.error") + ": " + err.message;
      } finally {
        loading.value = false;
      }
    };

    const calculateSummaryStats = () => {
      totalRevenue.value = monthlyData.value.reduce(
        (sum, m) => sum + m.revenue,
        0,
      );
      totalOrders.value = monthlyData.value.reduce(
        (sum, m) => sum + m.order_count,
        0,
      );

      if (monthlyData.value.length > 0) {
        avgMonthlyRevenue.value = totalRevenue.value / monthlyData.value.length;
      } else {
        avgMonthlyRevenue.value = 0;
      }

      if (quarterlyData.value.length > 0) {
        const best = quarterlyData.value.reduce((max, q) =>
          q.total_revenue > max.total_revenue ? q : max,
        );
        bestQuarter.value = best.quarter;
      } else {
        bestQuarter.value = "-";
      }
    };

    const getBarHeight = (revenue) => {
      if (monthlyData.value.length === 0) return 0;
      const maxRevenue = Math.max(...monthlyData.value.map((m) => m.revenue));
      if (maxRevenue === 0) return 0;
      return (revenue / maxRevenue) * 200;
    };

    const getFulfillmentClass = (rate) => {
      if (rate >= 90) return "badge success";
      if (rate >= 75) return "badge warning";
      return "badge danger";
    };

    const getChangeValue = (current, previous, currency) => {
      const change = current - previous;
      if (change > 0) return "+" + formatCurrency(change, currency);
      if (change < 0) return "-" + formatCurrency(Math.abs(change), currency);
      return formatCurrency(0, currency);
    };

    const getChangeClass = (current, previous) => {
      const change = current - previous;
      if (change > 0) return "positive-change";
      if (change < 0) return "negative-change";
      return "";
    };

    const getGrowthRate = (current, previous) => {
      if (previous === 0) return "N/A";
      const rate = ((current - previous) / previous) * 100;
      const sign = rate > 0 ? "+" : "";
      return sign + rate.toFixed(1) + "%";
    };

    // Watch filters and reload data when they change
    watch(
      [selectedPeriod, selectedLocation, selectedCategory, selectedStatus],
      () => {
        loadData();
      },
    );

    onMounted(() => {
      loadData();
    });

    return {
      t,
      loading,
      error,
      quarterlyData,
      monthlyData,
      totalRevenue,
      avgMonthlyRevenue,
      totalOrders,
      bestQuarter,
      currentCurrency,
      formatCurrency,
      formatMonth,
      getBarHeight,
      getFulfillmentClass,
      getChangeValue,
      getChangeClass,
      getGrowthRate,
    };
  },
};
</script>

<style scoped>
.reports {
  padding: 0;
}

.card {
  background: white;
  border-radius: 12px;
  padding: 1.5rem;
  margin-bottom: 1.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.card-header {
  margin-bottom: 1.5rem;
}

.card-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: #0f172a;
  margin: 0;
}

.table-container {
  overflow-x: auto;
  -webkit-overflow-scrolling: touch;
}

.reports-table {
  width: 100%;
  border-collapse: collapse;
  min-width: 500px;
}

.reports-table th {
  background: #f8fafc;
  padding: 0.75rem;
  text-align: left;
  font-weight: 600;
  color: #64748b;
  border-bottom: 2px solid #e2e8f0;
  white-space: nowrap;
}

.reports-table td {
  padding: 0.75rem;
  border-bottom: 1px solid #e2e8f0;
}

.reports-table tr:hover {
  background: #f8fafc;
}

.chart-container {
  padding: 2rem 1rem;
  min-height: 300px;
  overflow-x: auto;
  -webkit-overflow-scrolling: touch;
}

.bar-chart {
  display: flex;
  align-items: flex-end;
  justify-content: flex-start;
  height: 250px;
  gap: 0.75rem;
  min-width: min-content;
  padding-right: 1rem;
}

.bar-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  flex: 0 0 auto;
  width: 60px;
  min-width: 60px;
}

.bar-container {
  height: 200px;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  width: 100%;
}

.bar {
  width: 80%;
  min-width: 20px;
  background: linear-gradient(to top, #3b82f6, #60a5fa);
  border-radius: 4px 4px 0 0;
  transition: all 0.3s;
  cursor: pointer;
}

.bar:hover {
  background: linear-gradient(to top, #2563eb, #3b82f6);
}

.bar-label {
  margin-top: 1rem;
  font-size: 0.7rem;
  color: #64748b;
  text-align: center;
  white-space: normal;
  word-break: break-word;
  max-width: 60px;
  line-height: 1.2;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1rem;
  margin-top: 1.5rem;
}

.stat-card {
  background: white;
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border-left: 4px solid #3b82f6;
}

.stat-label {
  font-size: 0.875rem;
  color: #64748b;
  margin-bottom: 0.5rem;
  line-height: 1.3;
}

.stat-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #0f172a;
  word-break: break-word;
}

.badge {
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.875rem;
  font-weight: 500;
  display: inline-block;
}

.badge.success {
  background: #dcfce7;
  color: #166534;
}

.badge.warning {
  background: #fef3c7;
  color: #92400e;
}

.badge.danger {
  background: #fee2e2;
  color: #991b1b;
}

.positive-change {
  color: #16a34a;
  font-weight: 600;
}

.negative-change {
  color: #dc2626;
  font-weight: 600;
}

.loading {
  text-align: center;
  padding: 3rem;
  color: #64748b;
}

.error {
  background: #fee2e2;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
}

/* Mobile responsive adjustments */
@media (max-width: 768px) {
  .card {
    padding: 1rem;
    margin-bottom: 1rem;
  }

  .card-title {
    font-size: 1.1rem;
  }

  .stats-grid {
    grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
    gap: 0.75rem;
  }

  .stat-card {
    padding: 1rem;
  }

  .stat-label {
    font-size: 0.75rem;
  }

  .stat-value {
    font-size: 1.5rem;
  }

  .chart-container {
    padding: 1.5rem 0.5rem;
    min-height: 250px;
  }

  .bar-wrapper {
    width: 50px;
    min-width: 50px;
  }

  .bar-label {
    font-size: 0.65rem;
    margin-top: 0.75rem;
  }

  .reports-table th {
    padding: 0.5rem;
    font-size: 0.85rem;
  }

  .reports-table td {
    padding: 0.5rem;
    font-size: 0.85rem;
  }
}

@media (max-width: 480px) {
  .card {
    padding: 0.75rem;
  }

  .stats-grid {
    grid-template-columns: 1fr;
  }

  .bar-wrapper {
    width: 45px;
    min-width: 45px;
  }

  .bar-label {
    font-size: 0.6rem;
    max-width: 45px;
  }

  .chart-container {
    padding: 1rem 0;
  }
}
</style>
