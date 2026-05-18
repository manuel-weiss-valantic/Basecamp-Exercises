<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Recommendations</h2>
      <p>Budget-constrained restock suggestions based on demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading restocking data...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success Banner -->
      <div v-if="successOrder" class="success-banner">
        <div class="success-banner-content">
          <span class="success-icon">&#10003;</span>
          <div>
            <strong>Order submitted!</strong>
            Expected delivery:
            <strong>{{ formatDeliveryDate(successOrder.expected_delivery_date) }}</strong>
          </div>
          <button class="dismiss-btn" @click="successOrder = null">&#10005;</button>
        </div>
      </div>

      <!-- Submit Error -->
      <div v-if="submitError" class="error">{{ submitError }}</div>

      <!-- Budget Controls -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
          <span class="budget-value">{{ currencySymbol }}{{ formatNumber(budget) }}</span>
        </div>
        <div class="budget-body">
          <input
            type="range"
            min="0"
            max="100000"
            step="1000"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="slider-labels">
            <span>{{ currencySymbol }}0</span>
            <span>{{ currencySymbol }}{{ formatNumber(100000) }}</span>
          </div>
          <div class="utilization-bar-track">
            <div
              class="utilization-bar-fill"
              :style="{ width: utilizationPercent + '%' }"
              :class="utilizationClass"
            ></div>
          </div>
          <div class="utilization-labels">
            <span>Recommended spend: <strong>{{ currencySymbol }}{{ formatNumber(recommendedSpend) }}</strong></span>
            <span class="utilization-pct" :class="utilizationClass">{{ utilizationPercent }}% of budget</span>
          </div>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <span class="rec-count">{{ recommendedItems.length }} item{{ recommendedItems.length !== 1 ? 's' : '' }} within budget</span>
        </div>

        <div v-if="recommendedItems.length === 0" class="empty-state">
          <p>No items require restocking within the current budget.</p>
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Qty to Restock</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
                <th>Trend</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td>{{ item.item_name }}</td>
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.restockQty }}</td>
                <td>{{ currencySymbol }}{{ formatNumber(item.unit_cost) }}</td>
                <td><strong>{{ currencySymbol }}{{ formatNumber(item.itemTotal) }}</strong></td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="card-footer">
          <div class="total-row">
            <span>Total order cost:</span>
            <strong>{{ currencySymbol }}{{ formatNumber(recommendedSpend) }}</strong>
          </div>
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : 'Place Order' }}
          </button>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const submitError = ref(null)
    const submitting = ref(false)
    const successOrder = ref(null)

    const forecasts = ref([])
    const inventory = ref([])
    const budget = ref(50000)

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const inventoryBySku = computed(() => {
      const map = {}
      for (const item of inventory.value) {
        map[item.sku] = {
          unit_cost: item.unit_cost,
          quantity_on_hand: item.quantity_on_hand
        }
      }
      return map
    })

    const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }

    const allCandidates = computed(() => {
      const candidates = []
      for (const forecast of forecasts.value) {
        const inv = inventoryBySku.value[forecast.item_sku]
        if (!inv || inv.unit_cost == null) continue

        const restockQty = Math.max(0, forecast.forecasted_demand - inv.quantity_on_hand)
        if (restockQty === 0) continue

        const itemTotal = restockQty * inv.unit_cost
        candidates.push({
          sku: forecast.item_sku,
          item_name: forecast.item_name,
          restockQty,
          unit_cost: inv.unit_cost,
          itemTotal,
          trend: forecast.trend
        })
      }

      candidates.sort((a, b) => {
        const ao = trendOrder[a.trend] ?? 99
        const bo = trendOrder[b.trend] ?? 99
        return ao - bo
      })

      return candidates
    })

    const recommendedItems = computed(() => {
      let running = 0
      const included = []
      for (const item of allCandidates.value) {
        if (running + item.itemTotal <= budget.value) {
          running += item.itemTotal
          included.push(item)
        }
      }
      return included
    })

    const recommendedSpend = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.itemTotal, 0)
    })

    const utilizationPercent = computed(() => {
      if (budget.value === 0) return 0
      return Math.min(100, Math.round((recommendedSpend.value / budget.value) * 100))
    })

    const utilizationClass = computed(() => {
      const pct = utilizationPercent.value
      if (pct >= 90) return 'util-high'
      if (pct >= 60) return 'util-medium'
      return 'util-low'
    })

    const formatNumber = (value) => {
      if (value == null) return '0'
      return Number(value).toLocaleString('en-US')
    }

    const formatDeliveryDate = (dateStr) => {
      if (!dateStr) return 'TBD'
      const d = new Date(dateStr)
      if (isNaN(d.getTime())) return dateStr
      return d.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecastData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        forecasts.value = forecastData
        inventory.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      submitError.value = null
      successOrder.value = null
      try {
        const payload = {
          items: recommendedItems.value.map(i => ({
            sku: i.sku,
            item_name: i.item_name,
            quantity: i.restockQty,
            unit_cost: i.unit_cost,
            total_cost: i.itemTotal
          }))
        }
        const response = await api.submitRestockingOrder(payload)
        successOrder.value = response
      } catch (err) {
        submitError.value = 'Failed to submit order: ' + (err.response?.data?.detail || err.message)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      submitError,
      submitting,
      successOrder,
      budget,
      currencySymbol,
      recommendedItems,
      recommendedSpend,
      utilizationPercent,
      utilizationClass,
      formatNumber,
      formatDeliveryDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.25rem;
}

.page-header p {
  font-size: 0.875rem;
  color: #64748b;
  margin: 0;
}

/* Success banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  margin-bottom: 1.5rem;
}

.success-banner-content {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.875rem 1rem;
  color: #065f46;
  font-size: 0.9rem;
}

.success-icon {
  font-size: 1.1rem;
  font-weight: 700;
  flex-shrink: 0;
}

.success-banner-content > div {
  flex: 1;
}

.dismiss-btn {
  background: none;
  border: none;
  cursor: pointer;
  color: #065f46;
  font-size: 0.85rem;
  padding: 0.25rem;
  opacity: 0.7;
  flex-shrink: 0;
}

.dismiss-btn:hover {
  opacity: 1;
}

/* Budget card */
.budget-card .card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.budget-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-body {
  padding: 0 0 0.5rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #3b82f6;
  cursor: pointer;
  margin-bottom: 0.25rem;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-bottom: 1rem;
}

.utilization-bar-track {
  width: 100%;
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 0.5rem;
}

.utilization-bar-fill {
  height: 100%;
  border-radius: 4px;
  transition: width 0.3s ease, background-color 0.3s ease;
}

.utilization-bar-fill.util-low {
  background: #10b981;
}

.utilization-bar-fill.util-medium {
  background: #f59e0b;
}

.utilization-bar-fill.util-high {
  background: #ef4444;
}

.utilization-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.8rem;
  color: #64748b;
}

.utilization-pct {
  font-weight: 600;
}

.utilization-pct.util-low {
  color: #10b981;
}

.utilization-pct.util-medium {
  color: #f59e0b;
}

.utilization-pct.util-high {
  color: #ef4444;
}

/* Recommendations header */
.rec-count {
  font-size: 0.8rem;
  color: #64748b;
  font-weight: 500;
}

/* Empty state */
.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.9rem;
}

/* Card footer with totals and action */
.card-footer {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 1.5rem;
  padding: 1rem 1.25rem;
  border-top: 1px solid #e2e8f0;
  background: #f8fafc;
  border-radius: 0 0 8px 8px;
}

.total-row {
  font-size: 0.9rem;
  color: #64748b;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.total-row strong {
  font-size: 1rem;
  color: #0f172a;
}

.place-order-btn {
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.6rem 1.4rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #2563eb;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  color: #94a3b8;
  cursor: not-allowed;
}

/* Spacing between cards */
.card {
  margin-bottom: 1.5rem;
}
</style>
