<template>
  <div class="restocking-view">
    <!-- Page header (match style of other views) -->
    <div class="page-header">
      <h2>Restocking Recommendations</h2>
      <p class="page-description">Set your available budget to see which items to restock based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading-state">Loading...</div>
    <div v-else-if="error" class="error-state">{{ error }}</div>
    <div v-else>

      <!-- Budget Slider Card -->
      <div class="card budget-card">
        <div class="budget-header">
          <h3 class="card-title">Available Budget</h3>
          <span class="budget-value">{{ formatCurrency(budget) }}</span>
        </div>
        <div class="slider-wrapper">
          <input
            type="range"
            v-model.number="budget"
            :min="0"
            :max="maxBudget"
            :step="100"
            class="budget-slider"
          />
          <div class="slider-labels">
            <span>$0</span>
            <span>{{ formatCurrency(maxBudget) }}</span>
          </div>
        </div>
        <div class="budget-stats">
          <div class="budget-stat">
            <span class="stat-label">Items to Restock</span>
            <span class="stat-value">{{ recommendedItems.length }}</span>
          </div>
          <div class="budget-stat">
            <span class="stat-label">Estimated Cost</span>
            <span class="stat-value">{{ formatCurrency(estimatedCost) }}</span>
          </div>
          <div class="budget-stat">
            <span class="stat-label">Remaining Budget</span>
            <span class="stat-value" :class="{ 'over-budget': remainingBudget < 0 }">
              {{ formatCurrency(Math.abs(remainingBudget)) }}
              {{ remainingBudget < 0 ? ' over' : ' remaining' }}
            </span>
          </div>
        </div>
      </div>

      <!-- Empty state -->
      <div v-if="allCandidates.length === 0" class="card">
        <p class="empty-state">All items are sufficiently stocked to meet forecasted demand. No restocking needed.</p>
      </div>

      <!-- Recommendations Card -->
      <div v-else class="card">
        <div class="recommendations-header">
          <div>
            <h3 class="card-title">
              Recommended Restock Items
            </h3>
            <p class="recommendations-meta">
              {{ recommendedItems.length }} of {{ allCandidates.length }} items fit within budget
            </p>
          </div>
          <div class="header-actions">
            <div v-if="orderSubmitted" class="order-success">
              Order submitted — check the Orders tab for details.
            </div>
            <button
              v-else-if="recommendedItems.length > 0"
              @click="placeOrder"
              :disabled="submitting"
              class="btn-primary"
            >
              {{ submitting ? 'Placing Order...' : 'Place Order' }}
            </button>
          </div>
        </div>
        <div v-if="submitError" class="error-banner">{{ submitError }}</div>

        <div class="restock-grid">
          <div
            v-for="item in allCandidates"
            :key="item.sku"
            :class="['restock-card', isIncluded(item) ? 'included' : 'excluded']"
          >
            <div class="restock-card-header">
              <div class="restock-card-title">
                <div class="item-name">{{ item.item_name }}</div>
                <div class="item-sku">{{ item.sku }}</div>
              </div>
              <span :class="['badge', item.trend === 'increasing' ? 'success' : item.trend === 'decreasing' ? 'danger' : 'info']">
                {{ item.trend }}
              </span>
            </div>
            <div class="restock-card-body">
              <div class="detail-row">
                <span class="detail-label">Forecasted demand</span>
                <span class="detail-value">{{ item.forecasted_demand }} units</span>
              </div>
              <div class="detail-row">
                <span class="detail-label">On hand</span>
                <span class="detail-value">{{ item.quantity_on_hand }} units</span>
              </div>
              <div class="detail-row">
                <span class="detail-label">Order qty</span>
                <span class="detail-value order-qty">{{ item.gap }} units</span>
              </div>
              <div v-if="item.has_cost_data" class="detail-row">
                <span class="detail-label">Unit cost</span>
                <span class="detail-value">${{ item.unit_cost.toFixed(2) }}</span>
              </div>
              <div v-if="item.has_cost_data" class="detail-row line-total">
                <span class="detail-label">Line total</span>
                <span class="detail-value">{{ formatCurrency(item.gap * item.unit_cost) }}</span>
              </div>
            </div>
            <div v-if="!isIncluded(item)" class="excluded-notice">
              {{ item.has_cost_data ? 'Excluded: insufficient budget' : 'No cost data — cannot order' }}
            </div>
          </div>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const inventoryItems = ref([])
    const budget = ref(10000)
    const submitting = ref(false)
    const orderSubmitted = ref(false)
    const submitError = ref(null)

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        forecasts.value = forecastsData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load restocking data'
        console.error('Load error:', err)
      } finally {
        loading.value = false
      }
    }

    onMounted(loadData)

    // O(1) lookup map from sku -> inventory item
    const inventoryBySku = computed(() => {
      const map = {}
      for (const item of inventoryItems.value) {
        map[item.sku] = item
      }
      return map
    })

    // Join forecasts to inventory, compute gap, filter out gap=0, sort
    const allCandidates = computed(() => {
      const result = []
      for (const forecast of forecasts.value) {
        const inv = inventoryBySku.value[forecast.item_sku] || null
        const quantityOnHand = inv ? inv.quantity_on_hand : 0
        const unit_cost = inv ? inv.unit_cost : null
        const gap = Math.max(0, forecast.forecasted_demand - quantityOnHand)
        if (gap === 0) continue
        result.push({
          sku: forecast.item_sku,
          item_name: forecast.item_name,
          trend: forecast.trend,
          current_demand: forecast.current_demand,
          forecasted_demand: forecast.forecasted_demand,
          quantity_on_hand: quantityOnHand,
          gap,
          unit_cost,
          has_cost_data: inv !== null
        })
      }
      // Sort: items with cost data first, then increasing trend first, then largest gap
      const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }
      result.sort((a, b) => {
        if (a.has_cost_data !== b.has_cost_data) return a.has_cost_data ? -1 : 1
        const td = (trendOrder[a.trend] ?? 1) - (trendOrder[b.trend] ?? 1)
        if (td !== 0) return td
        return b.gap - a.gap
      })
      return result
    })

    // Sum of all orderable line costs, rounded up to next $1000, minimum $10,000
    const maxBudget = computed(() => {
      const total = allCandidates.value
        .filter(item => item.has_cost_data)
        .reduce((sum, item) => sum + item.gap * item.unit_cost, 0)
      return Math.max(10000, Math.ceil(total / 1000) * 1000)
    })

    // Greedy selection — fill budget with items that have cost data, sorted by priority
    const recommendedItems = computed(() => {
      const orderable = allCandidates.value.filter(item => item.has_cost_data)
      const selected = []
      let spent = 0
      for (const item of orderable) {
        const cost = item.gap * item.unit_cost
        if (spent + cost <= budget.value) {
          selected.push(item)
          spent += cost
        }
        // skip if doesn't fit, continue checking smaller items
      }
      return selected
    })

    const estimatedCost = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.gap * item.unit_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - estimatedCost.value)

    const isIncluded = (item) => recommendedItems.value.some(r => r.sku === item.sku)

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0) return
      submitting.value = true
      submitError.value = null
      try {
        const orderItems = recommendedItems.value.map(item => ({
          sku: item.sku,
          name: item.item_name,
          quantity: item.gap,
          unit_cost: item.unit_cost
        }))
        await api.submitRestockingOrder(orderItems)
        orderSubmitted.value = true
      } catch (err) {
        submitError.value = 'Failed to place order. Please try again.'
        console.error('Order error:', err)
      } finally {
        submitting.value = false
      }
    }

    return {
      loading,
      error,
      forecasts,
      inventoryItems,
      budget,
      maxBudget,
      submitting,
      orderSubmitted,
      submitError,
      allCandidates,
      recommendedItems,
      estimatedCost,
      remainingBudget,
      isIncluded,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking-view {
  padding: 0;
}

.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.25rem;
}

