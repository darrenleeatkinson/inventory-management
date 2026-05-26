<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Budget-based restocking recommendations from demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Allocation Card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget Allocation</h3>
        </div>
        <div class="budget-body">
          <div class="budget-row">
            <div class="budget-value-block">
              <div class="budget-label">Available Budget</div>
              <div class="budget-amount">${{ budget.toLocaleString() }}</div>
            </div>
            <div class="slider-block">
              <input
                type="range"
                class="budget-slider"
                :min="0"
                :max="totalRecommendedCost"
                :step="500"
                v-model.number="budget"
              />
            </div>
            <div class="max-block">
              <button class="btn-max" @click="budget = totalRecommendedCost">Max</button>
            </div>
          </div>

          <div class="budget-pills">
            <span class="stat-pill">{{ selectedItems.length }} items selected</span>
            <span class="stat-pill">${{ selectedCost.toLocaleString() }} total cost</span>
            <span class="stat-pill">{{ budgetUsedPct.toFixed(0) }}% of budget used</span>
          </div>

          <div class="progress-bar-container">
            <div class="progress-bar-fill" :style="{ width: budgetUsedPct + '%' }"></div>
          </div>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations</h3>
          <span class="count-chip">{{ recommendations.length }}</span>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th class="col-check">
                  <input
                    type="checkbox"
                    :checked="allSelected"
                    @change="toggleAll"
                  />
                </th>
                <th>SKU</th>
                <th>Item</th>
                <th>Category</th>
                <th>Current Demand</th>
                <th>Forecasted</th>
                <th>Change</th>
                <th>Trend</th>
                <th>Suggested Qty</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
                <th>Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="{ 'row-dimmed': !selectedSkus.has(item.sku) && !autoSelectedSkus.has(item.sku) }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(item.sku)"
                    @change="toggleItem(item.sku)"
                  />
                </td>
                <td class="sku-cell">{{ item.sku }}</td>
                <td><strong>{{ item.name }}</strong></td>
                <td>{{ item.category }}</td>
                <td>{{ item.current_demand.toLocaleString() }}</td>
                <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                <td :class="item.demand_increase_pct >= 0 ? 'change-positive' : 'change-negative'">
                  {{ item.demand_increase_pct >= 0 ? '+' : '' }}{{ item.demand_increase_pct.toFixed(1) }}%
                </td>
                <td><span :class="['badge', item.trend]">{{ item.trend }}</span></td>
                <td>{{ item.suggested_qty.toLocaleString() }}</td>
                <td>${{ item.unit_cost.toFixed(2) }}</td>
                <td><strong>${{ item.line_total.toLocaleString() }}</strong></td>
                <td><span class="lead-badge">{{ item.lead_time_days }} days</span></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order Section -->
      <div class="card order-footer">
        <div v-if="submittedOrder" class="success-banner">
          Order {{ submittedOrder.order_number }} placed successfully
        </div>
        <div v-else class="order-row">
          <div class="order-summary">
            {{ selectedItems.length }} items &middot; ${{ selectedCost.toLocaleString() }} total &middot; Est. delivery in {{ maxLeadTime }} days
          </div>
          <button
            class="btn-place-order"
            :disabled="selectedItems.length === 0 || submitting"
            :class="{ 'btn-disabled': selectedItems.length === 0 || submitting }"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const recommendations = ref([])
    const budget = ref(0)
    const selectedSkus = ref(new Set())
    const submitting = ref(false)
    const submittedOrder = ref(null)

    const totalRecommendedCost = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.line_total, 0)
    )

    const autoSelectedSkus = computed(() => {
      const selected = new Set()
      let remaining = budget.value
      for (const item of recommendations.value) {
        if (item.line_total <= remaining) {
          selected.add(item.sku)
          remaining -= item.line_total
        }
      }
      return selected
    })

    watch(budget, () => {
      selectedSkus.value = new Set(autoSelectedSkus.value)
    })

    const selectedItems = computed(() =>
      recommendations.value.filter(r => selectedSkus.value.has(r.sku))
    )

    const selectedCost = computed(() =>
      selectedItems.value.reduce((sum, r) => sum + r.line_total, 0)
    )

    const budgetUsedPct = computed(() =>
      budget.value > 0 ? Math.min(100, (selectedCost.value / budget.value) * 100) : 0
    )

    const maxLeadTime = computed(() =>
      selectedItems.value.reduce((max, r) => Math.max(max, r.lead_time_days), 0)
    )

    const allSelected = computed(() =>
      recommendations.value.length > 0 &&
      recommendations.value.every(r => selectedSkus.value.has(r.sku))
    )

    const toggleItem = (sku) => {
      const next = new Set(selectedSkus.value)
      if (next.has(sku)) {
        next.delete(sku)
      } else {
        next.add(sku)
      }
      selectedSkus.value = next
    }

    const toggleAll = (event) => {
      if (event.target.checked) {
        selectedSkus.value = new Set(recommendations.value.map(r => r.sku))
      } else {
        selectedSkus.value = new Set()
      }
    }

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingRecommendations()
        recommendations.value = data
        budget.value = totalRecommendedCost.value
        selectedSkus.value = new Set(autoSelectedSkus.value)
      } catch (err) {
        error.value = 'Failed to load restocking recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      try {
        const order = await api.submitRestockingOrder(selectedItems.value, budget.value)
        submittedOrder.value = order
        setTimeout(() => {
          submittedOrder.value = null
          loadRecommendations()
        }, 4000)
      } catch (err) {
        console.error('Failed to place order:', err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      loading,
      error,
      recommendations,
      budget,
      selectedSkus,
      submitting,
      submittedOrder,
      totalRecommendedCost,
      autoSelectedSkus,
      selectedItems,
      selectedCost,
      budgetUsedPct,
      maxLeadTime,
      allSelected,
      toggleItem,
      toggleAll,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

/* Budget Card */
.budget-body {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-row {
  display: grid;
  grid-template-columns: 180px 1fr auto;
  align-items: center;
  gap: 1.5rem;
}

.budget-label {
  font-size: 0.8rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.25rem;
}

.budget-amount {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.btn-max {
  padding: 0.375rem 0.875rem;
  background: #f1f5f9;
  color: #475569;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s ease;
  white-space: nowrap;
}

.btn-max:hover {
  background: #e2e8f0;
  color: #0f172a;
}

.budget-pills {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
}

.stat-pill {
  background: #f1f5f9;
  color: #475569;
  border-radius: 20px;
  padding: 4px 12px;
  font-size: 0.8rem;
  font-weight: 600;
}

.progress-bar-container {
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  overflow: hidden;
}

.progress-bar-fill {
  height: 100%;
  border-radius: 3px;
  background: #2563eb;
  transition: width 0.3s ease;
}

/* Table */
.count-chip {
  background: #f1f5f9;
  color: #475569;
  border-radius: 12px;
  padding: 2px 10px;
  font-size: 0.8rem;
  font-weight: 700;
}

.col-check {
  width: 40px;
  text-align: center;
}

.sku-cell {
  font-family: 'SFMono-Regular', 'Consolas', 'Liberation Mono', monospace;
  font-size: 0.813rem;
  color: #64748b;
}

.change-positive {
  color: #059669;
  font-weight: 600;
}

.change-negative {
  color: #dc2626;
  font-weight: 600;
}

.lead-badge {
  background: #f1f5f9;
  color: #475569;
  border-radius: 4px;
  padding: 2px 8px;
  font-size: 0.8rem;
  font-weight: 600;
}

.row-dimmed {
  opacity: 0.4;
}

/* Order Footer */
.order-footer {
  position: sticky;
  bottom: 1.5rem;
}

.order-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.order-summary {
  color: #475569;
  font-size: 0.938rem;
  font-weight: 500;
}

.btn-place-order {
  background: #2563eb;
  color: white;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-weight: 600;
  border: none;
  cursor: pointer;
  font-size: 0.938rem;
  transition: background 0.15s ease;
  white-space: nowrap;
}

.btn-place-order:hover:not(.btn-disabled) {
  background: #1d4ed8;
}

.btn-place-order.btn-disabled {
  background: #cbd5e1;
  color: #94a3b8;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  font-weight: 600;
  text-align: center;
}
</style>
