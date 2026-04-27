<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Allocate budget across forecasted demand and submit replenishment orders.</p>
    </div>

    <!-- Success banner lives outside the loading gate so it persists during re-fetch -->
    <div v-if="lastOrder" class="success-banner">
      <div class="banner-content">
        Order <strong>{{ lastOrder.order_number }}</strong> placed &mdash;
        expected delivery {{ formatDate(lastOrder.expected_delivery) }} (7-day lead time).
        <router-link to="/orders" class="banner-link">View in Orders</router-link>
      </div>
      <button class="banner-dismiss" @click="lastOrder = null" aria-label="Dismiss">&times;</button>
    </div>

    <div v-if="placeError" class="error place-error">{{ placeError }}</div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Stat cards -->
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-label">SKUs to Restock</div>
          <div class="stat-value">{{ recommendations.length }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">Total Units Needed</div>
          <div class="stat-value">{{ totalGapUnits.toLocaleString() }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">Total Gap Cost</div>
          <div class="stat-value">{{ formatCurrency(totalGapCost) }}</div>
        </div>
      </div>

      <!-- Budget slider -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget Allocation</h3>
          <span class="budget-hint">Drag to adjust spend — recommendations update automatically</span>
        </div>
        <div class="budget-controls">
          <div class="budget-display">{{ formatCurrency(budget) }}</div>
          <input
            type="range"
            v-model.number="budget"
            :min="0"
            :max="sliderMax"
            :step="500"
            class="budget-slider"
          />
          <div class="budget-range-labels">
            <span>$0</span>
            <span>{{ formatCurrency(sliderMax) }}</span>
          </div>
        </div>
      </div>

      <!-- Recommendations table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            Recommendations
            <span class="recs-count">{{ selectedRecommendations.length }} of {{ recommendations.length }} SKUs</span>
          </h3>
        </div>

        <div v-if="selectedRecommendations.length === 0" class="empty-state">
          No SKUs fit within the current budget. Increase the budget or check demand forecasts.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Trend</th>
                <th>Forecasted Demand</th>
                <th>On Hand</th>
                <th>Gap</th>
                <th>Restock Qty</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="rec in selectedRecommendations" :key="rec.sku">
                <td><strong>{{ rec.sku }}</strong></td>
                <td>{{ rec.name }}</td>
                <td>
                  <span :class="['badge', rec.trend]">{{ capitalize(rec.trend) }}</span>
                </td>
                <td>{{ rec.forecasted_demand.toLocaleString() }}</td>
                <td>{{ rec.on_hand.toLocaleString() }}</td>
                <td><strong>{{ rec.gap.toLocaleString() }}</strong></td>
                <td><strong>{{ rec.restock_qty.toLocaleString() }}</strong></td>
                <td>{{ formatUnitCost(rec.unit_cost) }}</td>
                <td><strong>{{ formatLineCost(rec.restock_qty * rec.unit_cost) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Summary row -->
        <div class="order-summary">
          <div class="summary-stats">
            <div class="summary-stat">
              <span class="summary-label">Total Items</span>
              <span class="summary-value">{{ selectedRecommendations.length }}</span>
            </div>
            <div class="summary-stat">
              <span class="summary-label">Total Units</span>
              <span class="summary-value">{{ totalUnits.toLocaleString() }}</span>
            </div>
            <div class="summary-stat">
              <span class="summary-label">Total Cost</span>
              <span class="summary-value">{{ formatLineCost(totalCost) }}</span>
            </div>
            <div class="summary-stat">
              <span class="summary-label">Remaining Budget</span>
              <span class="summary-value remaining">{{ formatCurrency(budget - totalCost) }}</span>
            </div>
          </div>
          <div class="summary-action">
            <button
              class="btn-place-order"
              :disabled="selectedRecommendations.length === 0 || placing"
              @click="placeOrder"
            >
              {{ placing ? 'Placing Order...' : 'Place Order' }}
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

const TREND_RANK = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const inventory = ref([])
    const forecasts = ref([])
    const budget = ref(0)
    const placing = ref(false)
    const placeError = ref(null)
    const lastOrder = ref(null)

    // Join inventory + forecasts, compute gap per SKU, keep only gap > 0
    // Sorted: increasing first, then stable, then decreasing; ties broken by gap_cost desc
    const recommendations = computed(() => {
      const inventoryMap = new Map(inventory.value.map(item => [item.sku, item]))

      return forecasts.value
        .map(forecast => {
          const inv = inventoryMap.get(forecast.item_sku)
          if (!inv) return null
          const gap = Math.max(0, forecast.forecasted_demand - inv.quantity_on_hand)
          if (gap === 0) return null
          const gap_cost = gap * inv.unit_cost
          return {
            sku: inv.sku,
            name: inv.name,
            trend: forecast.trend,
            forecasted_demand: forecast.forecasted_demand,
            on_hand: inv.quantity_on_hand,
            unit_cost: inv.unit_cost,
            gap,
            gap_cost
          }
        })
        .filter(Boolean)
        .sort((a, b) => {
          const trendDiff = (TREND_RANK[a.trend] ?? 99) - (TREND_RANK[b.trend] ?? 99)
          if (trendDiff !== 0) return trendDiff
          return b.gap_cost - a.gap_cost
        })
    })

    const totalGapCost = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.gap_cost, 0)
    )

    const totalGapUnits = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.gap, 0)
    )

    // Slider max rounded up to nearest $1,000; floor at 1000 when no data
    const sliderMax = computed(() =>
      totalGapCost.value > 0 ? Math.ceil(totalGapCost.value / 1000) * 1000 : 1000
    )

    // Greedy selection: walk sorted recommendations, assign qty = min(gap, floor(remaining / unit_cost))
    // Include SKU if qty > 0; stop early once remaining can no longer buy anything
    const selectedRecommendations = computed(() => {
      let remaining = budget.value
      const selected = []

      for (const rec of recommendations.value) {
        if (remaining <= 0) break
        const qty = Math.min(rec.gap, Math.floor(remaining / rec.unit_cost))
        if (qty > 0) {
          selected.push({ ...rec, restock_qty: qty })
          remaining -= qty * rec.unit_cost
        }
      }

      return selected
    })

    const totalUnits = computed(() =>
      selectedRecommendations.value.reduce((sum, r) => sum + r.restock_qty, 0)
    )

    const totalCost = computed(() =>
      selectedRecommendations.value.reduce((sum, r) => sum + r.restock_qty * r.unit_cost, 0)
    )

    const formatCurrency = (value) =>
      value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })

    const formatUnitCost = (value) =>
      value.toLocaleString('en-US', { style: 'currency', currency: 'USD', minimumFractionDigits: 2, maximumFractionDigits: 2 })

    const formatLineCost = (value) =>
      value.toLocaleString('en-US', { style: 'currency', currency: 'USD', minimumFractionDigits: 2, maximumFractionDigits: 2 })

    const formatDate = (dateStr) => {
      const date = new Date(dateStr)
      if (isNaN(date.getTime())) return dateStr
      return date.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })
    }

    const capitalize = (str) =>
      str ? str.charAt(0).toUpperCase() + str.slice(1) : str

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [inv, fcst] = await Promise.all([
          api.getInventory(),
          api.getDemandForecasts()
        ])
        inventory.value = inv
        forecasts.value = fcst
        // Default budget: half of max, snapped to step
        budget.value = sliderMax.value / 2
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedRecommendations.value.length === 0 || placing.value) return
      placing.value = true
      placeError.value = null
      try {
        const items = selectedRecommendations.value.map(r => ({
          sku: r.sku,
          name: r.name,
          quantity: r.restock_qty,
          unit_cost: r.unit_cost
        }))
        const response = await api.createRestockOrder({ items })
        lastOrder.value = response
        // Re-fetch to reflect latest inventory state after the order
        await loadData()
      } catch (err) {
        placeError.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        placing.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      budget,
      placing,
      placeError,
      lastOrder,
      recommendations,
      selectedRecommendations,
      sliderMax,
      totalGapCost,
      totalGapUnits,
      totalUnits,
      totalCost,
      formatCurrency,
      formatUnitCost,
      formatLineCost,
      formatDate,
      capitalize,
      placeOrder
    }
  }
}
</script>