.page-description {
  color: #64748b;
  font-size: 0.9rem;
}

.loading-state,
.error-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
}

.error-state {
  color: #dc2626;
}

/* Budget Card */
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.25rem;
}

.budget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #2563eb;
}

.slider-wrapper {
  margin-bottom: 1.25rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
  margin-bottom: 0.5rem;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.8rem;
  color: #94a3b8;
}

.budget-stats {
  display: flex;
  gap: 2rem;
  padding-top: 1.25rem;
  border-top: 1px solid #e2e8f0;
}

.budget-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.stat-label {
  font-size: 0.72rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: #94a3b8;
}

.stat-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.over-budget {
  color: #dc2626;
}

/* Recommendations Card */
.recommendations-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 1.25rem;
}

.recommendations-meta {
  font-size: 0.82rem;
  color: #64748b;
  margin-top: 0.2rem;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
  flex-shrink: 0;
}

.btn-primary {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 6px;
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
  white-space: nowrap;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.order-success {
  padding: 0.5rem 1rem;
  background: #d1fae5;
  color: #065f46;
  border-radius: 6px;
  font-size: 0.85rem;
  font-weight: 500;
}

.error-banner {
  margin-bottom: 1rem;
  padding: 0.625rem 1rem;
  background: #fee2e2;
  color: #991b1b;
  border-radius: 6px;
  font-size: 0.85rem;
}

/* Item Grid */
.restock-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
  gap: 1rem;
}

.restock-card {
  border-radius: 8px;
  border: 1px solid #e2e8f0;
  padding: 1rem;
  transition: box-shadow 0.15s;
}

.restock-card.included {
  border-left: 4px solid #10b981;
  background: #f0fdf4;
}

.restock-card.excluded {
  border-left: 4px solid #cbd5e1;
  background: #f8fafc;
  opacity: 0.8;
}

.restock-card-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 0.875rem;
  gap: 0.5rem;
}

.item-name {
  font-size: 0.9rem;
  font-weight: 600;
  color: #0f172a;
  line-height: 1.3;
}

.item-sku {
  font-size: 0.72rem;
  color: #64748b;
  font-family: monospace;
  margin-top: 0.2rem;
}

.restock-card-body {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}

.detail-row {
  display: flex;
  justify-content: space-between;
  font-size: 0.8rem;
}

.detail-label {
  color: #64748b;
}

.detail-value {
  font-weight: 500;
  color: #334155;
}

.order-qty {
  color: #2563eb;
  font-weight: 700;
}

.line-total {
  padding-top: 0.4rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.2rem;
}

.line-total .detail-value {
  font-weight: 700;
  color: #0f172a;
}

.excluded-notice {
  margin-top: 0.75rem;
  padding: 0.375rem 0.625rem;
  background: #f1f5f9;
  border-radius: 4px;
  font-size: 0.73rem;
  color: #64748b;
  font-style: italic;
}

.empty-state {
  text-align: center;
  padding: 2.5rem;
  color: #64748b;
}
</style>
