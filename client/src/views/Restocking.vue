<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Allocate your budget and get recommended items to restock based on demand forecasts.</p>
    </div>

    <!-- Budget slider card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Budget Allocation</h3>
        <span class="budget-display">${{ budget.toLocaleString() }}</span>
      </div>
      <div class="slider-section">
        <div class="slider-labels">
          <span>$0</span>
          <span>$500,000</span>
        </div>
        <input
          type="range"
          min="0"
          max="500000"
          step="1000"
          v-model.number="budget"
          class="budget-slider"
        />
        <!-- Budget utilization bar -->
        <div class="budget-bar-container">
          <div
            class="budget-bar-fill"
            :style="{ width: budget > 0 ? Math.min((totalCost / budget) * 100, 100) + '%' : '0%' }"
            :class="{ 'over-budget': totalCost > budget }"
          ></div>
        </div>
        <div class="budget-bar-labels">
          <span>Allocated: ${{ totalCost.toLocaleString() }}</span>
          <span>Budget: ${{ budget.toLocaleString() }}</span>
        </div>
      </div>
    </div>

    <!-- Recommended items table -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommended Items</h3>
        <span class="items-count-badge">{{ recommendedItems.length }} items</span>
      </div>

      <div v-if="loading" class="loading">Loading demand forecasts...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else>
        <div v-if="budget === 0 || recommendedItems.length === 0" class="empty-state">
          <p v-if="budget === 0">Set a budget above to see recommended restocking items.</p>
          <p v-else>No items fit within the current budget. Try increasing your budget.</p>
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th>Forecasted Demand</th>
                <th>Unit Price</th>
                <th>Subtotal</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.item_sku">
                <td><strong>{{ item.item_name }}</strong></td>
                <td><code>{{ item.item_sku }}</code></td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.forecasted_demand.toLocaleString() }} units</td>
                <td>${{ item.unit_price.toFixed(2) }}</td>
                <td><strong>${{ (item.forecasted_demand * item.unit_price).toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Order summary -->
        <div class="order-summary">
          <div class="summary-row">
            <span class="summary-label">{{ recommendedItems.length }} items recommended</span>
            <span class="summary-value">
              Total: <strong>${{ totalCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong>
              of ${{ budget.toLocaleString() }} budget
            </span>
          </div>
          <div class="summary-row">
            <span class="summary-label remaining">
              Remaining budget: <strong>${{ remainingBudget.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong>
            </span>
          </div>
        </div>

        <!-- Success message -->
        <div v-if="successMessage" class="success-message">{{ successMessage }}</div>

        <!-- Place Order button -->
        <div class="action-row">
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0"
            :class="{ disabled: recommendedItems.length === 0 }"
            @click="placeOrder"
          >
            Place Order
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { api } from '../api'
import { restockStore } from '../restockStore'

export default {
  name: 'Restocking',
  setup() {
    const router = useRouter()
    const budget = ref(0)
    const forecasts = ref([])
    const loading = ref(false)
    const error = ref(null)
    const successMessage = ref('')

    // Greedy recommendation algorithm: sort by forecasted_demand desc,
    // include items while running_cost + item_cost <= budget
    const recommendedItems = computed(() => {
      if (budget.value === 0 || forecasts.value.length === 0) return []

      const sorted = [...forecasts.value].sort(
        (a, b) => b.forecasted_demand - a.forecasted_demand
      )

      let runningCost = 0
      const result = []

      for (const item of sorted) {
        const itemCost = item.forecasted_demand * item.unit_price
        if (runningCost + itemCost <= budget.value) {
          result.push(item)
          runningCost += itemCost
        }
      }

      return result
    })

    const totalCost = computed(() =>
      recommendedItems.value.reduce(
        (sum, item) => sum + item.forecasted_demand * item.unit_price,
        0
      )
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        const response = await api.getDemandForecasts()
        // getDemandForecasts may return response.data or the array directly
        forecasts.value = Array.isArray(response) ? response : response.data
      } catch (err) {
        error.value = 'Failed to load demand forecasts'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = () => {
      if (recommendedItems.value.length === 0) return

      const order = {
        id: `RST-${Date.now()}`,
        order_number: `RST-${new Date().getFullYear()}-${String(restockStore.orders.length + 1).padStart(4, '0')}`,
        customer: 'Internal Restock',
        items: recommendedItems.value.map(item => ({
          sku: item.item_sku,
          name: item.item_name,
          quantity: item.forecasted_demand,
          unit_price: item.unit_price
        })),
        status: 'Submitted',
        order_date: new Date().toISOString(),
        expected_delivery: new Date(Date.now() + 14 * 86400000).toISOString(),
        total_value: totalCost.value,
        warehouse: 'All',
        category: 'Restock'
      }

      restockStore.orders.push(order)
      successMessage.value = 'Order placed successfully! Redirecting to Orders...'

      setTimeout(() => {
        router.push('/orders')
      }, 1500)
    }

    onMounted(loadForecasts)

    return {
      budget,
      loading,
      error,
      recommendedItems,
      totalCost,
      remainingBudget,
      successMessage,
      restockStore,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
}

.slider-section {
  padding: 0.5rem 0;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
  margin-bottom: 0.5rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  border-radius: 3px;
  appearance: none;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  accent-color: #2563eb;
}

.budget-slider::-webkit-slider-thumb {
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-bar-container {
  margin-top: 1rem;
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
}

.budget-bar-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 4px;
  transition: width 0.3s ease;
}

.budget-bar-fill.over-budget {
  background: #dc2626;
}

.budget-bar-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
  margin-top: 0.375rem;
}

.items-count-badge {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  background: #f1f5f9;
  padding: 0.25rem 0.75rem;
  border-radius: 20px;
}

.empty-state {
  text-align: center;
  padding: 3rem 2rem;
  color: #64748b;
  font-size: 0.938rem;
}

code {
  font-family: 'Courier New', monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}

.order-summary {
  margin-top: 1.25rem;
  padding: 1rem;
  background: #f8fafc;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.25rem 0;
  font-size: 0.938rem;
  color: #334155;
}

.summary-row + .summary-row {
  margin-top: 0.375rem;
}

.summary-label {
  color: #64748b;
}

.summary-label.remaining {
  color: #059669;
}

.summary-value {
  color: #0f172a;
}

.success-message {
  margin-top: 1rem;
  padding: 0.75rem 1rem;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  color: #065f46;
  font-size: 0.938rem;
  font-weight: 500;
}

.action-row {
  display: flex;
  justify-content: flex-end;
  margin-top: 1.25rem;
}

.place-order-btn {
  padding: 0.625rem 1.75rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(.disabled) {
  background: #1d4ed8;
}

.place-order-btn.disabled {
  background: #cbd5e1;
  color: #94a3b8;
  cursor: not-allowed;
}
</style>