<style scoped>
.success-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  color: #065f46;
}

.banner-content {
  flex: 1;
}

.banner-link {
  color: #047857;
  font-weight: 600;
  text-decoration: underline;
  margin-left: 0.25rem;
}

.banner-link:hover {
  color: #065f46;
}

.banner-dismiss {
  flex-shrink: 0;
  background: transparent;
  border: none;
  color: #059669;
  font-size: 1.25rem;
  line-height: 1;
  cursor: pointer;
  padding: 0.25rem 0.375rem;
  border-radius: 4px;
  transition: background 0.15s;
}

.banner-dismiss:hover {
  background: #a7f3d0;
  color: #065f46;
}

.place-error {
  margin-bottom: 1.25rem;
}

/* Budget card */
.budget-card .card-header {
  border-bottom: 1px solid #e2e8f0;
  padding-bottom: 0.875rem;
  margin-bottom: 1.25rem;
}

.budget-hint {
  font-size: 0.813rem;
  color: #94a3b8;
  font-weight: 400;
}

.budget-controls {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
  padding: 0.5rem 0 0.25rem;
}

.budget-display {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.035em;
  line-height: 1;
}

.budget-slider {
  width: 100%;
  max-width: 680px;
  height: 6px;
  accent-color: #3b82f6;
  cursor: pointer;
}

.budget-range-labels {
  width: 100%;
  max-width: 680px;
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #94a3b8;
  font-weight: 500;
}

/* Table card */
.recs-count {
  display: inline-block;
  margin-left: 0.75rem;
  font-size: 0.813rem;
  font-weight: 500;
  color: #64748b;
  background: #f1f5f9;
  padding: 0.125rem 0.625rem;
  border-radius: 999px;
  vertical-align: middle;
}

.empty-state {
  padding: 3rem;
  text-align: center;
  color: #94a3b8;
  font-size: 0.938rem;
}

/* Order summary strip */
.order-summary {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1.5rem;
  padding: 1rem 1.25rem;
  border-top: 1px solid #e2e8f0;
  background: #f8fafc;
  border-radius: 0 0 10px 10px;
  flex-wrap: wrap;
}

.summary-stats {
  display: flex;
  gap: 2rem;
  flex-wrap: wrap;
}

.summary-stat {
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
}

.summary-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #94a3b8;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.summary-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.summary-value.remaining {
  color: #059669;
}

.summary-action {
  flex-shrink: 0;
}

.btn-place-order {
  padding: 0.625rem 1.75rem;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s, box-shadow 0.2s;
  white-space: nowrap;
}

.btn-place-order:hover:not(:disabled) {
  background: #2563eb;
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.35);
}

.btn-place-order:disabled {
  background: #cbd5e1;
  color: #94a3b8;
  cursor: not-allowed;
  box-shadow: none;
}
</style>
